# 07 — Modèles NLP : traitement des avis

Précédent : [06_Transverse.md](06_Transverse.md) · Vue d'ensemble : [Data_flow.md](Data_flow.md) · Entrée : [03_Silver.md](03_Silver.md) (entité `avis`) · Sortie : [04_Gold.md](04_Gold.md) (`indicateurs_sentiment`)

---

## 1. Rôle

Les avis (Google Maps / TripAdvisor, scraping) sont la seule source qualitative : ils captent ce que les chiffres ne disent pas (qualité de vie, nuisances, sécurité, transports). Le NLP transforme du texte libre en champs exploitables (`nlp.*`) puis en indicateurs agrégés par zone.

Le sujet impose de **documenter en détail** tout modèle d'IA utilisé : ce fichier est la base du livrable "doc algorithmes IA".

## 2. Où ça tourne

| Étape | Couche | Job |
|---|---|---|
| Nettoyage texte, détection de langue | Silver | `jobs/silver/avis` |
| Inférence sentiment + thèmes | Silver (même job, étape finale) | `jobs/silver/avis_nlp` |
| Agrégation par zone | Gold | `jobs/gold/indicateurs_sentiment` |

Inférence distribuée avec Spark (`pandas_udf` par lot de textes, modèle chargé une fois par exécuteur). Le modèle utilisé est **toujours tracé** (`modele_nlp`, `version_modele`) : changer de modèle = rejouer la table, pas de mélange silencieux.

## 3. Pipeline

```
texte brut ─► nettoyage ─► détection langue ─► sentiment ─► thèmes ─► champs nlp.*
             (HTML, emojis,  (fr / autre)      (3 classes)  (liste fermée)
              espaces)
```

Les avis hors français sont gardés mais non scorés (`nlp.label = null`, `flag_qualite = 'langue_non_supportee'`).

## 4. Modèles

| Tâche | Modèle proposé | Type | Sortie |
|---|---|---|---|
| Détection de langue | `fastText lid.176` (ou `langdetect`) | classifieur léger | `langue`, confiance |
| Sentiment | `cmarkea/distilcamembert-base-sentiment` (fallback : `nlptown/bert-base-multilingual-uncased-sentiment`) | Transformer français fine-tuné, 1 à 5 étoiles | `label` (négatif / neutre / positif), `score` (probabilité) |
| Thèmes | zero-shot (`MoritzLaurer/mDeBERTa-v3-base-xnli-multilingual-nli-2mil7`) sur une **liste fermée** de thèmes | classification NLI multilingue | `themes[]` (0 à 3 thèmes, seuil de confiance) |
| Exploration (optionnel) | BERTopic + `paraphrase-multilingual-MiniLM-L12-v2` | embeddings + clustering | thèmes émergents, pour **faire évoluer** la liste fermée, pas pour la prod |

Liste fermée de thèmes (v1) : `transport`, `securite`, `bruit`, `proprete`, `commerces`, `ecoles`, `espaces_verts`, `voisinage`, `prix_cout_vie`, `autre`.

Les noms de modèles sont des **propositions** : à valider par un test comparatif (§6) avant figement.

## 5. Format de sortie (Silver `avis` → document MongoDB)

```json
"nlp": {
  "langue": "fr",
  "label": "negatif",
  "score": 0.91,
  "etoiles_predites": 2,
  "themes": ["bruit", "transport"],
  "modele": "distilcamembert-base-sentiment",
  "version_modele": "2024-01"
}
```

## 6. Évaluation

- **Jeu étalon** : ~200 avis annotés à la main (sentiment + thèmes), échantillon stratifié par commune.
- **Métriques** : F1 macro (sentiment), F1 par thème, matrice de confusion.
- **Comparaison** : au moins 2 modèles de sentiment sur le jeu étalon avant de choisir.
- **Biais connus à documenter** : sarcasme, avis courts, avis sur un commerce (et non sur la zone) → thème `autre` et filtre sur la source.
- Résultats consignés dans `doc/` (livrable IA).

## 7. Agrégation Gold (`indicateurs_sentiment`)

Par `(niveau_geo, code_geo, periode)` : `nb_avis`, `score_moyen`, `part_negatifs`, `part_positifs`, top thèmes avec leur part de mentions négatives.
**Seuil de significativité** : sous N avis (à fixer, ex. 20) → indicateur marqué `insuffisant`, non affiché en carte (cf. [04_Gold.md](04_Gold.md)).

## 8. Contraintes

- Aucun nom d'auteur en clair (hash) — cf. [06_Transverse.md](06_Transverse.md) §10.
- Modèles sous licence ouverte (vérifier chaque licence Hugging Face avant usage).
- CPU suffisant : volume d'avis modeste (milliers à dizaines de milliers), pas de GPU requis.

## 9. Definition of done

- [ ] Jeu étalon annoté.
- [ ] 2 modèles de sentiment comparés, choix justifié.
- [ ] Job Silver `avis_nlp` idempotent, `modele_nlp` / `version_modele` renseignés.
- [ ] Doc IA rédigée (fonctionnement, limites, métriques).
