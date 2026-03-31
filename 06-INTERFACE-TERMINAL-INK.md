# 06 - Interface terminal (Ink)

## Vue d'ensemble

Claude Code utilise un **moteur de rendu React personnalise pour le terminal** base sur le concept d'Ink, mais entierement reecrit (~50 fichiers, 250KB+ de code). Ce n'est PAS la librairie `ink-cli` standard : c'est une implementation complete qui gere le rendu ANSI, le layout Yoga (flexbox), les evenements clavier/souris, et un systeme de composants React complet.

## Architecture du moteur de rendu

```
ink/
├── ink.tsx                    # Moteur principal, reconciliation
├── root.ts                    # Root node du Virtual DOM
├── dom.ts                     # Gestion du Virtual DOM
├── reconciler.ts              # React Fiber reconciliation
├── renderer.ts                # Pipeline de rendu
├── render-node-to-output.ts   # Conversion nodes → ANSI
├── render-border.ts           # Rendu des bordures
├── render-to-screen.ts        # Ecriture sur stdout
├── output.ts                  # Formatage de la sortie terminal
├── log-update.ts              # Batching des mises a jour
├── frame.ts                   # Gestion des frames
│
├── layout/                    # Systeme de layout
│   ├── engine.ts              # Moteur de layout
│   ├── node.ts                # Noeuds de layout
│   ├── geometry.ts            # Calculs geometriques
│   └── yoga.ts                # Bindings Yoga (flexbox)
│
├── components/                # Composants de base
│   ├── Box.tsx                # Conteneur flexbox
│   ├── Text.tsx               # Texte avec styles
│   ├── ScrollBox.tsx          # Zone scrollable
│   ├── Button.tsx             # Bouton interactif
│   ├── Link.tsx               # Lien cliquable
│   ├── Spacer.tsx             # Espaceur flexible
│   ├── Newline.tsx            # Saut de ligne
│   ├── RawAnsi.tsx            # ANSI brut
│   ├── NoSelect.tsx           # Contenu non-selectionnable
│   ├── AlternateScreen.tsx    # Ecran alternatif terminal
│   └── ErrorOverview.tsx      # Affichage d'erreurs
│
├── hooks/                     # Hooks React specifiques terminal
│   ├── use-input.ts           # Gestion des entrees clavier
│   ├── use-stdin.ts           # Acces stdin brut
│   ├── use-app.ts             # Contexte applicatif
│   ├── use-terminal-viewport.ts # Dimensions du terminal
│   ├── use-animation-frame.ts # Animations frame-based
│   ├── use-selection.ts       # Etat de selection
│   ├── use-declared-cursor.ts # Position du curseur
│   ├── use-interval.ts        # Intervalles
│   ├── use-terminal-title.ts  # Titre du terminal
│   ├── use-terminal-focus.ts  # Focus du terminal
│   ├── use-search-highlight.ts # Surbrillance de recherche
│   └── use-tab-status.ts     # Statut d'onglet
│
├── events/                    # Systeme d'evenements
│   ├── emitter.ts             # Emetteur d'evenements
│   ├── dispatcher.ts          # Dispatch d'evenements
│   ├── event.ts               # Evenement de base
│   ├── input-event.ts         # Evenement d'entree
│   ├── keyboard-event.ts      # Evenement clavier
│   ├── click-event.ts         # Evenement clic
│   ├── focus-event.ts         # Evenement focus
│   └── terminal-event.ts     # Evenement terminal
│
├── termio/                    # I/O terminal bas niveau
│   ├── tokenize.ts            # Tokenisation ANSI
│   ├── parser.ts              # Parseur de sequences
│   ├── ansi.ts                # Codes ANSI
│   ├── sgr.ts                 # Select Graphic Rendition
│   ├── csi.ts                 # Control Sequence Introducer
│   ├── osc.ts                 # Operating System Command
│   ├── esc.ts                 # Sequences d'echappement
│   ├── dec.ts                 # DEC private sequences
│   └── types.ts               # Types ANSI
│
├── parse-keypress.ts          # Parsing des touches (~23KB)
├── colorize.ts                # Systeme de couleurs
├── styles.ts                  # Styles CSS-like
├── stringWidth.ts             # Calcul de largeur de chaines
├── wrapAnsi.ts                # Word-wrap ANSI-safe
├── bidi.ts                    # Support bidirectionnel
├── focus.ts                   # Gestion du focus
├── selection.ts               # Gestion de la selection
├── searchHighlight.ts         # Surbrillance de recherche
├── hit-test.ts                # Tests de clic (hit testing)
├── optimizer.ts               # Optimisation du rendu
├── tabstops.ts                # Tabulations
├── terminal.ts                # Abstraction terminal
├── terminal-querier.ts        # Interrogation du terminal
├── supports-hyperlinks.ts     # Detection support liens
└── constants.ts               # Constantes
```

## Pipeline de rendu

```
Arbre React (JSX)
    │
    ▼
React Reconciler (reconciler.ts)
    │ Diff l'arbre virtual et produit des mutations
    ▼
Virtual DOM (dom.ts)
    │ Arbre de noeuds avec styles et props
    ▼
Layout Engine (layout/)
    │ Calcule les positions via Yoga (flexbox)
    ▼
render-node-to-output.ts
    │ Convertit chaque noeud en cellules ANSI
    ▼
output.ts
    │ Assemble les cellules en lignes de texte
    ▼
render-to-screen.ts
    │ Ecrit les lignes sur stdout avec diff minimal
    ▼
Terminal (ecran physique)
```

### Yoga Layout Engine

Le layout utilise **Yoga** (le moteur flexbox de Facebook/Meta) pour calculer les positions :

```typescript
// Chaque <Box> cree un noeud Yoga
<Box flexDirection="column" padding={1} borderStyle="round">
  <Text bold>Title</Text>
  <Box flexDirection="row" gap={2}>
    <Text>Left</Text>
    <Spacer />
    <Text>Right</Text>
  </Box>
</Box>
```

Proprietes de layout supportees :
- `flexDirection`, `flexGrow`, `flexShrink`, `flexBasis`
- `alignItems`, `alignSelf`, `justifyContent`
- `padding`, `margin` (tous les cotes)
- `width`, `height`, `minWidth`, `maxWidth`
- `borderStyle` (single, double, round, bold, etc.)
- `gap` (espacement entre enfants)

## Composants de base

### `<Box>` - Conteneur flexbox

```tsx
<Box
  flexDirection="column"
  padding={1}
  borderStyle="round"
  borderColor="blue"
>
  {children}
</Box>
```

Equivalent terminal d'un `<div>` avec flexbox.

### `<Text>` - Texte style

```tsx
<Text bold color="green">Success!</Text>
<Text dimColor>Secondary text</Text>
<Text underline>Link text</Text>
<Text strikethrough>Deprecated</Text>
```

### `<ScrollBox>` - Zone scrollable

```tsx
<ScrollBox height={20}>
  {longContent}
</ScrollBox>
```

Gere le defilement vertical avec barre de progression.

### `<Button>` - Bouton interactif

```tsx
<Button onPress={() => handleClick()}>
  Click me
</Button>
```

## Composants applicatifs (`components/`)

Le dossier `components/` contient ~146 composants React specifiques a Claude Code :

### Composants principaux
- **MessageHistory** : Affichage de la conversation
- **InputArea** : Zone de saisie multiline
- **ToolProgress** : Barre de progression des outils
- **PermissionDialog** : Dialogue de permission
- **FilePreview** : Apercu de fichier
- **DiffView** : Affichage de diff
- **MarkdownRenderer** : Rendu Markdown dans le terminal
- **StatusBar** : Barre d'etat en bas
- **BreadcrumbNav** : Navigation contextuelle

### Design system (`components/design-system/`)
- **ThemeProvider** : Gestion des themes (dark/light/auto)
- **ThemedBox / ThemedText** : Composants theme-aware
- **Systeme de couleurs** : Palette coherente pour le terminal

## Systeme d'evenements

### Entrees clavier

```
Flux des evenements clavier :

stdin (raw mode)
    │
    ▼
parse-keypress.ts
    │ Decode les sequences ANSI en evenements structures
    │ (Ctrl+C, arrows, function keys, Unicode, etc.)
    ▼
events/dispatcher.ts
    │ Route l'evenement dans l'arbre React
    ▼
use-input() hook
    │ Composant consomme l'evenement
    ▼
Handler du composant
```

### Parsing des touches

Le parseur de keypress (`parse-keypress.ts`, ~23KB) gere :
- Caracteres simples et Unicode
- Sequences d'echappement ANSI (CSI, SS3)
- Combinaisons Ctrl/Alt/Shift
- Touches speciales (arrows, Home, End, Page Up/Down)
- Sequences de collage (bracketed paste)
- Evenements souris (si actives)

### Hooks d'entree

```typescript
// Utilisation dans un composant
function MyComponent() {
  useInput((input, key) => {
    if (key.return) {
      handleSubmit()
    } else if (key.escape) {
      handleCancel()
    } else if (key.ctrl && input === 'c') {
      handleExit()
    }
  })

  return <Text>My component</Text>
}
```

## Optimisations de rendu

### Diff minimal
Le renderer ne reecrit que les lignes qui ont change entre deux frames.

### Batching des mises a jour
Les mises a jour React sont batchees : plusieurs `setState` dans le meme tick produisent un seul re-rendu.

### Cache de largeur de lignes
Un cache (`line-width-cache.ts`) evite de recalculer la largeur visible des lignes ANSI.

### Cache des noeuds
Un cache (`node-cache.ts`) stocke les resultats de rendu des noeuds statiques.

## Pour recreer cette partie

### Option 1 : Utiliser Ink existant (recommande pour un MVP)

```bash
npm install ink react
```

```tsx
import { render, Box, Text } from 'ink'

function App() {
  return (
    <Box flexDirection="column">
      <Text bold>Claude Code Clone</Text>
      <Text>Ready for input...</Text>
    </Box>
  )
}

render(<App />)
```

### Option 2 : Implementation personnalisee (pour controle total)

1. **Reconciler React** : Utiliser `react-reconciler` pour creer un renderer custom
2. **Layout** : Integrer `yoga-layout` pour le calcul flexbox
3. **Rendu ANSI** : Implementer la conversion noeud → sequences ANSI
4. **Entrees** : Parser les evenements clavier depuis stdin en raw mode
5. **Diff** : Implementer un diff de frames pour minimiser les ecritures stdout

### Points cles

- Le terminal est un canvas 2D de cellules caractere
- Chaque cellule a un caractere, une couleur foreground et background
- Le layout flexbox calcule les positions x,y de chaque noeud
- Le rendu convertit les noeuds positionnes en sequences ANSI
- Le diff compare les frames pour ne mettre a jour que le necessaire
- stdin en raw mode permet de capter chaque touche individuellement
