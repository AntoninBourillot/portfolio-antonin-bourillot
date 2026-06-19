# CLAUDE.md — Portfolio Antonin Bourillot

> Ce fichier est lu automatiquement par Claude Code au démarrage de chaque session
> dans ce dossier. Il sert de mémoire persistante du projet : ne pas le supprimer,
> le mettre à jour à chaque évolution importante (nouveau projet, nouvelle
> convention, assets ajoutés/manquants).

## Vue d'ensemble

Site portfolio personnel d'un architecte d'intérieur (Antonin Bourillot), orienté
collectivités, réemploi des matériaux et développement durable.

- **Single-page application** en HTML/CSS/JS vanilla, **un seul fichier `index.html`**
  (~1850 lignes). Pas de framework, pas de build step, pas de dépendances npm.
- Polices via Google Fonts CDN : Playfair Display (titres), DM Mono (labels/UI),
  Lora (texte courant).
- Palette CSS dans `:root` : `--ink`, `--paper`, `--moss` (vert sapin, couleur de
  marque), `--oak`, `--stone`.
- **3 langues** : FR (défaut) / EN / ES.
  - Textes d'interface globaux → objet `I18N` + attributs `data-i18n` /
    `data-i18n-html` + fonction `t(key)`.
  - Textes propres à un projet (titre, catégorie, description, prix) → champ
    `i18n:{en:{...}, es:{...}}` sur l'objet projet + fonction `pf(proj, field)`
    (repli automatique sur le FR si la traduction est absente).

## Structure du fichier `index.html`

- `<style>` : tout le CSS (variables, layout responsive à 2 breakpoints : 900px
  tablette, 520px mobile).
- `<body>` : 7 "vues" virtuelles, des `<div class="view" id="view-xxx">`, togglées
  en JS via la classe `.active`. **Pas de vraies pages séparées** — tout est dans
  ce fichier, le routing est géré par un hash d'URL.
  - `view-home` : hero + grille des projets sélectionnés
  - `view-tag` : grille de projets filtrée par tag
  - `view-project` : détail d'un projet (carrousel hero, description, tags,
    "carte du projet" en mind-map interactive, suggestions de projets similaires
    par score de tags communs)
  - `view-map` : carte interactive drag/zoom de **tous** les projets (positions
    `mapX`/`mapY`)
  - `view-reflex` : "Réflexions & recherches" — **actuellement 3 cartes
    hardcodées en HTML**, pas pilotées par un tableau de données (contrairement
    aux projets)
  - `view-about` : démarche, compétences, galerie de croquis (carrousel centré)
  - `view-contact` : email / téléphone / LinkedIn
- `<script>` : données (`PROJECTS`, `TAG_META`, `TAG_COLORS`, `I18N`, `SKETCHES`)
  + toute la logique (rendu, routing, drag, zoom, i18n).

## Routing

- `decodeHash(hash)` → `{view, param}` (ex: `#project/p11` → `{view:'project',
  param:'p11'}`)
- `navigate(view, param)` → met à jour l'URL (`history.pushState`) puis appelle
  `renderView(view, param)`.
- Tout changement de vue passe par ces deux fonctions — ne pas créer de système
  de routing parallèle.

## Modèle de données — `PROJECTS`

Chaque projet est un objet du tableau `PROJECTS` avec les champs :

| Champ | Rôle |
|---|---|
| `id` | identifiant unique (ex: `'p11'`) |
| `title`, `cat`, `desc` | titre / catégorie affichée / description (FR) |
| `tags` | array d'ids référencés dans `TAG_META`/`TAG_COLORS` |
| `thumb` | image de vignette (grille d'accueil) ; `null` → SVG placeholder généré (couleurs `color`/`stroke`) |
| `images` | array de chemins pour le carrousel hero de la page projet ; peut être vide |
| `mapX`, `mapY` | position sur la vue "Carte des projets" globale |
| `i18n` | `{en:{title,cat,desc,...}, es:{...}}` — traductions, repli FR si absent |
| `prize` *(optionnel)* | mention de prix/concours, affichée dans les méta |
| `video` *(optionnel)* | lien YouTube → sticker rond cliquable sur le hero |
| `linkedin` *(optionnel)* | lien post LinkedIn → sticker rond cliquable |
| `board` *(optionnel)* | fonction `buildXxxBoard(proj)` = mind-map personnalisée. Sinon `getBoardData()` génère une disposition automatique simple (hub + images + tags). |

### Les 9 projets actuels

| id | Titre | Catégorie | Tags | Board perso ? |
|---|---|---|---|---|
| p1 | Réhabilitation maison de quartier | Archi intérieur · École | developpement-durable, collectivites | non |
| p2 | Centre intergénérationnel — **YAPADAGE** | Archi intérieur · École | developpement-durable, collectivites, ux | **oui** (`buildYapadageBoard`) |
| p5 | Conception menuiserie - Escalier | Menuiserie · Personnel | menuiserie, reemploi | non |
| p6 | Carnets espaces collectifs | Croquis | croquis, collectivites | non |
| p7 | Épicerie de quartier — Café ISHI | Archi intérieur · Projet | erp, petit-espace, inclusion | non |
| p8 | AMO — CHUZELLES | Archi · Collectivité | collectivites, moe | non |
| p9 | Permis de construire — PUIVERT | Archi · Conseil | developpement-durable | non |
| p10 | Espace de jeu — Le Pré Des Sourires | Archi intérieur · Projet, groupe | ux, inclusion | non |
| p11 | Réhabilitation — **SEA SHEARD** | Archi intérieur · Projet · 2024 | developpement-durable, collectivites, ux, realise-en-groupe, reemploi | **oui** (`buildSeaSheardBoard`) — projet primé, a vidéo + post LinkedIn |

### Système de tags

`TAG_META` (label fr/en/es + classe CSS `.t-xx`) et `TAG_COLORS` (couleur hex,
réutilisée pour les traits de liaison dans les boards) — 11 tags actuellement :
`developpement-durable`, `collectivites`, `menuiserie`, `croquis`, `erp`,
`petit-espace`, `inclusion`, `ux`, `moe`, `realise-en-groupe`, `reemploi`.

## Système "Carte du projet" (board / mind-map)

Chaque page projet a une mind-map interactive : drag des nœuds, pan/zoom du
canevas, liens colorés avec élasticité (les nœuds liés suivent légèrement quand
on en déplace un), anti-collision automatique entre blocs.

- **Disposition radiale** : un hub central `{CX,CY}`, des branches de 1er niveau
  réparties en cercle à angle régulier via `pbPolar(cx,cy,angleDeg,radius)`
  (rayon `R1`), puis les enfants de chaque branche continuent sur le **même axe
  angulaire** mais plus loin (rayon `R2`, ex. `R1≈300`, `R2≈590-600`) — ça évite
  que les liens de branches différentes se croisent visuellement.
- `pbTopLeft(centre,w,h)` convertit un point central en coin haut-gauche pour le
  positionnement CSS `left/top`.
- **Types de nœuds** :
  - `text` : bloc texte, `label` optionnel + `text` optionnel
  - `image` : carré 190px, `src` + `caption` optionnel ; si `src` est absent/null
    → cadre vide avec icône (même principe que la galerie de croquis)
  - `tag` : pastille colorée, juste un `label` + `color`
- **Liens** : `{from, to, color}`. Option `dashed:true, noPull:true` pour les
  liaisons "faibles" (pointillées, sans force d'attraction élastique) — utilisé
  dans Sea Sheard pour relier des éléments transversaux entre branches.
- Les libellés (`L.xxx`) sont toujours définis dans un objet `*_BOARD_I18N`
  séparé (fr/en/es) — **ne jamais coder un label en dur dans la fonction
  `buildXxxBoard`**.

## Historique récent (à jour au 19/06/2026)

1. **Sea Sheard → Yapadage : 4 inspirations copiées** (branche `insp` du board
   Yapadage, autour de `ANG_INSP=225°`, `R2=590`) :
   - "Collectif Fos" (Madrid), "Playground" (Istanbul), "Networking Space"
     (Kiev), "Court métrage Gerry's Game"
   - Réutilisent les **mêmes fichiers image** que dans Sea Sheard
     (`images/sea-sheard-*`), pas de duplication de fichier. Toujours présentes
     aussi dans le board de Sea Sheard (copie, pas déplacement).
2. **Yapadage : 3 nouvelles images sous la branche "Les grands principes"**
   (`ANG_PRINC=315°`, `R2=590`, couleur du lien = `TAG_COLORS['collectivites']`) :
   - "Mouvement" → `images/yapadage-mouvement.jpg`
   - "Répétition" → `images/yapadage-repetition.jpg`
   - "Vide et plein" → `images/yapadage-vide-et-plein.jpg`
   - Ces 3 fichiers ont été fournis par l'utilisateur et sont **présents** dans
     ce dossier (`images/`).

## ⚠️ Assets manquants (référencés dans le code, fichier absent du dossier)

À demander à l'utilisateur ou à remplacer avant mise en prod :

- **p1** : `images/placeholder-green.svg` (thumb seulement, pas d'image dans le
  carrousel du projet)
- **p7 (Café ISHI)** : `images/ishi-apres.webp`, `images/ishi-interieur.webp`
- **p10 (Pré des Sourires)** : `images/pre-des-sourires.jpg`
- **p11 (Sea Sheard)** : `sea-sheard-bat-entier.png`, `sea-sheard-visuel-arriere.png`,
  `sea-sheard-doshi-retreat.jpg`, `sea-sheard-collectif-fos-madrid.jpg`,
  `sea-sheard-playground-istanbul.jpeg`, `sea-sheard-domus-knokke.avif`,
  `sea-sheard-networking-kiev.jpeg`, `sea-sheard-gerrys-game.png`,
  `sea-sheard-balles-tennis.png`, `sea-sheard-ostrea.jpg`,
  `sea-sheard-acier-corten.jpg` (tous dans `images/`)
- **p2, p5, p6, p8, p9** : `images:[]` → aucun visuel de carrousel pour
  l'instant ; vignette de grille = SVG placeholder généré (pas un vrai défaut
  bloquant, mais à enrichir si des photos existent)
- **Galerie de croquis** (page "À propos", tableau `SKETCHES`) : 6 entrées,
  toutes `src:null` → cadres vides en attente

✅ **Présents** dans `images/` à ce stade : uniquement les 3 fichiers Yapadage
listés ci-dessus (Mouvement, Répétition, Vide et plein).

## Conventions à respecter en continuant

- **Nommage des images** : `images/<slug-projet>-<description>.<ext>`
  (ex: `yapadage-mouvement.jpg`, `sea-sheard-ostrea.jpg`).
- Toute nouvelle légende d'image dans un board → l'ajouter dans les **3 langues**
  de l'objet `*_BOARD_I18N` correspondant (pas seulement en FR).
- Tout nouveau tag → l'ajouter dans `TAG_META` (label 3 langues + nouvelle classe
  CSS `.t-xx` à créer dans `<style>`) **et** dans `TAG_COLORS`.
- Toute nouvelle clé de texte UI global → l'ajouter dans les **3 blocs** de
  `I18N` (fr/en/es) en même temps, jamais juste en FR.
- Ne pas dupliquer de markup HTML pour les projets : tout est généré depuis les
  données par les fonctions de rendu (`renderGrid`, `renderProjectView`,
  `renderProjBoard`, etc.) — ajouter un projet = ajouter une entrée dans
  `PROJECTS`, pas écrire du HTML à la main.

## Infos de contact (footer / page Contact)

- Email : antonin.bourillot@gmail.com
- Téléphone : +33 6 22 50 67 48
- LinkedIn : https://www.linkedin.com/in/antonin-bourillot/

## Pistes pour la suite (non commencées)

- Fournir les images manquantes listées ci-dessus (notamment Sea Sheard, très
  incomplet en assets réels malgré son board déjà riche).
- Passer la section "Réflexions" sur un vrai tableau de données comme
  `PROJECTS`, au lieu des 3 cartes actuellement en dur dans le HTML.
- Alimenter la galerie de croquis (`SKETCHES`) avec de vrais dessins.
- Décider si Yapadage doit avoir un `thumb` et des `images` de carrousel propres
  (actuellement `null` / `[]`), indépendamment de son board déjà personnalisé.
