# DVF — Sources & données à récupérer via API

DVF = colonne vertébrale (prix transactions). Sources ci-dessous enrichissent/join DVF.

## BAN — API Adresse
[data.gouv.fr/dataservices/api-adresse-base-adresse-nationale-ban](https://www.data.gouv.fr/dataservices/api-adresse-base-adresse-nationale-ban)

Référentiel officiel des adresses françaises (25M+), API de géocodage/reverse-géocodage en temps réel, gérée par IGN/ANCT/DINUM.
- Géocodage adresse DVF → lat/lon précis
- id BAN, code INSEE commune, score matching
- **Intérêt**: clé de jointure entre DVF (adresse brute, souvent imprécise) et toutes autres sources (contours, écoles, gares...). Sans elle, impossible de positionner précisément une transaction sur la carte.

## CSTB — BDNB (Base Nationale des Bâtiments)
[data.gouv.fr/organizations/centre-scientifique-et-technique-du-batiment](https://www.data.gouv.fr/organizations/centre-scientifique-et-technique-du-batiment/datasets)

Cartographie de 27M+ bâtiments résidentiels/tertiaires en France, caractéristiques physiques du bâti, maintenue par le CSTB.
- id bâtiment, année construction, surface, nb logements
- matériaux, classe DPE, géoloc bâtiment
- **Intérêt**: join sur adresse/parcelle DVF → enrichit chaque transaction avec vraies caractéristiques du bâti (pas juste surface déclarée DVF), utile pour expliquer variations de prix.

## ANAH — Agence Nationale de l'Habitat
[data.gouv.fr/organizations/agence-nationale-de-lhabitat](https://www.data.gouv.fr/organizations/agence-nationale-de-lhabitat/datasets)

Établissement public en charge des aides à la rénovation du parc privé, publie ses dispositifs et opérations par territoire.
- Liste opérations programmées (OPAH): commune couverte oui/non, périmètre, dates
- Programme Habiter Mieux: nb logements rénovés, montant aides versées, par commune/département
- **Intérêt**: indicateur "commune sous dispositif rénovation aidée" — corrèle avec évolution prix, utile pour repérer zones en transformation.

## API Mes Aides Réno
[data.gouv.fr/dataservices/api-mes-aides-reno](https://www.data.gouv.fr/dataservices/api-mes-aides-reno)

API conversationnelle (publicodes) qui calcule l'éligibilité et le montant des aides à la rénovation énergétique selon situation du foyer.
- Input: code commune + revenu + type logement
- Output: montant aides rénovation énergétique éligibles
- **Intérêt**: indicateur "potentiel aide rénovation" par zone — utile pour analystes évaluant coût réel de rénovation d'un bien selon sa localisation.

## Ministères économiques et financiers (DGFiP)
[data.gouv.fr/organizations/ministeres-economiques-et-financiers](https://www.data.gouv.fr/organizations/ministeres-economiques-et-financiers/datasets)

Organisation publiant DVF (Direction Générale des Finances Publiques) et autres datasets fiscaux.
- Éditeur officiel de DVF lui-même
- Vérifier autres datasets fiscaux du même org (ex: taxe foncière par commune) si pertinent
- **Intérêt**: confirme fiabilité/officialité de la source DVF; explorer si dataset fiscal complémentaire utile (ex: taxe foncière = charge récurrente à intégrer dans analyse investissement).

## Priorité de jointure
1. BAN — obligatoire, clé géo
2. CSTB BDNB — enrichit caractéristiques bâti
3. ANAH — contexte politique logement/rénovation
4. Mes Aides Réno — bonus indicateur
