# Homepedia — Récap : vocabulaire + ce qu'on a mis en place et pourquoi

Doc de synthèse pour s'y retrouver. Détails complets dans `TODO.md`, `spec/` et `schema/`.

---

## 1. Le projet en 3 lignes

- **Quoi** : app d'analyse du marché immobilier français (prix, socio-éco, risques, qualité de vie) à plusieurs échelles (commune / département / région).
- **Pour qui** : agences immo, collectivités, analystes — pas les particuliers acheteurs.
- **Focus** : DataOps → collecter des sources hétérogènes, les nettoyer, les stocker proprement (SQL + NoSQL), les traiter en big data (Spark), les visualiser (Streamlit + cartes).

---

## 2. Vocabulaire

### 2.1 Données & sources

| Terme | Définition | Pourquoi ça compte ici |
|---|---|---|
| **Open data** | Données publiques librement réutilisables (ici via data.gouv.fr). | Quasi toutes nos sources en viennent → gratuites, officielles, documentées. |
| **Dataset** | Fichier (ou lot de fichiers) téléchargeable : CSV, xlsx, GPKG… Données figées à une date. | Revenu Filosofi, Admin Express, DVF sont des datasets. |
| **API** | Service interrogeable par requête HTTP, renvoie des données à la demande (souvent JSON). | DPE et BAN sont des API → données à jour, requête ciblée (par adresse). |
| **Dataservice** | Nom data.gouv.fr pour une API publique référencée. | Même chose qu'une API côté usage. |
| **Scraping** | Récupérer des données en lisant automatiquement des pages web non prévues pour ça. | Avis Google/TripAdvisor, annonces Orpi. Toléré en usage académique seulement, attention aux ToS. |
| **ToS** (Terms of Service) | Conditions d'utilisation d'un site. | Le scraping peut les violer → on documente et on limite à l'usage académique. |
| **Sitemap** | Fichier XML listant toutes les URLs d'un site. | Point d'entrée du scraping Orpi (liste des annonces). |
| **Source officielle / non-officielle** | Officielle = publiée par un organisme public (INSEE, IGN, DGFiP…). Non-officielle = scraping, données commerciales. | Les non-officielles sont volatiles → besoin de traçabilité (`url_source`, `date_scraping`). |

### 2.2 Organismes & sources de données

| Sigle | Signification | Ce qu'on en tire |
|---|---|---|
| **DVF** | Demandes de Valeurs Foncières (publié par la **DGFiP**, Direction Générale des Finances Publiques) | Toutes les ventes immobilières réelles depuis 2014 : prix, surface, type, date, adresse. **Donnée centrale** du projet. |
| **INSEE** | Institut National de la Statistique et des Études Économiques | Codes communes, population, revenus (Filosofi), chômage. |
| **Filosofi** | Fichier Localisé Social et Fiscal de l'INSEE | Revenus des ménages par commune (médiane, déciles, pauvreté). |
| **IGN** | Institut national de l'information géographique et forestière | Admin Express (contours des communes/départements/régions). |
| **Admin Express COG** | Référentiel IGN des limites administratives, aligné sur le **COG** (Code Officiel Géographique de l'INSEE) | Les polygones pour dessiner les cartes. |
| **BAN** | Base Adresse Nationale (25M+ adresses) | Géocodage : transforme une adresse texte en coordonnées + code commune. |
| **ADEME** | Agence de la transition écologique | Publie les DPE. |
| **DPE** | Diagnostic de Performance Énergétique | Classe énergie A→G d'un logement (+ classe GES). |
| **GES** | Gaz à Effet de Serre | 2e note du DPE (émissions), A→G. |
| **CSTB / BDNB** | Centre Scientifique et Technique du Bâtiment / Base de Données Nationale des Bâtiments | Caractéristiques physiques des bâtiments (année, matériaux, nb logements). |
| **ANAH** | Agence Nationale de l'Habitat | Aides à la rénovation, opérations programmées par commune. |
| **OPAH / PIG** | Opération Programmée d'Amélioration de l'Habitat / Programme d'Intérêt Général | Types de dispositifs ANAH sur un territoire. |
| **Mes Aides Réno** | API qui calcule les aides à la rénovation selon la situation d'un foyer | Indicateur "potentiel d'aide" par zone. |
| **Géorisques** | Portail public des risques naturels/technologiques | Risque retrait-gonflement des argiles (et plus tard inondation, séisme…). |
| **Retrait-gonflement des argiles** | Sols argileux qui gonflent/se rétractent avec l'humidité → fissures des maisons. Aggravé par le réchauffement climatique. | Répond à la question du sujet sur les risques climatiques d'un investissement. |

### 2.3 Géographie administrative

| Terme | Définition |
|---|---|
| **Commune** | Plus petit échelon administratif (~35 000). Niveau "city" du sujet. |
| **EPCI** | Établissement Public de Coopération Intercommunale = regroupement de communes (intercommunalité). Plus proche d'un "bassin de vie" réel. |
| **Département** | 101 en France. Niveau intermédiaire. |
| **Région** | 18 en France. Vue la plus large. |
| **Code INSEE** (code commune) | Identifiant officiel unique d'une commune (ex : `38185`). ≠ code postal ! Un code postal peut couvrir plusieurs communes et inversement. |
| **DROM** | Départements et Régions d'Outre-Mer. |
| **Zone d'emploi** | Découpage INSEE basé sur les déplacements domicile-travail (utilisé pour le chômage). |

### 2.4 Géomatique & cartographie

| Terme | Définition |
|---|---|
| **Géocodage** | Adresse texte → coordonnées lat/lon. |
| **Reverse-géocodage** | Coordonnées → adresse. |
| **Score de matching** | Confiance (0→1) du géocodage BAN. Permet de filtrer les géocodages douteux. |
| **Lat / Lon** | Latitude / longitude. |
| **Polygone / géométrie** | Forme (contour) d'une zone stockée en base. |
| **GPKG (GeoPackage)** | Format fichier géo (SQLite) multi-couches. Format d'Admin Express. Ignoré par git (`*.gpkg`, trop lourd). |
| **Couche (layer)** | Dans un GPKG, un jeu de géométries d'un même type (COMMUNE, DEPARTEMENT…). |
| **Géométrie simplifiée** | Contours avec moins de points → fichier léger, affichage rapide, précision suffisante pour une carte. |
| **Topologie conservée** | Malgré la simplification, les frontières entre zones voisines restent collées (pas de trous/chevauchements). |
| **Choroplèthe** | Carte où chaque zone est colorée selon une valeur (ex : prix/m² par département). |
| **Carte de chaleur (heatmap)** | Densité de points représentée en dégradé. |
| **Carte à bulles** | Cercles proportionnels à une valeur, posés sur les zones. |
| **Isoligne** | Lignes reliant des points de même valeur (comme les courbes de niveau). |
| **Leaflet** | Librairie JS de cartes interactives (utilisable en Python via folium / streamlit-folium). |
| **Chartjunk** | Décoration inutile dans un graphique qui gêne la lecture. À éviter. |

### 2.5 Bases de données

| Terme | Définition |
|---|---|
| **BDD relationnelle (SQL)** | Données en tables (lignes/colonnes) liées par des clés. Schéma fixe. Ici : **PostgreSQL**. |
| **PostGIS** | Extension PostgreSQL qui ajoute les types et fonctions géographiques (`geometry`, calcul de distance, intersection…). |
| **BDD non-relationnelle (NoSQL)** | Données sans schéma fixe (documents JSON, etc.). Ici : **MongoDB** (option Elasticsearch pour la recherche texte). |
| **Document** (MongoDB) | Un enregistrement JSON (≈ une ligne, mais structure libre et imbriquée). |
| **Collection** (MongoDB) | Ensemble de documents (≈ une table). |
| **Schéma** | Description de la structure : tables, colonnes, types, relations. |
| **PK** (Primary Key, clé primaire) | Colonne qui identifie de façon unique chaque ligne. |
| **FK** (Foreign Key, clé étrangère) | Colonne qui pointe vers la PK d'une autre table → crée la relation. |
| **Clé pivot / clé de jointure** | Colonne commune qui permet de relier des sources différentes. Chez nous : **`code_insee`**. |
| **Jointure (JOIN)** | Combiner deux tables via une clé commune. |
| **Cardinalité** (1—1, 1—N, N—1) | Combien de lignes d'une table correspondent à une ligne de l'autre. Ex : 1 commune → N transactions. |
| **Nullable** | Colonne qui peut être vide (ex : `id_ban` si le géocodage a échoué). |
| **Index** | Structure qui accélère les recherches sur une colonne (comme l'index d'un livre). |
| **ERD / Diagramme entité-relation** | Schéma visuel des tables et de leurs liens (fait en **Mermaid** dans nos .md). |
| **Mermaid** | Langage texte qui génère des diagrammes, rendu directement par GitHub/PyCharm. |
| **Migration** | Modifier la structure d'une BDD existante (ajout colonne, table…). Coûteux en SQL, inutile en NoSQL. |
| **jsonb** | Type PostgreSQL pour stocker du JSON indexable. |
| **Staging** | Zone de stockage intermédiaire des données brutes, avant nettoyage. |
| **Append-only** | On ajoute, on ne modifie jamais (cas des avis). |
| **Agrégation** (`$match`, `$group`) | Calculs groupés dans MongoDB (ex : nb de mentions "rat" par région). |

### 2.6 Traitement, big data, IA

| Terme | Définition |
|---|---|
| **DataOps** | Pratiques pour industrialiser le cycle de la donnée : collecte → nettoyage → stockage → analyse → mise à dispo. |
| **Pipeline** | Enchaînement automatisé des étapes de traitement. |
| **ETL** | Extract (récupérer), Transform (nettoyer/standardiser), Load (charger en base). |
| **Standardisation** | Uniformiser formats, unités, codes géo entre sources hétérogènes. |
| **Traçabilité** | Pouvoir dire d'où vient chaque donnée et quand elle a été récupérée. |
| **Hadoop / Spark** | Frameworks de traitement distribué (obligatoire dans le sujet). Spark = calcul en mémoire, plus rapide. |
| **Cluster / nœud** | Ensemble de machines (nœuds) qui se partagent un calcul. |
| **Databricks** | Plateforme cloud clé en main pour Spark. |
| **NLP** | Natural Language Processing = traitement automatique du texte. |
| **Sentiment analysis** | Détermine si un texte est positif/négatif (score, label). |
| **Mots-clés / thèmes** | Termes et sujets extraits automatiquement d'un avis. |
| **Streamlit** | Framework Python pour faire une app web de data rapidement. |

### 2.7 Indicateurs métier

| Terme | Définition |
|---|---|
| **Valeur foncière** | Prix de vente déclaré dans DVF. |
| **Mutation** | Terme juridique pour un changement de propriétaire (= une vente dans DVF). |
| **Prix/m²** | `valeur_fonciere / surface_reelle_bati`. Indicateur n°1. |
| **Référence cadastrale** | Identifiant de la parcelle au cadastre. |
| **Revenu médian par UC** | Revenu disponible "au milieu" (50 % au-dessus, 50 % en dessous), divisé par les **unités de consommation** (1 pour le 1er adulte, 0,5 par autre personne ≥14 ans, 0,3 par enfant <14 ans) → comparable entre foyers de tailles différentes. |
| **Déciles (D1…D9)** | Seuils qui coupent la population en 10 parts égales. D1 = 10 % les plus modestes sont en dessous ; D9 = 10 % les plus aisés au-dessus. **D9/D1** ou **D9−D1** = mesure d'inégalité. |
| **Ménages fiscaux** | Foyers déclarant des revenus. Petit nombre → statistique peu fiable. |
| **Taux de pauvreté** | Part de la population sous le seuil (60 % du niveau de vie médian national). |
| **Prix demandé vs prix vendu** | Annonce Orpi vs transaction DVF. L'écart mesure la **tension du marché** / la marge de négociation. |

---

## 3. Ce qu'on a mis en place (et pourquoi)

### 3.1 Organisation du repo

```
doc/
├── TODO.md          → feuille de route du projet (5 axes du sujet + bonus)
├── RECAP.md         → ce fichier
├── spec/            → une fiche par source : quels champs récupérer, intérêt
│   ├── DVF_sources.md           (+ BAN, BDNB, ANAH, Mes Aides Réno, DGFiP)
│   ├── AdminExpress_sources.md
│   ├── Revenu_commune_sources.md
│   └── DPE_sources.md
└── schema/          → schéma BDD versionné
    ├── README.md    → règles de versioning + historique
    ├── v1/  (relationnel + NoSQL)
    ├── v2/
    ├── v3/
    └── v4/  (corrections)
homepedia-project.pdf → sujet officiel (v2.1)
```

**Pourquoi** : séparer le *quoi* (spec des sources) du *comment on stocke* (schéma) et du *planning* (TODO). Chaque doc reste court et lisible.

### 3.2 Collecte par paliers (étapes 1 → 3)

| Étape | Sources | Objectif |
|---|---|---|
| **1 — Socle** | DVF, Admin Express, Revenu Filosofi, avis textuels (scraping + NLP) | MVP : carte + prix + 1 indicateur socio-éco + volet texte/IA. |
| **2 — Enrichissement** | Population INSEE, DPE (API), risque argiles | Qualité du parc, pondération par population, risque climatique. |
| **3 — Complément** | Écoles, gares SNCF, chômage INSEE, BAN, annonces Orpi | Atteindre 12+ sources, attractivité (éducation, transport), marché actuel. |

**Pourquoi** : le sujet demande 12+ sources, mais tout brancher d'un coup = gros risque de ne rien avoir qui marche. Chaque palier donne une app fonctionnelle, on enrichit ensuite.

### 3.3 Choix clés et justifications

1. **`code_insee` comme clé pivot partout**
   Toutes les sources officielles le fournissent (ou on l'obtient via BAN). → Une seule clé pour tout croiser, en SQL comme en NoSQL. Plus fiable que le code postal ou le nom de commune (homonymes, orthographes).

2. **BAN comme clé de géocodage fine (`id_ban`)**
   Les adresses DVF sont brutes/imprécises. BAN les normalise et donne lat/lon + code INSEE. → Placer une transaction précisément sur la carte, la relier à un DPE, un bâtiment, une école proche. `id_ban` est **nullable** car le géocodage peut échouer : on garde la transaction quand même (rattachée par `code_insee`).

3. **Deux bases : PostgreSQL/PostGIS + MongoDB**
   - **SQL** pour les données tabulaires structurées (transactions, revenus, contours…) : jointures, intégrité, requêtes géo avec PostGIS.
   - **NoSQL** pour les avis : texte libre, structure variable selon la source, champs NLP qui évoluent sans migration, gros volume en écriture, requêtes = full-text et agrégations plutôt que jointures.
   - Le sujet exige explicitement les deux.

4. **Hiérarchie géo `region → departement → commune`, + `epci`**
   Permet l'agrégation multi-échelle demandée (ville / département / région). EPCI en plus car c'est l'échelle pertinente pour des questions type "où implanter mes franchises".

5. **API DPE plutôt que dataset statique**
   Requête par adresse (via BAN), données à jour (~35k nouveaux DPE/semaine). Plus précis et plus frais.

6. **`dpe_logement` distinct de `batiment.classe_dpe`**
   BDNB donne un résumé par bâtiment ; l'API DPE donne un diagnostic par logement. Les deux coexistent : granularités différentes.

7. **`risque_commune` générique (colonne `type_risque`)**
   Aujourd'hui juste les argiles, mais ajouter inondation/séisme = nouvelles lignes, pas nouvelle table. Évite une migration future.

8. **`annonce_orpi` séparée de `transaction_dvf`**
   Source non-officielle, volatile (annonce peut disparaître), prix *demandé* ≠ prix *vendu*. Champs `url_source` + `date_scraping` obligatoires → traçabilité/audit (exigé par le sujet).

9. **`chomage_departement` au niveau département**
   L'INSEE ne publie pas le chômage à la commune. On stocke au niveau disponible plutôt que d'inventer une précision fausse.

10. **Collection `documents_bruts` (staging, option)**
    Garder la réponse brute de chaque API avant transformation → rejouer un nettoyage, débugger, prouver la provenance.

11. **Nb de ménages fiscaux conservé**
    Sert de garde-fou statistique : ne pas sur-interpréter le revenu d'une commune de 50 foyers.

### 3.4 Versioning du schéma

- Un dossier `vN/` par version majeure. Nouvelle version quand une source change la structure (nouvelle table, nouvelle clé).
- Une version validée est **figée** : on ne la modifie plus, on crée la suivante.
- Chaque version indique en tête : sources couvertes + diff vs précédente.

**Pourquoi** : garder l'historique des décisions (utile pour le livrable "schéma + description"), voir l'évolution palier par palier, éviter de casser ce qui a été validé.

| Version | Ajout | Tables |
|---|---|---|
| v1 | Étape 1 (DVF + BAN/BDNB/ANAH/Mes Aides Réno, Admin Express, Revenu, avis) | 9 SQL + collection `avis` NoSQL |
| v2 | Étape 2 | + `dpe_logement`, `risque_commune` |
| v3 | Étape 3 | + `etablissement_scolaire`, `gare`, `chomage_departement`, `annonce_orpi` |
| v4 | Corrections | − `avis_texte` (SQL), + `sentiment_commune`, PK composites revenu/chômage → 17 tables SQL |

### 3.5 Corrections v4 (incohérences repérées en relisant v1-v3)

- **`avis_texte`** était décrite comme table SQL *et* comme collection NoSQL `avis` → risque de double stockage. **v4** : avis uniquement en MongoDB ; côté SQL, table agrégée **`sentiment_commune`** (nb avis, score moyen, % négatif, thèmes par commune et période), calculée par un job Spark. Permet de croiser sentiment × prix × revenu en SQL sans toucher au texte brut.
- **`revenu_commune`** : PK `code_insee` seule empêchait de stocker plusieurs années → **PK composite `(code_insee, annee)`**, relation 1—N.
- **`chomage_departement`** : même problème avec les trimestres → **PK composite `(code_departement, trimestre)`**, relation 1—N.
- **`aide_renovation`** : statut ambigu en v1 ("pas stocké tel quel", absente de la répartition) → confirmée en SQL comme cache des simulations API.
- **Décompte v1** : "9 tables SQL" = répartition v1 (sans `aide_renovation` ni `avis_texte`). Expliqué dans `schema/README.md`.

Vocabulaire lié : **PK composite** = clé primaire formée de plusieurs colonnes ; c'est la combinaison qui est unique (ex : une commune *et* une année).

---

## 4. Où on en est / suite

- ✅ Sources identifiées et spécifiées (étapes 1-3)
- ✅ Schéma relationnel v1 → v4 (v4 = corrections) + schéma NoSQL v1
- ⬜ Récupération effective des données (scripts de collecte)
- ⬜ Méthodologie de nettoyage/standardisation (livrable)
- ⬜ Création des BDD (PostgreSQL/PostGIS + MongoDB) + index
- ⬜ Cluster Spark / Databricks
- ⬜ NLP sur les avis + doc du modèle (livrable)
- ⬜ App Streamlit + cartes Leaflet
