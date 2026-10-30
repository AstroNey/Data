# 06 — Serving : bases exposées à l'app

Précédent : [05_Gold.md](05_Gold.md) · Vue d'ensemble : [Data_flow.md](Data_flow.md) · Suivant : [07_NLP.md](07_NLP.md)

---

## 1. Rôle

Copier les tables utiles du lake vers des bases **optimisées pour la lecture interactive**. Le lake (Delta) est fait pour les gros calculs Spark, pas pour répondre en 100 ms à un clic sur une carte.

- L'app Streamlit **ne lit que le Serving**, jamais le lake.
- Le Serving est **jetable** : il se reconstruit entièrement depuis Silver/Gold.
- Aucune transformation métier ici : uniquement chargement, index, droits.

## 2. Répartition

| Base | Reçoit | Pourquoi |
|---|---|---|
| **PostgreSQL + PostGIS** | `silver.ref_geo` (géométries), toutes les tables `gold.*`, `gold.transaction_carte`, `silver.equipement` | Données tabulaires, filtres/tri, jointures, requêtes spatiales (zone cliquée, rayon autour d'un point). |
| **MongoDB** | `silver.avis` (texte + NLP), `silver.annonce` (Orpi, structure variable) | Documents à structure variable, recherche plein texte, affichage d'avis bruts. |

Les tables Silver volumineuses (transactions complètes, DPE) **restent dans le lake** : l'app n'en a pas besoin ligne à ligne.

## 3. Chargement PostgreSQL

### 3.1 Méthode

1. Spark écrit dans une table de staging : `stg_<table>` via JDBC (`mode=overwrite`, `batchsize=10000`).
2. Création des index sur `stg_<table>`.
3. Bascule **atomique** dans une transaction :
   ```sql
   BEGIN;
   ALTER TABLE <table> RENAME TO <table>_old;
   ALTER TABLE stg_<table> RENAME TO <table>;
   DROP TABLE <table>_old;
   COMMIT;
   ```
   → l'app ne voit jamais une table à moitié chargée.
4. `ANALYZE <table>` (statistiques pour le planificateur).

### 3.2 Géométries

- Stockées en Delta en WKB → converties à l'insertion : `ST_SetSRID(ST_GeomFromWKB(geom), 4326)`.
- Deux versions par zone pour la carte : géométrie complète + géométrie **simplifiée** (`ST_SimplifyPreserveTopology`) selon l'échelle (région / département / commune) → GeoJSON léger pour Leaflet.

### 3.3 Index

| Table | Index | Requête servie |
|---|---|---|
| `ref_geo` | GIST sur `geom` | "quelle zone contient ce point", carte |
| `ref_geo` | btree `(niveau_geo, code_geo)`, btree `code_parent` | navigation hiérarchique |
| `gold.*` | btree `(niveau_geo, code_geo, periode)` | fiche zone |
| `gold.*` | btree `(niveau_geo, periode)` | carte choroplèthe d'un niveau |
| `transaction_carte` | GIST sur `geom`, btree `date_mutation` | carte de points / chaleur |

### 3.4 Organisation

- Schémas PostgreSQL : `ref` (référentiel), `gold` (indicateurs), `meta` (copie de la table `_runs`, date de fraîcheur affichée dans l'app).
- Vues SQL légères pour l'app (ex. `v_carte_prix` = `gold.indicateurs_prix` ⨝ `ref_geo` simplifiée) → l'app ne connaît que les vues.

## 4. Chargement MongoDB

- Connecteur Spark MongoDB, `mode=overwrite` sur une collection de staging puis `renameCollection(..., dropTarget=true)` → bascule atomique.
- Un document = un avis : texte, note, date, `code_insee`, bloc `nlp {label, score, themes, modele}`, bloc `_lineage {source, batch_id, url}`.
- Index :
  - `{ code_insee: 1, date: -1 }` → avis d'une commune, les plus récents d'abord
  - index **texte** sur `texte` (langue `french`) → recherche plein texte
  - `{ "nlp.label": 1 }` → filtre positif / négatif

## 5. Accès

- Un utilisateur **lecture seule** par base pour l'app (`homepedia_app`), un utilisateur écriture pour les jobs (`homepedia_etl`).
- Identifiants en variables d'environnement (`.env` hors git), jamais dans le code.
- Côté Streamlit : `st.cache_data` avec TTL sur les requêtes (les données changent au mieux une fois par jour).

## 6. Fraîcheur

`meta.fraicheur` : pour chaque table servie, date du dernier chargement + `batch_id` des sources + millésime des données. Affiché dans l'app ("Données DVF jusqu'au 31/12/2025, mis à jour le 19/10/2026").

## 7. Contrôles de sortie

| Contrôle | Bloquant |
|---|---|
| Nb lignes Serving = nb lignes Gold/Silver source | oui (avant la bascule) |
| Géométries valides (`ST_IsValid`) | oui (avant la bascule) |
| Requêtes types de l'app < 500 ms (fiche zone, carte département) | non (alerte) |

## 8. Definition of done

- [ ] Bases recréables entièrement par un seul job ("rebuild serving").
- [ ] Chargement par staging + bascule atomique.
- [ ] Index créés pour chaque requête de l'app.
- [ ] Utilisateur lecture seule pour l'app.
- [ ] Table de fraîcheur alimentée.
