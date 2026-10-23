# 02 — Bronze : brut, structuré, tracé

Précédent : [01_Landing.md](01_Landing.md) · Vue d'ensemble : [Data_flow.md](Data_flow.md) · Suivant : [03_Silver.md](03_Silver.md)

---

## 1. Rôle

Transformer les fichiers Landing (formats hétérogènes) en **tables Delta lisibles par Spark**, sans changer le contenu.

- Un seul format en aval (Delta) quel que soit le format source.
- Contenu **fidèle** à la source : mêmes lignes, mêmes valeurs, mêmes erreurs.
- Ajout des métadonnées de traçabilité.

> Règle d'or : si une valeur est fausse dans la source, elle est fausse en Bronze. On corrige en Silver.

## 2. Ce qui entre / ce qui sort

| Entrée | Sortie |
|---|---|
| Dossier Landing avec `_manifest.json` non encore chargé | Table Delta `bronze.<source>[_<objet>]` |

## 3. Emplacement et nommage

```
lake/bronze/<table>/          # table Delta
```

| Source | Table(s) Bronze | Remarque |
|---|---|---|
| DVF | `bronze.dvf` | un fichier annuel → mêmes colonnes, empilés |
| Admin Express | `bronze.admin_express_commune`, `_departement`, `_region`, `_epci`, `_arrondissement_municipal` | une table par couche GPKG |
| Filosofi | `bronze.filosofi` | onglet utile du xlsx |
| Avis | `bronze.avis_raw` | 1 ligne = 1 page HTML (contenu complet en string) |
| DPE | `bronze.dpe_api` | 1 ligne = 1 résultat de l'API (JSON aplati niveau 1) |
| Population, chômage, écoles, gares, argiles | `bronze.<source>` | |
| BAN | `bronze.ban` | résultats de géocodage |
| Orpi | `bronze.orpi_raw` | 1 ligne = 1 page annonce HTML |

## 4. Traitement

### 4.1 Lecture selon le format

| Format | Lecture | Pièges |
|---|---|---|
| TXT/CSV (DVF) | `spark.read.csv(sep="\|", header=True, inferSchema=False)` | Séparateur `\|`, encodage à vérifier (UTF-8 vs latin-1). |
| xlsx (Filosofi, INSEE) | pandas/openpyxl → Spark DataFrame | Lignes d'en-tête/notes au-dessus du tableau : sauter les N premières lignes (paramètre par source). |
| GPKG (Admin Express) | GeoPandas (`read_file(layer=...)`) → géométrie en **WKB** + colonne `_crs` | Spark ne lit pas le GPKG nativement. Garder le CRS d'origine, reprojection en Silver. |
| JSON (API) | `spark.read.json` | Objets imbriqués : aplatir seulement le 1er niveau, garder le reste en string JSON. |
| HTML (scraping) | Lecture texte complète | **Pas de parsing ici** : l'extraction des champs est faite en Silver. |

### 4.2 Typage : tout en `string`

Aucune inférence de type. Raisons :
- `inferSchema` transforme `01004` en `1004` (zéro de tête perdu → code INSEE faux).
- `"1 234,50"` (format FR) échoue en numérique → ligne perdue ou `null` silencieux.
- Une source qui change un type d'une année sur l'autre ne casse pas le chargement.

Seule exception : la géométrie (binaire WKB).

### 4.3 Noms de colonnes

- Noms d'origine **conservés en sens**, mais rendus compatibles Delta : minuscules, espaces et accents → `_`.
  Ex. DVF `Valeur fonciere` → `valeur_fonciere`, `Code departement` → `code_departement`.
- Le mapping (nom original → nom Bronze) est généré automatiquement et loggé.

### 4.4 Colonnes techniques

Ajoutées à **chaque** ligne de **chaque** table :

| Colonne | Type | Contenu |
|---|---|---|
| `_source` | string | `dvf`, `filosofi`… |
| `_batch_id` | string | repris du manifest Landing |
| `_fichier_source` | string | chemin du fichier dans Landing |
| `_url_source` | string | URL d'origine (fichier, page API, page scrapée) |
| `_date_ingestion` | timestamp | date du dépôt Landing |
| `_date_chargement_bronze` | timestamp | date d'exécution du job Bronze |
| `_num_ligne` | long | position dans le fichier source (debug) |
| `_millesime` | string | millésime des données (manifest) |

### 4.5 Écriture

- Delta, partitionné par `_batch_id` (et `_millesime` pour DVF, ~20M lignes).
- **Append** d'un nouveau lot.
- **Idempotence** : avant écriture, suppression des lignes du même `_batch_id` (`replaceWhere "_batch_id = '...'"`). Relancer un lot = le remplacer, jamais le doubler.
- Le lot est inscrit dans la table de suivi `_runs` (cf. [06_Transverse.md](06_Transverse.md) §4).

### 4.6 Évolution de structure

Une source qui ajoute une colonne → `mergeSchema=true` (colonne ajoutée, `null` pour les anciens lots).
Une source qui **supprime ou renomme** une colonne → le job échoue volontairement : à traiter à la main (mapping en Silver).

## 5. Spécificités par source

- **DVF** : fichier annuel unique par millésime. Une mutation (vente) = plusieurs lignes (une par lot / local / nature de culture). On garde ce découpage tel quel.
- **Admin Express** : charger **aussi** la couche `ARRONDISSEMENT_MUNICIPAL` (Paris/Lyon/Marseille : DVF localise par arrondissement).
- **Filosofi** : valeurs secrétisées (petites communes) sous forme de `s`, `nd` ou vide → gardées telles quelles en string.
- **DPE API** : un même DPE peut revenir dans deux lots (recouvrement des fenêtres de date) → doublons gardés en Bronze, dédoublonnés en Silver.
- **Avis / Orpi** : HTML complet en une colonne `contenu_html`, même si lourd.

## 6. Contrôles de sortie

| Contrôle | Bloquant |
|---|---|
| Nb lignes Bronze du lot = nb lignes lues dans le(s) fichier(s) | oui |
| Colonnes attendues présentes (liste par source) | oui |
| Aucune colonne technique nulle | oui |
| Nb lignes cohérent avec le lot précédent de la même source (± 30 %) | non (alerte) |

## 7. Definition of done

- [ ] Un job Bronze générique paramétré par source (format, séparateur, lignes à sauter, couches).
- [ ] Toutes les tables Bronze ont les colonnes techniques.
- [ ] Relancer un lot ne crée pas de doublon.
- [ ] Lecture possible de n'importe quel lot via `_batch_id` (rejeu Silver).
