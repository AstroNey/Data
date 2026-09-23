# API DPE Logements — données précises

[data.gouv.fr/dataservices/api-dpe-logements](https://www.data.gouv.fr/dataservices/api-dpe-logements)

Source: ADEME. API temps réel (pas fichier statique), requêtable par adresse, 9M+ diagnostics, ~35k nouveaux/semaine.

## Champs à récupérer
- Code INSEE commune / adresse (clé de jointure avec BAN/commune)
- Classe énergie (A-G)
- Classe GES (A-G, émissions gaz à effet de serre)
- Date diagnostic
- Surface logement, type logement

## Intérêt
- Alimente directement la table `dpe_logement` (schema v2)
- Query par adresse via BAN → plus précis que le CSV statique, données à jour en continu
- Remplace la source "DPE Logements existants (dataset statique)" notée en étape 2 du TODO
