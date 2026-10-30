# 02 — Landing : fichiers tels que reçus

Précédent : [01_Transverse.md](01_Transverse.md) · Vue d'ensemble : [Data_flow.md](Data_flow.md) · Suivant : [03_Bronze.md](03_Bronze.md)

---

## 1. Rôle

Point d'entrée unique de toute donnée externe. On **dépose** ce que la source renvoie, octet pour octet, sans rien interpréter.

- Preuve de provenance (on peut montrer exactement ce qu'on a reçu, et quand).
- Rejeu complet : si Bronze/Silver/Gold sont corrompus, on reconstruit tout depuis Landing sans re-télécharger.
- Découplage : un script de collecte ne connaît **que** Landing. Il ne parse pas, ne nettoie pas, n'écrit dans aucune base.

## 2. Ce qui entre / ce qui sort

| Entrée | Sortie |
|---|---|
| Fichier téléchargé (CSV, TXT, xlsx, GPKG, ZIP) | Fichier identique dans `landing/` |
| Réponse d'API (JSON, une page = un fichier) | Fichier `.json` + paramètres de la requête |
| Page scrapée (HTML) | Fichier `.html` + URL + date |
| — | `_manifest.json` décrivant le dépôt |

## 3. Arborescence

```
lake/landing/
└── <source>/
    └── <date_ingestion>/            # YYYY-MM-DD (date du run, pas date des données)
        ├── _manifest.json
        ├── <fichier_original>       # nom d'origine conservé
        └── ...
```

Exemples :
```
lake/landing/dvf/2026-10-19/valeursfoncieres-2025.txt.zip
lake/landing/admin_express/2026-10-19/ADMIN-EXPRESS-COG_2026.gpkg
lake/landing/filosofi/2026-10-19/base-cc-filosofi-2023.xlsx
lake/landing/dpe_api/2026-10-19/page_000001.json
lake/landing/avis/2026-10-19/google_38185_000001.html
```

- `<source>` : snake_case, sans accent, liste fermée (cf. [01_Transverse.md](01_Transverse.md) §2).
- Archives ZIP **gardées zippées** : la décompression est faite par Bronze (le fichier Landing reste l'original).

## 4. Manifest

Un `_manifest.json` par dossier de dépôt. Il est écrit **en dernier** : sa présence signifie "dépôt complet". Sans manifest → Bronze ignore le dossier.

```json
{
  "source": "dvf",
  "batch_id": "dvf_20261019T083000",
  "date_ingestion": "2026-10-19T08:30:00Z",
  "methode": "download",
  "fichiers": [
    {
      "nom": "valeursfoncieres-2025.txt.zip",
      "url": "https://www.data.gouv.fr/...",
      "taille_octets": 412345678,
      "sha256": "9f2c...",
      "http_status": 200
    }
  ],
  "millesime_donnees": "2025",
  "licence": "Licence Ouverte 2.0",
  "script": "collect/dvf.py",
  "version_script": "git:cc4ef93"
}
```

| Champ | Pourquoi |
|---|---|
| `batch_id` | Identifiant suivi jusqu'au Gold (lineage). Format `<source>_<YYYYMMDDTHHMMSS>`. |
| `sha256` | Détecte un fichier inchangé → pas de nouveau lot inutile. Détecte une corruption. |
| `millesime_donnees` | ≠ date d'ingestion : DVF téléchargé en 2026 contient des ventes 2021-2025. |
| `licence` | Exigence open data / réutilisation. |
| `script` + `version_script` | Reproductibilité : quel code a produit ce dépôt. |

Pour les API et le scraping, chaque entrée de `fichiers` contient en plus `params` (requête envoyée) ou `url_page`.

## 5. Méthodes de collecte

### 5.1 Datasets (DVF, Admin Express, Filosofi, INSEE, écoles, gares, Géorisques)

1. Résoudre l'URL de la ressource via l'API data.gouv.fr (`/api/1/datasets/<id>/`) plutôt qu'une URL en dur (les URLs de fichiers changent à chaque millésime).
2. Télécharger en streaming (fichiers lourds : DVF ~ 400 Mo/an zippé).
3. Calculer le `sha256`. Si identique au dernier dépôt de la même source → **ne rien déposer**, logger "inchangé".
4. Écrire les fichiers, puis le manifest.

### 5.2 API (DPE, BAN, Mes Aides Réno)

- Une page de résultat = un fichier JSON, nommé `page_<n>.json`.
- Paramètres de la requête stockés dans le manifest (filtre date, commune, pagination).
- **Incrémental** : DPE interrogé par `date_etablissement_dpe >= dernière date chargée` (lue dans le manifest précédent).
- Respect des quotas : pause entre appels, retry avec backoff exponentiel sur 429/5xx, arrêt après N échecs.
- BAN en masse : préférer l'endpoint **CSV batch** (`/search/csv/`) au géocodage ligne par ligne.

### 5.3 Scraping (avis, Orpi)

- Lire `robots.txt`, respecter ses interdictions et un délai entre requêtes (≥ 1 s).
- User-Agent explicite identifiant le projet (usage académique).
- Stocker le **HTML brut**, pas le texte extrait : si l'extraction change, on rejoue depuis Landing.
- Orpi : snapshot complet quotidien du sitemap → permet de détecter les annonces disparues (vendues/retirées).
- Documenter dans le manifest : URL, date, ToS consultées.

## 6. Règles

1. **Immuable** : on n'écrit jamais deux fois dans un même dossier, on ne modifie jamais un fichier.
2. **Pas de suppression**, sauf purge volontaire documentée (ex : espace disque) — et seulement après que Bronze a été vérifié.
3. **Pas de transformation** : ni décompression, ni conversion d'encodage, ni renommage.
4. Un échec en cours de dépôt → dossier supprimé (pas de manifest = dépôt invalide), on relance.
5. `lake/` est **hors git** (`.gitignore`), seul le code de collecte est versionné.

## 7. Contrôles de sortie

| Contrôle | Bloquant |
|---|---|
| Manifest présent et valide (JSON, champs obligatoires) | oui |
| Chaque fichier listé existe et son `sha256` correspond | oui |
| Taille > 0, `http_status` = 200 | oui |
| Taille cohérente avec le dépôt précédent (± 50 %) | non (alerte) |

## 8. Definition of done

- [ ] Un script de collecte par source dans `collect/`, qui n'écrit que dans `lake/landing/<source>/`.
- [ ] Manifest généré pour chaque dépôt.
- [ ] Relancer le script sur une source inchangée ne crée pas de nouveau dépôt.
- [ ] Échec réseau → pas de dossier partiel laissé.
