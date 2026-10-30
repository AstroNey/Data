# 08 — Application cible

Précédent : [07_NLP.md](07_NLP.md) · Vue d'ensemble : [Data_flow.md](Data_flow.md) · Données lues : [06_Serving.md](06_Serving.md)

---

## 1. Objectif

Application web interactive d'**analyse du marché immobilier français** (Streamlit + Folium/Leaflet).
Utilisateurs cibles : agences immobilières, collectivités, analystes — **pas** les acheteurs particuliers.

Principe : l'app ne lit que la couche Serving ([06_Serving.md](06_Serving.md)) et n'effectue aucun calcul lourd (tout est pré-agrégé en Gold).

## 2. Pages

| Page | Contenu | Source de données |
|---|---|---|
| **Carte** | Choroplèthe d'un indicateur au choix (prix/m², revenu, sentiment…) ; échelle commune / département / région ; points ou chaleur des transactions | PostGIS : `v_carte_prix`, `transaction_carte` |
| **Fiche zone** | Tous les indicateurs d'une zone, évolution dans le temps, comparaison à la zone parente | PostGIS : `gold.profil_zone` |
| **Comparateur** | 2 à 5 zones côte à côte, croisement de 2 indicateurs (nuage de points, ex. prix/m² vs revenu médian) | PostGIS : `gold.*` |
| **Tableaux** | Tables triables / filtrables, indicateurs clés, **export CSV** | PostGIS : `gold.*` |
| **Avis & sentiment** | Score et thèmes d'une zone, avis récents, recherche plein texte, filtre positif / négatif | MongoDB : `avis` ; PostGIS : `indicateurs_sentiment` |
| **Prix demandé vs vendu** (étape 3) | Écart entre annonces Orpi et transactions DVF par zone | MongoDB : `annonce` ; PostGIS : `indicateurs_prix` |
| **Méthodologie & fraîcheur** | Sources, millésimes, date de mise à jour, limites | PostGIS : `meta.fraicheur` |

## 3. Questions auxquelles l'app doit répondre

- Où le prix au m² a-t-il le plus progressé sur 5 ans ?
- Quel est le ratio prix / revenu par département (accessibilité, inégalités) ?
- Quelles zones cumulent prix élevé et sentiment négatif (nuisances) ?
- Quel est le parc énergétique (DPE) et le risque argiles d'une zone avant d'y investir ?
- Un bien est-il affiché au-dessus du prix de marché (annonce vs DVF) ?

## 4. Exigences d'interaction

- **Multi-échelle** : ville, département, région, avec navigation hiérarchique (clic sur une zone → zoom sur ses enfants).
- Choix libre de l'indicateur, de la période et du type de bien.
- Affichage des **seuils de significativité** : zone avec trop peu de ventes / d'avis → grisée, jamais interpolée.
- Cartes sobres, représentation adaptée à la donnée (choroplèthe pour un taux, bulles pour un volume), pas de chartjunk.
- Réponse < 500 ms sur les requêtes types ([06_Serving.md](06_Serving.md) §7).

## 5. Hors périmètre (v1)

Authentification, temps réel, déploiement en ligne, vue admin : bonus, traités après le MVP.

## 6. MVP (étape 1)

Carte choroplèthe prix/m² + fiche zone (prix, revenu, sentiment) + page avis. Les autres pages arrivent avec les sources des étapes 2-3.
