# 06 — Règles transverses : conventions, orchestration, qualité, lineage

Vue d'ensemble : [Data_flow.md](Data_flow.md) · Couches : [Landing](01_Landing.md) · [Bronze](02_Bronze.md) · [Silver](03_Silver.md) · [Gold](04_Gold.md) · [Serving](05_Serving.md)

---

## 1. Arborescence du dépôt

```
Data/
├── collect/            # scripts de collecte → Landing uniquement (1 fichier par source)
├── jobs/
│   ├── bronze/         # job générique + config par source
│   ├── silver/         # 1 job par entité
│   ├── gold/           # 1 job par table d'indicateurs
│   └── serving/        # chargement PostgreSQL / MongoDB
├── quality/            # contrôles de qualité par couche
├── config/
│   └── sources.yaml    # registre des sources (cf. §2)
├── orchestration/      # DAGs
├── app/                # Streamlit
├── doc/
└── lake/               # hors git : landing/ bronze/ silver/ gold/
```

## 2. Registre des sources

Un seul fichier `config/sources.yaml` décrit toutes les sources. Les jobs génériques (collecte, Bronze) le lisent au lieu d'avoir des paramètres en dur.

```yaml
dvf:
  type: dataset            # dataset | api | scraping
  dataset_slug: demandes-de-valeurs-foncieres   # résolu via /api/1/datasets/<slug>/
  frequence: semestrielle
  format: csv
  separateur: "|"
  encodage: utf-8
  tables_bronze: [dvf]
  entites_silver: [transaction]
  responsable: <prénom>
  licence: Licence Ouverte 2.0
```

Liste fermée des noms de source : `admin_express`, `cog`, `dvf`, `ban`, `filosofi`, `insee_population`, `insee_chomage`, `dpe_api`, `georisques_argiles`, `ecoles`, `gares`, `avis`, `orpi`.

## 3. Conventions de nommage

| Élément | Convention | Exemple |
|---|---|---|
| Source | snake_case, sans accent | `dpe_api` |
| Table | `<couche>.<nom>` | `silver.transaction` |
| Colonne métier | snake_case français | `prix_m2_median` |
| Colonne technique | préfixe `_` | `_batch_id` |
| Colonne de statut | suffixe `_statut` | `revenu_median_uc_statut` |
| Batch | `<source>_<YYYYMMDDTHHMMSS>` | `dvf_20261019T083000` |
| Codes géo | toujours string | `code_insee = "01004"` |

## 4. Suivi des runs

Table Delta `meta._runs`, une ligne par exécution de job :

| Colonne | Contenu |
|---|---|
| `run_id` | identifiant unique |
| `batch_id` | lot traité |
| `source`, `couche`, `job` | quoi |
| `debut`, `fin`, `statut` | `succes`, `echec`, `bloque_qualite` |
| `nb_lignes_entree`, `nb_lignes_sortie` | volumes |
| `nb_lignes_flaggees` | qualité (Silver) |
| `message` | erreur ou résumé des contrôles |

Sert à : savoir quels lots restent à traiter à chaque couche (incrémental), produire le rapport qualité, afficher la fraîcheur dans l'app.

## 5. Orchestration

### 5.1 DAG par source

```
collect ─► check_landing ─► bronze ─► check_bronze ─► silver ─► check_silver ─► gold ─► check_gold ─► serving
```

- Chaque `check_*` bloque la suite si un contrôle bloquant échoue (statut `bloque_qualite`). Les couches aval gardent la version précédente, saine.
- Dépendance globale : le DAG `admin_express` + `cog` doit avoir réussi avant tout `silver` d'une autre source.
- Gold dépend de plusieurs sources → DAG `gold` séparé, déclenché quand une table Silver amont a changé.

### 5.2 Planification

| Source | Déclenchement |
|---|---|
| DVF | vérification hebdo, nouveau fichier ~ avril et octobre |
| Admin Express, COG, Filosofi, population, écoles, gares | vérification mensuelle (changent ~1×/an) |
| Chômage | vérification mensuelle (trimestriel) |
| DPE API | hebdomadaire, incrémental |
| Orpi | quotidien |
| Avis | manuel (campagnes de scraping) |

La vérification ne coûte rien grâce au `sha256` (cf. [01_Landing.md](01_Landing.md) §5.1) : fichier inchangé → rien ne tourne.

### 5.3 Outil

| Option | Pour | Contre |
|---|---|---|
| **Prefect** | Python pur, léger, UI locale, simple à démarrer | moins standard en entreprise |
| **Airflow** | standard du marché, valorisant | lourd (scheduler, webserver, DB), Docker indispensable |
| **Databricks Workflows** | intégré si on est sur Databricks | lié à la plateforme |

Recommandation : **Prefect** en local ; Databricks Workflows si le cluster est sur Databricks.

## 6. Qualité de données

### 6.1 Où

Un contrôle à chaque frontière de couche (détails dans chaque fichier, section "Contrôles de sortie").

### 6.2 Deux niveaux

- **Bloquant** : la donnée est fausse ou inutilisable (clé dupliquée, code géo inconnu, lignes perdues) → le lot s'arrête.
- **Alerte** : la donnée est suspecte (volume inhabituel, distribution qui bouge) → le lot passe, l'alerte est loggée dans `_runs`.

### 6.3 Outil

Checks PySpark maison dans `quality/` au départ (fonctions simples : `assert_unique`, `assert_not_null`, `assert_in_ref`, `assert_volume`). Passage à Great Expectations ou Soda seulement si le nombre de règles le justifie.

## 7. Lineage

- Les colonnes `_source` et `_batch_id` suivent chaque ligne de Bronze à Silver.
- En Gold, `batch_ids_sources` liste les lots ayant servi au calcul.
- Remonter d'un chiffre affiché à la source : `gold` → `batch_ids_sources` → `meta._runs` → manifest Landing → URL + `sha256` du fichier original.

## 8. Idempotence et rejeu

| Couche | Mode d'écriture | Rejouer un lot |
|---|---|---|
| Landing | nouveau dossier | interdit (immuable) |
| Bronze | `replaceWhere _batch_id` | remplace le lot |
| Silver | `MERGE` sur clé métier | met à jour les mêmes lignes |
| Gold | `overwrite` des partitions `periode` | recalcule |
| Serving | staging + bascule | recharge tout |

Delta **time travel** (`VERSION AS OF`) permet de revenir à l'état d'avant un mauvais run. Rétention : 30 jours (`VACUUM` au-delà).

## 9. Environnements

- **dev** : un échantillon (1 département, ex. Isère `38`) → jobs en quelques minutes, sur un poste.
- **full** : France entière, sur le cluster.
- Même code, paramètre `--perimetre=38` ou `--perimetre=france`.

## 10. Sécurité et conformité

- Secrets (mots de passe BDD, clés API) dans `.env`, hors git.
- Scraping : usage académique uniquement, `robots.txt` respecté, ToS consultées et notées dans le manifest.
- Avis : aucun nom d'auteur en clair (hash), pas de donnée personnelle dans Gold ni Serving.
- DVF : réutilisation encadrée (pas de réidentification des vendeurs/acheteurs, pas d'indexation nominative).
