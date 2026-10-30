# 03 — Silver : nettoyé, typé, conforme

Précédent : [02_Bronze.md](02_Bronze.md) · Vue d'ensemble : [Data_flow.md](Data_flow.md) · Suivant : [04_Gold.md](04_Gold.md)

---

## 1. Rôle

Produire des **entités métier fiables et joignables entre elles**. C'est ici que vit la **méthodologie de nettoyage** (livrable du sujet) : chaque règle ci-dessous doit y être reportée.

- Une table Silver = une entité métier (une commune, une vente, un DPE, un avis…), pas une source.
- Toutes les tables Silver parlent le même langage : mêmes codes géo, mêmes types, mêmes unités, même millésime COG.
- On **marque** les lignes douteuses, on ne les supprime pas.

## 2. Ce qui entre / ce qui sort

| Entrée | Sortie |
|---|---|
| Nouveaux lots Bronze (`_batch_id` non encore traités) | Tables Delta `silver.<entite>`, mises à jour par `MERGE` |
| Référentiel géo Silver (pour les jointures) | |

## 3. Ordre de traitement

Le référentiel passe **toujours en premier** : toutes les autres tables valident leurs codes géo contre lui.

```
1. silver.ref_geo  (Admin Express + COG + table de passage)
2. silver.adresse  (BAN)                      ─┐
3. silver.transaction (DVF) ── géocodage ──────┘
4. silver.revenu_commune, silver.population, silver.chomage ...
5. silver.dpe_logement, silver.risque_commune, silver.equipement (écoles, gares)
6. silver.avis (extraction HTML + NLP), silver.annonce (Orpi)
```

## 4. Règles communes (toutes les tables)

### 4.1 Typage

| Donnée | Règle |
|---|---|
| Numériques FR | supprimer espaces / espaces insécables, `,` → `.`, cast `decimal`. Échec de cast → `null` + `flag` (jamais 0). |
| Dates | cast ISO `date` depuis `dd/MM/yyyy` (DVF) ou ISO (API). Dates futures ou < 1900 → `null` + flag. |
| Montants | en euros, `decimal(14,2)`. |
| Surfaces | en m², `decimal(10,2)`. |
| Booléens | `oui/non`, `O/N`, `1/0`, `true/false` (insensible à la casse) → `boolean`. Autre valeur → `null` + flag. |
| Texte | `trim`, espaces multiples réduits, encodage UTF-8. Libellés normalisés en minuscules sans accents dans une colonne `_norm` à côté de l'original. |
| Valeurs secrétisées / non disponibles (`s`, `nd`, `NA`, vide) | `null` + colonne `<champ>_statut` (`secret`, `absent`). |

### 4.2 Codes géographiques

- `code_insee` : string **5 caractères**. Bronze ne perd aucun zéro (tout en string), mais le zéro peut manquer **dans la source** :
  - DVF brut : pas de colonne `code_insee`, on la construit : `code_departement + lpad(code_commune, 3, '0')` (`Code commune` arrive sans zéros de tête, à vérifier sur échantillon).
  - xlsx INSEE / Filosofi : cellule numérique dans Excel → zéro perdu dans le fichier lui-même (`"1004"`).
- Règle de complétion, **seulement pour le cas non ambigu** (un seul zéro de tête perdu, valeur purement numérique) :

  | Code | Longueur attendue | Exemple propre | Après Excel | Complétion |
  |---|---|---|---|---|
  | `code_insee` | 5 | `01053` | `1053` | 4 chiffres → `lpad` à 5 (départements 01–09) |
  | `code_departement` | 2 (3 pour les DROM) | `01` | `1` | 1 chiffre → `lpad` à 2 |
  | `code_region` | 2 | `01` (Guadeloupe) … `06` (Mayotte) | `1` … `6` | 1 chiffre → `lpad` à 2 |

  Valeur brute gardée dans `code_insee_source` / `code_departement_source` / `code_region_source`, flag non bloquant `code_geo_complete`.
- Tout autre format (`code_insee` sur 3 caractères, caractères inattendus…) → pas de correction, la validation contre `silver.ref_geo` le marque `code_geo_inconnu` (cf. §4.3).
- Pas concernés : départements 10–95 (pas de zéro de tête), Corse (`2A004` contient une lettre, gardé en texte par Excel), DROM (`971…` commence par 9), `code_epci` (SIREN commençant par 2).
- Corse : `2A` / `2B`, jamais `20`.
- DROM : départements sur 3 chiffres (`971`…) ; reconstruction `code_insee` depuis DVF à vérifier sur échantillon.
- Arrondissements municipaux (Paris `751xx`, Lyon `6938x`, Marseille `132xx`) : on garde le code arrondissement **et** on ajoute `code_insee_commune` (75056, 69123, 13055) pour agréger à la commune.
- `code_departement` (2-3 car.), `code_region` (2 car.), `code_epci` (9 car.) : string.

> **À valider** sur les données réelles :
> - DVF brut : format de `Code commune` (sans zéros de tête ?) et de `Code departement` (`01` ou `1` ?) — conditionne la règle de construction.
> - Sources xlsx : liste des fichiers où le zéro est réellement perdu (`code_insee`, `code_departement` du chômage INSEE, `code_region` des tableaux régionaux).
> - Nb de lignes portant `code_geo_complete` / `code_geo_inconnu` par source : si `code_geo_inconnu` est massif, la règle de complétion est incomplète.

### 4.3 Millésime COG unique

Les communes fusionnent/se scindent chaque année : un code INSEE de 2018 peut ne plus exister en 2026.

- On fixe un **COG de référence** = millésime de l'Admin Express chargé (ex. COG 2026).
- La table de passage INSEE (mouvements de communes) est chargée dans `silver.ref_passage_cog` : `code_insee_origine → code_insee_reference`.
- Chaque table Silver porte `code_insee_source` (tel que reçu) **et** `code_insee` (ramené au COG de référence).
- Code introuvable dans le référentiel après passage → `flag_qualite` contient `code_geo_inconnu`.

### 4.4 Colonnes de qualité et de lineage

| Colonne | Rôle |
|---|---|
| `flag_qualite` | tableau de codes d'anomalie (`prix_aberrant`, `surface_nulle`, `geocodage_faible`…). Vide = ligne saine. |
| `est_exploitable` | booléen dérivé : `true` si aucun flag bloquant. Gold ne lit que ces lignes. |
| `_source`, `_batch_id` | repris de Bronze. |
| `_date_maj_silver` | date du dernier `MERGE` sur la ligne. |

### 4.5 Dédoublonnage et clés

- Chaque table a une **clé métier** déclarée (cf. §5). Doublons sur la clé → on garde le lot le plus récent (`_date_ingestion` max).
- Écriture par `MERGE` sur la clé métier → idempotent, rejouable.

## 5. Traitements par entité

### 5.1 `silver.ref_geo` (Admin Express + COG)

- Une ligne par zone, tous niveaux : `niveau_geo ∈ {commune, arrondissement, epci, departement, region}`, `code_geo`, `nom`, `code_parent` (hiérarchie commune → département → région, commune → EPCI).
- Géométrie reprojetée en **EPSG:4326** (WGS84, attendu par Leaflet) ; on garde aussi une version EPSG:2154 (Lambert 93) pour les calculs de surface/distance en mètres.
- Géométrie validée (`ST_IsValid`, sinon `ST_MakeValid`).
- Centroïde calculé (bulles, étiquettes).
- **Clé** : `(niveau_geo, code_geo)`.

### 5.2 `silver.transaction` (DVF)

1. **Regrouper en mutation** : DVF = une ligne par lot/local. Clé de mutation = `(date_mutation, numero_disposition, valeur_fonciere, code_commune, section, no_plan)` ou `id_mutation` si on utilise DVF géolocalisé (Etalab).
2. **Nature de mutation** : garder `Vente` et `Vente en l'état futur d'achèvement`. Autres (`Échange`, `Adjudication`, `Expropriation`…) → `flag_qualite = hors_marche`.
3. **Type de bien** : calculé depuis les locaux de la mutation : `appartement`, `maison`, `local_commercial`, `terrain`, `mixte` (plusieurs types → prix non attribuable).
4. **Surface** : somme des `surface_reelle_bati` des locaux du type principal ; dépendances exclues.
5. **Prix/m²** = `valeur_fonciere / surface` uniquement pour les mutations **mono-type** (`appartement` ou `maison`) ; sinon `null`.
6. **Aberrations** (flag, pas suppression) :
   - `valeur_fonciere` ≤ 1 000 € ou > 50 M€
   - surface < 9 m² (seuil légal habitable) ou > 1 000 m² pour un appartement
   - prix/m² hors bornes de Tukey sur `log(prix_m2)` : `[Q1 − 1,5·IQR ; Q3 + 1,5·IQR]`, calculées par **(département, année, type_bien)** (bornes calculées, pas fixes) :
     - Pas de percentiles fixes (P1/P99) : ils flaguent par construction 2 % des ventes, même toutes légitimes. Tukey ne flague que les valeurs réellement isolées (possiblement aucune).
     - Échelle log : la distribution des prix/m² est très asymétrique ; en échelle brute on flaguerait des biens chers légitimes et presque aucun bien anormalement bas.
     - Effectif minimum : groupe < 30 ventes → bornes calculées au niveau région.
     - Bornes et flags **recalculés sur toute la partition (département, année)** à chaque chargement, pas sur le seul lot : le millésime DVF de l'année en cours est partiel et republié chaque semestre.
7. **Géocodage** : jointure sur `silver.adresse` (BAN) → `id_ban`, `lat`, `lon`, `score_geocodage`. Score < 0.6 → flag `geocodage_faible`. Échec → `null`, ligne gardée (rattachée par `code_insee`).
- **Clé** : `id_mutation`.

> **À valider** (étape 6) : `k = 1,5` et seuil de 30 ventes sont des valeurs de départ. À confirmer sur DVF réel : part de ventes flaguées `prix_aberrant` par département (attendu ~1 %), examen d'un échantillon de lignes flaguées. Si trop de faux positifs → `k = 3`.

> Option à évaluer : charger **DVF géolocalisé (Etalab)** au lieu du DVF brut DGFiP — déjà regroupé par `id_mutation`, code commune sur 5 caractères et lat/lon fournis. Économise les étapes 1 et 7, mais dépend d'un retraitement tiers (à documenter).

### 5.3 `silver.adresse` (BAN)

- Adresse normalisée, `id_ban`, `code_insee`, `lat`, `lon`, `score`, `type_resultat` (`housenumber`, `street`, `municipality`).
- Résultat `municipality` (seule la commune a été trouvée) → flag `geocodage_commune_seule`.
- **Clé** : `adresse_norm` (adresse d'entrée normalisée) → évite de re-géocoder la même adresse.

### 5.4 `silver.revenu_commune` (Filosofi)

- Une ligne par `(code_insee, annee)`.
- Valeurs secrétisées → `null` + `_statut = secret` (jamais 0 : un 0 ferait chuter les moyennes).
- `nb_menages_fiscaux` gardé : sert de seuil de fiabilité en Gold.
- Contrôle de cohérence : `D1 ≤ médiane ≤ D9`, sinon flag.

### 5.5 `silver.dpe_logement` (API ADEME)

- Dédoublonnage sur `numero_dpe` (un DPE remplacé → garder le plus récent).
- Classes `A`–`G` validées (autre valeur → `null` + flag).
- DPE antérieurs au 1/07/2021 (ancienne méthode) → `methode = ancienne`, à ne pas mélanger sans le signaler.
- Rattachement adresse → `id_ban` via BAN.

### 5.6 `silver.avis`

1. **Extraction** depuis `contenu_html` Bronze : texte, note, date, auteur pseudonymisé (hash), lieu → `code_insee`.
2. **Nettoyage texte** : HTML, URLs, emojis (convertis ou retirés), espaces ; détection de langue (on garde `fr`, autres flaggés).
3. **Dédoublonnage** : hash du `(texte_norm, lieu, date)`.
4. **Données personnelles** : pas de nom d'auteur en clair (RGPD), uniquement un hash.
5. **Enrichissement NLP** : `sentiment_label`, `sentiment_score`, `themes[]`, `modele_nlp` + `version_modele` (pour savoir quel modèle a produit le score, et rejouer si on change de modèle).
- **Clé** : `id_avis` (hash).

### 5.7 Autres sources

| Entité | Points d'attention |
|---|---|
| `silver.population` | 1 ligne `(code_insee, annee)`, mise au COG. |
| `silver.chomage` | grain natif **département × trimestre** (pas de chômage communal : ne pas inventer). |
| `silver.risque_commune` | `(code_insee, type_risque)`, niveau normalisé `faible / moyen / fort`. |
| `silver.equipement` | écoles + gares dans une même entité : `type_equipement`, `lat/lon`, `code_insee`, attributs spécifiques en colonne JSON. |
| `silver.annonce` (Orpi) | prix **demandé** ; `date_premiere_vue`, `date_derniere_vue` (calculées sur les snapshots quotidiens) → durée de mise en vente. |

## 6. Contrôles de sortie

| Contrôle | Bloquant |
|---|---|
| Clé métier unique | oui |
| `code_insee` présent dans `silver.ref_geo` pour ≥ 99 % des lignes | oui |
| Types conformes (aucune colonne string là où un type est attendu) | oui |
| Part de lignes `est_exploitable = false` < seuil par source (ex. 15 % DVF) | non (alerte) |
| Distribution prix/m² par département stable vs lot précédent | non (alerte) |

## 7. Livrable "méthodologie de nettoyage"

Ce fichier en est la base. Pour le livrable, ajouter pour chaque règle : nb de lignes concernées sur les données réelles (calculé à partir des `flag_qualite`), exemples avant/après.

## 8. Definition of done

- [ ] `silver.ref_geo` chargé et validé avant toute autre table.
- [ ] Chaque table : clé métier, `MERGE`, colonnes qualité/lineage.
- [ ] Toutes les tables joignables sur `code_insee` (même COG).
- [ ] Rapport de qualité par lot (nb lignes par flag).
