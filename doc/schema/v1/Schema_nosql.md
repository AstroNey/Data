# Homepedia — Schéma NoSQL v1 (MongoDB)

**Sources couvertes**: avis textuels (Google Maps/TripAdvisor).
**Vs version précédente**: n/a (première version).

Complète le schéma relationnel ([Schema_relationnel.md](Schema_relationnel.md)). Stocke données textuelles/non structurées, volumétrie variable, pas de schéma fixe utile.

## Collection `avis`

Un document = un avis brut + résultat NLP.

```json
{
  "_id": "ObjectId",
  "code_insee": "38185",
  "commune_nom": "Le Grand Lemps",
  "source": "google_maps",
  "etablissement": {
    "nom": "Le Bistrot du Coin",
    "categorie": "restaurant",
    "lat": 45.371,
    "lon": 5.421
  },
  "texte_brut": "Super repas mais j'ai vu un rat dans la salle...",
  "date_avis": "2025-03-12",
  "note": 3,
  "nlp": {
    "langue": "fr",
    "score_sentiment": -0.42,
    "label_sentiment": "negatif",
    "mots_cles": ["rat", "service", "prix"],
    "themes": ["hygiene", "service"]
  },
  "date_scraping": "2026-09-20T10:00:00Z"
}
```

- Index: `code_insee`, `source`, `nlp.label_sentiment`, `date_avis`
- Agrégation type: "nb mentions rats par région" → `$match` sur `mots_cles` + `$group` par code_insee/région

## Collection `documents_bruts` (option, staging avant nettoyage)

Stocke réponses API brutes avant transformation (traçabilité pipeline) — un doc par appel source.

```json
{
  "_id": "ObjectId",
  "source": "mes_aides_reno",
  "code_insee": "38185",
  "payload_brut": { "...": "réponse API telle quelle" },
  "date_recuperation": "2026-09-20T10:00:00Z",
  "statut_traitement": "a_traiter"
}
```

## Pourquoi NoSQL ici et pas relationnel
- Texte libre, structure variable selon source (Google vs TripAdvisor)
- Champs NLP évolutifs (ajout thèmes/modèles sans migration schéma)
- Volumétrie avis potentiellement énorme, écriture append-only
- Requêtes principales = full-text / agrégation, pas jointures

## Lien avec le relationnel
- `code_insee` = même clé pivot que côté SQL → permet de croiser avis (Mongo) avec prix DVF / revenu (Postgres) dans l'app
