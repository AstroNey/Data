# Revenu des Français à la commune — données précises

[data.gouv.fr/datasets/revenu-des-francais-a-la-commune](https://www.data.gouv.fr/datasets/revenu-des-francais-a-la-commune) · [reuses/dataservices](https://www.data.gouv.fr/datasets/revenu-des-francais-a-la-commune/reuses_and_dataservices)

Source: INSEE Filosofi (données fiscales et sociales). Pas d'API query dédiée — fichier CSV/xlsx téléchargeable + API générique data.gouv.fr (metadata/download).

## Champs à récupérer

Descriptif: fichier plat (1 ligne = 1 commune), indicateurs de revenu des ménages calculés par INSEE à partir des déclarations fiscales (Filosofi).

- **Code commune INSEE** — clé de jointure avec DVF/BAN. *Intérêt*: rattache la commune à toutes les autres sources.
- **Revenu médian disponible par unité de consommation (UC)** — niveau de vie local. *Intérêt*: corrèle avec prix DVF, base de l'indicateur richesse/pauvreté par zone.
- **Déciles de niveau de vie (D1 à D9)** — distribution des revenus, pas juste la moyenne. *Intérêt*: détecte les communes à forte inégalité interne (écart D9-D1).
- **Nb de ménages fiscaux** — taille de l'échantillon. *Intérêt*: fiabilité statistique, évite de sur-interpréter une petite commune.
- **Taux de pauvreté** (si présent) — part sous seuil de pauvreté. *Intérêt*: répond direct à "comment réduire les inégalités dans mon département" (question type du sujet).

## Réutilisations existantes (inspiration viz)
- [Analyse des revenus en France](https://www.data.gouv.fr/reuses/analyse-des-revenus-en-france)
- [Carte des revenus](https://www.data.gouv.fr/reuses/carte-des-revenus)
