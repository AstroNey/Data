# Schéma — versioning

Un dossier `vN/` par version majeure du schéma. Nouvelle version = quand on ajoute des sources qui changent la structure (nouvelle table/collection, nouvelle clé de jointure, etc).

## Historique

- **v1** — socle étape 1 (DVF, Admin Express, Revenu commune, avis textuels). 9 tables SQL + 1 collection NoSQL. Voir [v1/Schema_relationnel.md](v1/Schema_relationnel.md) et [v1/Schema_nosql.md](v1/Schema_nosql.md).
- **v2** — étape 2 (+ population INSEE, DPE logements, risques argiles). +2 tables SQL (`dpe_logement`, `risque_commune`). NoSQL inchangé, cf. v1/Schema_nosql.md. Voir [v2/Schema_relationnel.md](v2/Schema_relationnel.md).
- **v3** — étape 3 (+ écoles, gares SNCF, chômage INSEE, annonces Orpi scraping). +4 tables SQL (`etablissement_scolaire`, `gare`, `chomage_departement`, `annonce_orpi`). NoSQL inchangé. Voir [v3/Schema_relationnel.md](v3/Schema_relationnel.md).

## Règle
- Ne pas modifier une version passée une fois "figée" (après check/validation) — créer v2, v3...
- Copier le fichier précédent comme base, puis diffs seulement (tables ajoutées/modifiées)
- Chaque version doit lister en haut: sources couvertes + ce qui change vs version précédente
