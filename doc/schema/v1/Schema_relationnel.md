# Homepedia — Schéma relationnel v1

**Sources couvertes**: DVF, BAN, Admin Express, Revenu commune, CSTB BDNB, ANAH, Mes Aides Réno.
**Vs version précédente**: n/a (première version).
Clé pivot partout: `code_insee_commune`.

## Tables

### `region`
| champ | type | note |
|---|---|---|
| code_region (PK) | varchar | |
| nom_region | varchar | |
| geometrie | geometry(polygon) | Admin Express |

### `departement`
| champ | type | note |
|---|---|---|
| code_departement (PK) | varchar | |
| nom_departement | varchar | |
| code_region (FK) | varchar | → region |
| geometrie | geometry(polygon) | Admin Express |

### `epci`
| champ | type | note |
|---|---|---|
| code_epci (PK) | varchar | |
| nom_epci | varchar | |
| geometrie | geometry(polygon) | Admin Express |

### `commune`
| champ | type | note |
|---|---|---|
| code_insee (PK) | varchar | clé pivot globale |
| nom_commune | varchar | |
| code_departement (FK) | varchar | → departement |
| code_epci (FK) | varchar, nullable | → epci |
| population | int | Admin Express / INSEE |
| geometrie | geometry(polygon) | Admin Express |

### `adresse` (BAN)
| champ | type | note |
|---|---|---|
| id_ban (PK) | varchar | |
| adresse_normalisee | varchar | |
| lat, lon | float | |
| code_insee (FK) | varchar | → commune |
| score_matching | float | |

### `transaction_dvf`
| champ | type | note |
|---|---|---|
| id_transaction (PK) | varchar/int | |
| id_ban (FK) | varchar, nullable | → adresse (géocodage) |
| code_insee (FK) | varchar | → commune |
| date_mutation | date | |
| valeur_fonciere | numeric | prix |
| surface_reelle_bati | numeric | |
| nb_pieces | int | |
| type_local | varchar | maison/appartement/terrain |
| ref_cadastrale | varchar | |

### `batiment` (CSTB BDNB)
| champ | type | note |
|---|---|---|
| id_batiment (PK) | varchar | |
| id_ban (FK) | varchar, nullable | → adresse |
| annee_construction | int | |
| surface | numeric | |
| nb_logements | int | |
| materiaux | varchar | |
| classe_dpe | varchar | A-G |

### `revenu_commune` (Filosofi/INSEE)
| champ | type | note |
|---|---|---|
| code_insee (PK, FK) | varchar | → commune |
| annee | int | |
| revenu_median_uc | numeric | |
| decile_1..decile_9 | numeric | |
| nb_menages_fiscaux | int | |
| taux_pauvrete | numeric, nullable | |

### `dispositif_anah`
| champ | type | note |
|---|---|---|
| id_operation (PK) | varchar | |
| code_insee (FK) | varchar | → commune |
| type_operation | varchar | OPAH / PIG / Habiter Mieux |
| date_debut, date_fin | date | |
| nb_logements_renoves | int, nullable | |
| montant_aides | numeric, nullable | |

### `aide_renovation` (Mes Aides Réno — appel API à la demande, pas stocké tel quel)
| champ | type | note |
|---|---|---|
| id_simulation (PK) | varchar | |
| code_insee (FK) | varchar | → commune |
| revenu_foyer | numeric | input |
| type_logement | varchar | input |
| montant_aide_estime | numeric | output API |

### `avis_texte` (source textuelle + NLP — non-relationnel de préférence, cf. section NoSQL)
| champ | type | note |
|---|---|---|
| id_avis (PK) | varchar | |
| code_insee (FK) | varchar | → commune |
| source | varchar | Google/TripAdvisor |
| texte_brut | text | |
| date_avis | date | |
| score_sentiment | float | calculé NLP |
| mots_cles | jsonb/array | extraits NLP |

## Relations (clé de jointure = code_insee)

```
region 1—N departement 1—N commune N—1 epci
commune 1—N adresse (BAN)
commune 1—N transaction_dvf
adresse 1—N transaction_dvf (via id_ban, optionnel)
adresse 1—N batiment (via id_ban, optionnel)
commune 1—1 revenu_commune (par année)
commune 1—N dispositif_anah
commune 1—N avis_texte
```

## Diagramme

```mermaid
erDiagram
    region ||--o{ departement : contient
    departement ||--o{ commune : contient
    epci ||--o{ commune : regroupe
    commune ||--o{ adresse : contient
    commune ||--o{ transaction_dvf : localise
    commune ||--|| revenu_commune : caracterise
    commune ||--o{ dispositif_anah : couvre
    commune ||--o{ avis_texte : concerne
    adresse ||--o{ transaction_dvf : geocode
    adresse ||--o{ batiment : geocode

    region {
        varchar code_region PK
        varchar nom_region
        geometry geometrie
    }
    departement {
        varchar code_departement PK
        varchar nom_departement
        varchar code_region FK
        geometry geometrie
    }
    epci {
        varchar code_epci PK
        varchar nom_epci
        geometry geometrie
    }
    commune {
        varchar code_insee PK
        varchar nom_commune
        varchar code_departement FK
        varchar code_epci FK
        int population
        geometry geometrie
    }
    adresse {
        varchar id_ban PK
        varchar adresse_normalisee
        float lat
        float lon
        varchar code_insee FK
    }
    transaction_dvf {
        varchar id_transaction PK
        varchar id_ban FK
        varchar code_insee FK
        date date_mutation
        numeric valeur_fonciere
        numeric surface_reelle_bati
        int nb_pieces
        varchar type_local
    }
    batiment {
        varchar id_batiment PK
        varchar id_ban FK
        int annee_construction
        numeric surface
        int nb_logements
        varchar classe_dpe
    }
    revenu_commune {
        varchar code_insee PK
        int annee
        numeric revenu_median_uc
        numeric decile_5
        int nb_menages_fiscaux
    }
    dispositif_anah {
        varchar id_operation PK
        varchar code_insee FK
        varchar type_operation
        date date_debut
        date date_fin
    }
    avis_texte {
        varchar id_avis PK
        varchar code_insee FK
        varchar source
        text texte_brut
        float score_sentiment
    }
```

## Répartition SQL / NoSQL
- **Relationnel (PostgreSQL/PostGIS)**: region, departement, epci, commune, adresse, transaction_dvf, batiment, revenu_commune, dispositif_anah — données structurées + géométries
- **Non-relationnel (MongoDB/Elasticsearch)**: avis_texte — texte libre + résultats NLP, volumétrie variable, requêtes full-text
