# Admin Express COG simplifiée — données précises

[data.gouv.fr/datasets/admin-express-cog-simplifiee-metropole-drom-saint-martin-saint-barthelemy](https://www.data.gouv.fr/datasets/admin-express-cog-simplifiee-metropole-drom-saint-martin-saint-barthelemy)

Source: IGN. Format GPKG (~57 Mo, géométries simplifiées, topologie conservée). Métropole + DROM + Saint-Martin/Saint-Barthélémy.

## Couches à récupérer

Descriptif: GeoPackage multi-couches, une couche par niveau administratif, chaque ligne = un polygone (contour) + attributs.

**COMMUNE** — plus petit échelon administratif, 35000+ communes en France.
- Code INSEE (clé de jointure avec DVF, revenu, population)
- Nom commune
- Géométrie (polygone)
- Code département, code région
- Population
- *Intérêt*: échelon le plus fin pour la carte, niveau requis par le sujet ("city")

**DEPARTEMENT** — 101 départements.
- Code département
- Nom
- Géométrie (polygone)
- Code région
- *Intérêt*: agrégation intermédiaire, niveau requis par le sujet ("department"), utile pour vues régionales moins granulaires

**REGION** — 18 régions.
- Code région
- Nom
- Géométrie (polygone)
- *Intérêt*: vue la plus large, niveau requis par le sujet ("region"), pour comparaisons macro

**EPCI** (intercommunalités) — regroupements de communes.
- Code EPCI
- Nom
- Géométrie (polygone)
- Liste communes membres
- *Intérêt*: échelon pertinent pour questions type "où localiser mes prochaines franchises" (bassin de vie plus réaliste qu'une commune isolée)

## Rôle dans Homepedia
- Contours pour cartes choroplèthes (prix/m², revenu, DPE...) à chaque échelle (ville/dept/région)
- Code INSEE commune = clé de jointure avec DVF, Revenu Filosofi, population INSEE
- Indispensable — sans ça pas de carte
