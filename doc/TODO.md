# Homepedia — TODO

App: analyse marché immobilier français. Users = agences immo, collectivités, analystes (pas acheteurs). Focus DataOps.

## 1. Collecte de données

Approche par étapes — ajouter sources progressivement, pas tout d'un coup. Min 12 sources visé au total, mais on construit palier par palier.

### Étape 1 — Socle (indispensable pour app minimale)
- [ ] [DVF — Demandes de valeurs foncières](https://www.data.gouv.fr/datasets/demandes-de-valeurs-foncieres) — 20M+ transactions immobilières réelles depuis 2014 (prix, surface, type bien, date, adresse). Intérêt: c'est LA donnée centrale, base de tout indicateur prix/m² et évolution.
- [ ] [Admin Express COG simplifiée (IGN)](https://www.data.gouv.fr/datasets/admin-express-cog-simplifiee-metropole-drom-saint-martin-saint-barthelemy) — contours géographiques (polygones) communes/départements/régions/EPCI. Intérêt: sans ça pas de carte, pas de choroplèthe, pas d'agrégation par zone.
- [ ] [Revenu des français à la commune](https://www.data.gouv.fr/datasets/revenu-des-francais-a-la-commune) — revenu médian, déciles, nb ménages fiscaux par commune (source INSEE Filosofi). Intérêt: 1er indicateur socio-éco, corrèle avec prix, répond à "inégalités par département".
- [ ] Avis textuels (Google Maps/TripAdvisor, scraping) → traité via NLP (sentiment analysis) — source textuelle obligatoire. Intérêt: seule source qualitative, capte ce que les chiffres ne disent pas (qualité de vie, nuisances), exigence explicite du sujet.

→ avec ces 4 : carte + prix + 1 indicateur socio-éco + volet textuel/IA fonctionnels. MVP viable.

### Étape 2 — Enrichissement (une fois étape 1 marche)
- [ ] [Data INSEE sur les communes](https://www.data.gouv.fr/datasets/data-insee-sur-les-communes) — population, structure démographique par commune. Intérêt: pondère les indicateurs (prix par habitant, densité), utile pour comparer zones.
- [ ] [API DPE Logements](https://www.data.gouv.fr/dataservices/api-dpe-logements) — classe énergétique (A-G) par logement, requêtable par adresse en temps réel, source ADEME (9M+ diagnostics, +35k/semaine). Intérêt: qualité du parc immobilier par zone, lien avec prix et aides rénovation. Préférée au dataset statique — query par adresse via BAN, données à jour.
- [ ] [Carte des risques retrait-gonflement des argiles](https://www.data.gouv.fr/datasets/carte-des-risques-retrait-gonflement-des-argiles) — zonage risque (faible à fort) par commune, lié au changement climatique. Intérêt: répond directement à la question type du sujet "risques d'investir dans ce lot causés par le réchauffement climatique".

### Étape 3 — Complément (si temps restant, pousse vers 12+)
- [ ] [Adresse et géolocalisation des établissements d'enseignement](https://www.data.gouv.fr/datasets/adresse-et-geolocalisation-des-etablissements-denseignement-des-premier-et-second-degres) — écoles/collèges/lycées géolocalisés. Intérêt: indicateur infra/éducation, critère d'attractivité d'une zone pour familles.
- [ ] [Liste des gares (SNCF)](https://www.data.gouv.fr/datasets/liste-des-gares) — gares réseau ferré géolocalisées + fréquentation. Intérêt: indicateur accessibilité/transport, corrèle avec prix (effet proximité gare).
- [ ] [INSEE — Taux de chômage localisés](https://www.insee.fr/fr/metadonnees/source/indicateur/p1660/bases-donnees-ligne) — taux chômage par région/département/zone d'emploi. Intérêt: 2e indicateur économique, complète le revenu pour analyse socio-éco.
- [ ] [Base Adresse Nationale (BAN)](https://www.data.gouv.fr/datasets/base-adresse-nationale) — 25M+ adresses géolocalisées avec précision, référentiel officiel. Intérêt: clé de géocodage fin pour joindre DVF/bâtiments/avis entre eux quand le code commune seul ne suffit pas.
- [ ] [Orpi — sitemap biens à vendre](https://www.orpi.com/sitemap-biens-a-vendre-1.xml) — **scraping** (annonces immo actives, prix demandé). Intérêt: complète DVF (transactions passées/réelles) avec offre actuelle sur le marché — permet comparer prix demandé vs prix vendu. Attention ToS (usage académique uniquement, cf. avertissement sujet).

- [ ] Pour chaque source ajoutée: couvrir niveaux état/région/département/ville, documenter sourcing (scraping toléré usage académique seulement, attention ToS)

## 2. Organisation base de données
- [ ] Concevoir BDD relationnelle (données tabulaires) — schéma
- [ ] Concevoir BDD non-relationnelle (données non structurées/textuelles/NoSQL)
- [ ] Standardiser données hétérogènes (formats, unités, codes géo)
- [ ] Indexation/optimisation efficaces
- [ ] Rédiger méthodologie de nettoyage des données

## 3. Analyse big data
- [ ] Monter cluster (Hadoop / Spark obligatoire)
- [ ] Envisager Databricks (offre gratuite, AWS/Azure/GCP) — voir avec pédago pour tokens campus/VMs locales
- [ ] Distribuer traitement sur plusieurs nœuds
- [ ] Si IA utilisée (ex: sentiment analysis) — documenter en détail implémentation/modèle/fonctionnement

## 4. Visualisations interactives / App
- [ ] Construire app (Streamlit suggéré)
- [ ] Vues tabulaires: tables triables/filtrables, indicateurs clés (prix/m², évolution prix, démographie, infra), croisement d'indicateurs, export
- [ ] Vues cartographiques: cartes interactives (Leaflet), bon choix de représentation (éviter chartjunk) — cartes bulles/isoligne/choroplèthe/chaleur/route/connexion
- [ ] Support multi-échelle: ville, département, région
- [ ] Forte interactivité: user choisit zones/indicateurs/résultats d'analyse

## 5. Livrables
- [ ] Schéma BDD + description
- [ ] Doc méthodologie nettoyage données
- [ ] Doc algorithmes IA (si utilisés)
- [ ] App interactive fonctionnelle

## Bonus (optionnel)
- [ ] Marchés immo d'autres pays
- [ ] Déployer app en ligne
- [ ] Auth + comptes
- [ ] Renforcer sécurité app
- [ ] Vue admin serveur
- [ ] Mises à jour temps réel
- [ ] Visite guidée

## Refs
- Inspiration: Zillow, MeilleursAgents, mapbox.com (ne pas copier)
- Choix viz: anychart, datavizcatalogue, from data to viz, datavizproject
- Doc source: homepedia-project.pdf (v2.1)
