# Homepedia — Flux de gestion des données

Étape 0 du projet : définir **comment la donnée circule** de la source jusqu'à l'app, avant d'écrire les scripts de collecte. La modélisation des tables découle de ce flux (couches Silver / Gold), pas l'inverse.

## Détail par étape

| Fichier | Contenu |
|---|---|
| [01_Landing.md](01_Landing.md) | Dépôt des fichiers bruts, manifest, méthodes de collecte (dataset / API / scraping) |
| [02_Bronze.md](02_Bronze.md) | Conversion en Delta, tout en string, colonnes techniques, idempotence |
| [03_Silver.md](03_Silver.md) | Nettoyage, codes géo, COG unique, flags qualité, traitements par entité (base du livrable nettoyage) |
| [04_Gold.md](04_Gold.md) | Définition des indicateurs, agrégation multi-échelle, seuils de significativité |
| [05_Serving.md](05_Serving.md) | Chargement PostgreSQL/PostGIS + MongoDB, index, bascule atomique, accès |
| [06_Transverse.md](06_Transverse.md) | Arborescence du dépôt, nommage, orchestration, qualité, lineage, environnements |
| [07_NLP.md](07_NLP.md) | Modèles NLP appliqués aux avis (langue, sentiment, thèmes), évaluation, sortie |
| [08_Application.md](08_Application.md) | App cible : pages, questions auxquelles elle répond, interactions |

---

## 1. Choix d'architecture

### Retenu : Medallion (Lakehouse) + couche de serving

```
Sources ──► LANDING ──► BRONZE ──► SILVER ──► GOLD ──► SERVING ──► App Streamlit
           (fichiers    (Delta,    (Delta,    (Delta,  (PostgreSQL/PostGIS
            bruts)      brut       nettoyé,   agrégats  + MongoDB)
                        tracé)     conforme)  métier)
           └──────────── Data Lake (Spark) ────────────┘
```

**Pourquoi Medallion** :
- Sources **majoritairement batch** (DVF semestriel, Filosofi/Admin Express annuels) + quelques API/scraping → pas besoin de streaming.
- Spark est **obligatoire** dans le sujet → Medallion est le pattern natif Spark/Databricks (Delta Lake).
- Chaque couche = un niveau de qualité clair → répond directement aux livrables "méthodologie de nettoyage" et "traçabilité".
- On peut **rejouer** n'importe quelle couche depuis la précédente (bug de nettoyage → on recalcule Silver depuis Bronze, sans re-télécharger).

### Alternatives écartées

| Pattern | Principe | Pourquoi pas ici |
|---|---|---|
| **Lambda** | Pipeline batch + pipeline temps réel en parallèle | Double code à maintenir, aucune source vraiment temps réel. |
| **Kappa** | Tout en streaming (Kafka) | Sources = fichiers annuels/semestriels → streaming artificiel. |
| **Data Vault 2.0** | Hubs / Links / Satellites, historisation totale | Très lourd à modéliser, pensé pour entrepôts d'entreprise multi-équipes. |
| **Data Mesh** | Données décentralisées par domaine métier | Pattern d'organisation (plusieurs équipes), pas d'architecture technique. |
| **ELT simple** (staging → marts, style dbt) | Proche du medallion, sans lake | Pas de stockage brut immuable pour les fichiers lourds (GPKG, DVF) ni d'intégration Spark native. |

Medallion n'est pas "dépassé" : c'est le standard actuel des lakehouses. L'amélioration réelle consiste à l'**encadrer** : zone Landing en amont, couche Serving en aval, et des règles transverses (qualité, idempotence, lineage) — c'est ce qu'on fait ci-dessous.

---

## 2. Les couches

### 2.0 Landing — fichiers tels que reçus

- **Contenu** : copie exacte de la source (CSV DVF, xlsx Filosofi, GPKG Admin Express, JSON des API, HTML scrapé).
- **Règle** : **immuable**, jamais modifié, jamais supprimé (sauf purge volontaire documentée).
- **Arborescence** : `landing/<source>/<date_ingestion>/<fichier>` — ex. `landing/dvf/2026-10-18/valeursfoncieres-2025.txt`.
- **Pourquoi** : preuve de provenance + possibilité de tout reconstruire si une couche aval est corrompue.

### 2.1 Bronze — brut, structuré, tracé

- **Contenu** : même données que Landing, converties en tables Delta. **Aucun nettoyage**, toutes colonnes en `string` (schema-on-read).
- **Colonnes techniques ajoutées** (toutes les tables) :

| colonne | rôle |
|---|---|
| `_source` | nom de la source (`dvf`, `filosofi`, `dpe_api`…) |
| `_fichier_source` / `_url_source` | fichier Landing ou URL d'origine |
| `_date_ingestion` | timestamp du chargement |
| `_batch_id` | identifiant du run (permet de rejouer / annuler un lot) |

- **Mode** : append-only (on empile les lots).

### 2.2 Silver — nettoyé, typé, conforme

C'est ici que vit la **méthodologie de nettoyage** (livrable). Une table Silver par entité métier (`transaction`, `commune`, `revenu_commune`, `dpe_logement`, `avis`…).

Traitements :
- **Typage** : dates ISO, numériques (virgule décimale FR → point), booléens.
- **Normalisation des codes géo** : `code_insee` sur 5 caractères (zéros de tête absents dans les sources : Excel, `Code commune` DVF ; complétion seulement si non ambiguë, sinon flag), Corse `2A`/`2B`, arrondissements Paris/Lyon/Marseille.
- **Mise au COG de référence** : les communes fusionnent chaque année → table de passage INSEE pour ramener toutes les sources au **même millésime COG** (sinon jointures cassées entre DVF 2018 et Admin Express 2026).
- **Dédoublonnage** : ex. DVF = une ligne par lot/local → regrouper en mutation.
- **Filtrage des aberrations** : prix/m² hors bornes, surfaces nulles, mutations hors vente (échanges, adjudications…) → **marquées** (`flag_qualite`), pas supprimées.
- **Géocodage BAN** : adresse → `id_ban`, lat/lon, `score_matching` (nullable si échec : on garde la ligne, rattachée par `code_insee`).
- **Avis** : nettoyage texte (HTML, emojis, langue) + **enrichissement NLP** (sentiment, thèmes) → champs `nlp.*`.

### 2.3 Gold — prêt pour l'analyse

- **Contenu** : indicateurs métier agrégés, directement consommables par l'app.
- **Grain** : `(niveau_geo, code_geo, periode)` avec `niveau_geo ∈ {commune, epci, departement, region}` → multi-échelle demandé par le sujet.
- **Exemples** :
  - `indicateurs_prix` : prix/m² médian, nb ventes, évolution N-1, par type de bien.
  - `indicateurs_socio` : revenu médian, taux pauvreté, chômage, ratio prix/revenu.
  - `indicateurs_sentiment` : agrégat NLP des avis (nb avis, score moyen, part négatifs, thèmes).
  - `indicateurs_logement` : DPE, risques, équipements (écoles, gares).
  - `profil_zone` : vue large croisant toutes les tables précédentes.
- **Règle** : jamais de calcul lourd côté app — tout agrégat affiché est pré-calculé ici par Spark.

### 2.4 Serving — bases exposées à l'app

| Base | Reçoit | Pourquoi |
|---|---|---|
| **PostgreSQL + PostGIS** | Référentiel géo + tout le Gold (dont échantillon de transactions pour la carte) | Jointures, intégrité, requêtes spatiales, filtres de l'app. |
| **MongoDB** | Silver `avis` (texte + champs NLP) + `annonce` (Orpi) | Texte libre, structure variable, full-text. |

L'app Streamlit **ne lit que la couche Serving**, jamais le lake.

---

## 3. Placement des sources (étapes 1-3)

| Source | Type | Fréquence | Bronze | Silver | Gold / Serving |
|---|---|---|---|---|---|
| DVF | dataset | semestrielle (avril/oct) | `bronze.dvf` | `transaction` | `indicateurs_prix` |
| Admin Express + COG | dataset GPKG | annuelle | `bronze.admin_express_*` | `ref_geo` (+ géométries), `ref_passage_cog` | PostGIS |
| Filosofi | dataset | annuelle | `bronze.filosofi` | `revenu_commune` | `indicateurs_socio` |
| Avis (scraping) | scraping | ad hoc | `bronze.avis_raw` | `avis` (+ NLP) | MongoDB + `indicateurs_sentiment` |
| Population INSEE | dataset | annuelle | `bronze.insee_population` | `population` | `indicateurs_socio` |
| DPE | API | incrémentale (hebdo) | `bronze.dpe_api` | `dpe_logement` | `indicateurs_logement` |
| Argiles (Géorisques) | dataset | rare | `bronze.georisques_argiles` | `risque_commune` | `indicateurs_logement` |
| Écoles / Gares | dataset | annuelle | `bronze.ecoles`, `bronze.gares` | `equipement` | `indicateurs_logement` |
| Chômage INSEE | dataset | trimestrielle | `bronze.insee_chomage` | `chomage` | `indicateurs_socio` |
| BAN | dataset/API | à la demande | `bronze.ban` | `adresse` (géocodage) | — |
| Orpi (scraping) | scraping | snapshot quotidien | `bronze.orpi_raw` | `annonce` | MongoDB, comparaison prix demandé / vendu |

---

## 4. Règles transverses

1. **Idempotence** : relancer un job sur le même lot donne le même résultat (écriture par `overwrite` de partition ou `MERGE` sur clé, jamais d'append aveugle en Silver/Gold).
2. **Incrémental** : chaque job ne traite que les nouveaux `_batch_id` / nouvelles partitions (indispensable pour DVF 20M+ lignes et DPE +35k/semaine).
3. **Qualité de données** : contrôles automatiques entre couches (non-nullité des clés, `code_insee` existant dans le référentiel, bornes de prix, nb de lignes vs lot précédent). Un lot qui échoue **ne passe pas** à la couche suivante.
4. **Lineage** : les colonnes `_source` / `_batch_id` suivent la donnée jusqu'au Gold → on sait d'où vient chaque chiffre affiché.
5. **Time travel** : Delta Lake garde l'historique des versions → retour arrière possible après un mauvais run.
6. **Référentiel d'abord** : Admin Express + COG chargés **avant** toute autre source Silver (toutes les jointures en dépendent).

---

## 5. Stack proposée

| Brique | Choix | Alternative |
|---|---|---|
| Stockage lake (Landing → Gold) | Delta Lake sur **Databricks Free Edition** (Volumes / Unity Catalog) | Disque local / MinIO |
| Traitement | PySpark sur Databricks (compute serverless de l'offre gratuite) | Cluster Spark Docker multi-nœuds |
| Collecte | Scripts Python en **Docker local**, dépôt vers Landing | — |
| Orchestration | Prefect en Docker local (déclenche collecte + jobs Databricks) | Databricks Workflows, Airflow |
| Qualité | Checks PySpark maison / Great Expectations | Soda |
| Serving | PostgreSQL 16 + PostGIS, MongoDB en **Docker local** | — |
| NLP | Hugging Face Transformers via pandas UDF (cf. [07_NLP.md](07_NLP.md)) | — |
| App | Streamlit + Folium (Leaflet) en Docker local (cf. [08_Application.md](08_Application.md)) | — |

### Répartition des environnements

```
 Docker local                          Databricks Free
┌──────────────────────┐   upload    ┌───────────────────────────┐
│ collecte (scripts)   │───────────► │ Landing → Bronze → Silver │
│                      │             │          → Gold  (Delta)  │
│ Serving              │ ◄────────── │                           │
│ PostgreSQL + MongoDB │  export Gold│                           │
│ App Streamlit        │  / Silver   └───────────────────────────┘
└──────────────────────┘
```

- **Databricks Free** : stockage médaillon + calculs Spark.
- **Docker local** : tout ce qui est collecte et serving, pour contourner les limites de l'offre gratuite (pas de scraping ni de bases externes, pas d'hébergement d'app).
- Points de contact à définir : dépôt des fichiers collectés vers les Volumes Databricks (API / CLI Databricks) et export Gold/Silver vers le Serving local (JDBC impossible depuis Databricks vers localhost → export Parquet/CSV puis chargement local).

### Point ouvert : cluster / calcul distribué

Le sujet impose Spark et parle de distribuer sur plusieurs nœuds. Databricks Free n'expose pas de cluster configurable (compute serverless, non dimensionnable). **À trancher avec le pédago** :
- soit Databricks Free suffit (Spark réel, distribué en interne par la plateforme) ;
- soit on ajoute un cluster Spark Docker multi-nœuds (1 master + N workers) en local pour démontrer la distribution.

Le code PySpark reste identique dans les deux cas.

---

## 6. Conséquence sur la collecte

Les scripts de collecte écrivent **uniquement en Landing**. Tout le reste (structuration, nettoyage, agrégation) est fait par les jobs Spark des couches suivantes.

## 7. Ordre de mise en place

1. Arborescence lake (`landing/`, `bronze/`, `silver/`, `gold/`) + conventions de nommage.
2. Pipeline complet **sur une seule source** de bout en bout : Admin Express (référentiel) puis DVF → valider le pattern.
3. Dupliquer le pattern pour Filosofi + avis → MVP étape 1.
4. Orchestration + contrôles qualité.
5. Ajouter les sources étapes 2-3 une par une.
