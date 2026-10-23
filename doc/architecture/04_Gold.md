# 04 — Gold : indicateurs prêts pour l'analyse

Précédent : [03_Silver.md](03_Silver.md) · Vue d'ensemble : [Data_flow.md](Data_flow.md) · Suivant : [05_Serving.md](05_Serving.md)

---

## 1. Rôle

Calculer, avec Spark, **tous les chiffres que l'app affiche**. L'app ne fait que lire et filtrer : aucun agrégat lourd côté Streamlit.

- Tables orientées **questions métier** (prix, socio-éco, sentiment, profil de zone), pas orientées source.
- **Multi-échelle** : chaque indicateur existe pour commune, EPCI, département, région (exigence du sujet).
- Chaque valeur est accompagnée de sa **fiabilité** (taille d'échantillon).

## 2. Ce qui entre / ce qui sort

| Entrée | Sortie |
|---|---|
| Tables Silver, lignes `est_exploitable = true` uniquement | Tables Delta `gold.<indicateur>` recalculées par période |
| `silver.ref_geo` (hiérarchie des zones) | |

## 3. Grain commun

Toutes les tables Gold partagent la même clé :

| Colonne | Valeurs |
|---|---|
| `niveau_geo` | `commune`, `epci`, `departement`, `region`, `france` |
| `code_geo` | code de la zone au COG de référence |
| `periode` | `2025` (année) ou `2025S1` (semestre) ou `2025T2` (trimestre) |
| `type_periode` | `annee`, `semestre`, `trimestre` |

Plus, sur chaque ligne : `nb_observations`, `est_significatif`, `date_calcul`, `batch_ids_sources` (lineage).

## 4. Règle d'agrégation multi-échelle

> **Une médiane ne s'agrège pas.** La médiane d'un département n'est **pas** la moyenne des médianes de ses communes.

Donc chaque niveau est calculé **depuis les lignes Silver**, pas depuis le niveau inférieur :

```python
for niveau in ["commune", "epci", "departement", "region", "france"]:
    (transactions
        .join(ref_geo_hierarchie, "code_insee")       # code_insee → code_<niveau>
        .groupBy(f"code_{niveau}", "periode", "type_bien")
        .agg(F.percentile_approx("prix_m2", 0.5).alias("prix_m2_median"),
             F.count("*").alias("nb_observations")))
```

Seuls les comptes et les sommes (nb ventes, population) peuvent être additionnés d'un niveau à l'autre.

## 5. Tables

### 5.1 `gold.indicateurs_prix` (source : `silver.transaction`)

Grain : `(niveau_geo, code_geo, periode, type_bien)` avec `type_bien ∈ {appartement, maison, tous}`.

| Indicateur | Définition |
|---|---|
| `prix_m2_median` | médiane du prix/m² des mutations mono-type exploitables |
| `prix_m2_p25`, `prix_m2_p75` | quartiles → dispersion du marché |
| `prix_median` | médiane de `valeur_fonciere` |
| `surface_mediane` | médiane de surface |
| `nb_ventes` | nb de mutations |
| `evol_prix_m2_1an` | `(médiane N / médiane N-1) - 1` |
| `evol_prix_m2_5ans` | idem sur 5 ans |
| `evol_nb_ventes_1an` | dynamisme du marché |

Seuil : `nb_ventes < 10` → `est_significatif = false`, indicateurs de prix à `null` (pas affichés sur la carte, grisés).

### 5.2 `gold.indicateurs_socio` (sources : revenu, population, chômage, prix)

| Indicateur | Définition | Niveau min. |
|---|---|---|
| `revenu_median_uc` | Filosofi (commune) ; au-dessus : **pondéré** par `nb_menages_fiscaux` (approximation documentée, la vraie médiane n'est pas recalculable sans les données individuelles) | commune |
| `ecart_interdecile` | `D9 / D1` → inégalités | commune |
| `taux_pauvrete` | Filosofi | commune |
| `population`, `densite` | INSEE / surface de la géométrie | commune |
| `taux_chomage` | INSEE | **département** (`null` en dessous, pas de fausse précision) |
| `annees_revenu_pour_60m2` | `prix_m2_median × 60 / revenu_median_uc` → accessibilité du logement | commune |

Seuil : `nb_menages_fiscaux < 50` → `est_significatif = false`.

### 5.3 `gold.indicateurs_sentiment` (source : `silver.avis`)

| Indicateur | Définition |
|---|---|
| `nb_avis` | taille d'échantillon |
| `score_sentiment_moyen` | moyenne de `sentiment_score` |
| `part_negatif` | % avis `negatif` |
| `themes_principaux` | top 5 thèmes par fréquence |
| `modele_nlp` | version du modèle ayant produit les scores |

Seuil : `nb_avis < 20` → `est_significatif = false`.

### 5.4 `gold.indicateurs_logement` (sources : DPE, risques, équipements)

| Indicateur | Définition |
|---|---|
| `part_dpe_A_B`, `part_dpe_F_G` | part de "passoires thermiques" (F/G : interdiction progressive à la location) |
| `classe_dpe_mediane` | |
| `niveau_risque_argiles` | niveau dominant (pondéré par surface ou population) |
| `nb_ecoles`, `nb_ecoles_pour_1000_hab` | |
| `distance_gare_mediane_km` | distance médiane transaction → gare la plus proche (calcul Lambert 93) |

### 5.5 `gold.profil_zone`

Table **large** : une ligne par `(niveau_geo, code_geo, periode)`, toutes les colonnes des tables 5.1 à 5.4 (pour `type_bien = tous`) + nom de la zone + `code_parent`.

- Alimente directement les vues tableaux de l'app (tri, filtre, export CSV) et le croisement d'indicateurs (scatter prix × revenu…).
- Ajoute les **rangs** (`rang_prix_m2_dans_departement`, `percentile_national`) pour les classements.

### 5.6 `gold.transaction_carte` (option)

Échantillon des transactions géocodées (lat/lon, prix/m², type, date) pour les cartes de points/chaleur. Limité aux colonnes utiles et aux 5 dernières années pour rester léger.

## 6. Recalcul

- Gold est **recalculé**, jamais patché : `overwrite` des partitions `periode` touchées par les nouveaux lots Silver.
- Un nouveau lot DVF (millésime 2025) → recalcul des périodes 2025 + des `evol_*` de 2026 (qui dépendent de 2025).
- Un changement de COG de référence → recalcul complet.

## 7. Contrôles de sortie

| Contrôle | Bloquant |
|---|---|
| Unicité de la clé `(niveau_geo, code_geo, periode[, type_bien])` | oui |
| Tous les `code_geo` existent dans `silver.ref_geo` | oui |
| Cohérence d'échelle : `somme(nb_ventes communes) = nb_ventes département` | oui |
| Couverture : % de communes avec un prix significatif ≥ seuil attendu | non (alerte) |
| Valeurs plausibles (prix/m² France entre 1 000 et 10 000 €) | non (alerte) |

## 8. Definition of done

- [ ] Chaque indicateur affiché par l'app a sa définition ici.
- [ ] Tous les niveaux géo calculés depuis Silver (pas d'agrégat d'agrégat sur des médianes).
- [ ] `nb_observations` + `est_significatif` sur chaque ligne.
- [ ] `gold.profil_zone` disponible pour les 4 échelles.
