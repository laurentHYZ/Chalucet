# Médiathèque Chalucet — Toulon

Page de présentation de la médiathèque Chalucet (Toulon) : un lieu culturel installé dans l'ancien hôpital Chalucet, entre patrimoine hospitalier et architecture contemporaine.

## Contenu

- `index.html` — page unique, autonome (HTML + CSS + JS, aucune dépendance hors Google Fonts)
- `img/` — photographies optimisées
- `dev-environment-guide.md` — guide de mise en place de l'environnement de développement

## Trois propositions de design

Un sélecteur en bas à gauche de la page permet de basculer entre trois directions artistiques (raccourcis clavier `1` / `2` / `3`) :

| # | Nom | Direction |
|---|-----|-----------|
| 01 | Éditorial | Ivoire, terracotta, serif éditorial — chaleureux et patrimonial |
| 02 | Méditerranée | Cobalt, sable, sans-serif affirmé — jeune et civique |
| 03 | Galerie | Blanc, noir, grotesque — muséal et minimal |

Chaque variante est aussi accessible par URL : `?design=a`, `?design=b`, `?design=c`.

## Voir la page

Ouvrir `index.html` dans un navigateur, ou servir le dossier :

```bash
python3 -m http.server 8000
```

Puis http://localhost:8000

## Crédits

Design & direction artistique : **Laurent Pinson**.
Photographies : © Lisa Ricciotti, © D. Huguenin / Vezzoni & Associés, Enssib, Infociments.
Page de présentation non officielle, réalisée à partir d'informations publiques.
