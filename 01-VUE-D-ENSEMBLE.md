# 01 - Vue d'ensemble

## Qu'est-ce que Claude Code ?

Claude Code est le CLI (Command Line Interface) officiel d'Anthropic. C'est un outil de developpement interactif qui permet aux developpeurs d'interagir avec Claude directement depuis leur terminal. Il fournit un environnement REPL complet avec execution d'outils, gestion de fichiers, acces au shell, et une interface terminal riche.

## Stack technique

| Composant | Technologie |
|-----------|-------------|
| **Langage** | TypeScript / TSX |
| **Runtime** | Bun (build, bundling, execution) |
| **Interface terminal** | React + Ink personnalise (rendu ANSI dans le terminal) |
| **Client API** | Anthropic SDK (`@anthropic-ai/sdk`) |
| **Gestion d'etat** | React Context API + pattern Store custom |
| **Protocole d'outils** | Model Context Protocol (MCP) |
| **Configuration** | JSON (user/project) + MDM (macOS) |
| **Shell** | Bash / PowerShell avec sandboxing optionnel |

## Architecture globale

```
┌──────────────────────────────────────────────────────────┐
│                   COUCHE PRESENTATION                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │   REPL   │  │  Bridge  │  │  Daemon  │               │
│  │ (Screen) │  │  (Web)   │  │ (Worker) │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
│       └──────────────┼─────────────┘                     │
│                      ▼                                    │
│  ┌──────────────────────────────────────┐                │
│  │        Ink Renderer (React)          │                │
│  │   Composants, Hooks UI, Layout      │                │
│  └──────────────────┬───────────────────┘                │
├─────────────────────┼────────────────────────────────────┤
│                COUCHE LOGIQUE METIER                      │
│                      ▼                                    │
│  ┌──────────────────────────────────────┐                │
│  │         QueryEngine                  │                │
│  │  Cycle de vie des requetes API       │                │
│  └──────────────────┬───────────────────┘                │
│         ┌───────────┼───────────┐                        │
│         ▼           ▼           ▼                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                │
│  │  Tools   │ │ Commands │ │  State   │                │
│  │ (41+)    │ │ (100+)   │ │ Manager  │                │
│  └──────────┘ └──────────┘ └──────────┘                │
├──────────────────────────────────────────────────────────┤
│                COUCHE SERVICES                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ API      │ │Analytics │ │ Auth     │ │ Config   │   │
│  │ Client   │ │Telemetry │ │ Keychain │ │ Settings │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
├──────────────────────────────────────────────────────────┤
│                COUCHE INFRASTRUCTURE                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ Memoire  │ │ Hooks    │ │ Perms    │ │ Feature  │   │
│  │ (Dream)  │ │ Systeme  │ │ Systeme  │ │ Flags    │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
└──────────────────────────────────────────────────────────┘
```

## Structure des fichiers

```
claude-code/
├── main.tsx                    # Point d'entree principal (~4700 lignes)
├── QueryEngine.ts              # Moteur de requetes API
├── query.ts                    # Preparation des messages
├── commands.ts                 # Registre des commandes
├── tools.ts                    # Registre des outils
├── Tool.ts                     # Interface de base Tool
├── Task.ts                     # Interface de base Task
├── context.ts                  # Gestion du contexte
├── setup.ts                    # Configuration initiale
├── ink.ts                      # Point d'entree Ink
├── history.ts                  # Historique des sessions
├── cost-tracker.ts             # Suivi des couts
├── costHook.ts                 # Affichage des couts
├── dialogLaunchers.tsx         # Lanceurs de dialogues
├── interactiveHelpers.tsx      # Helpers interactifs
├── replLauncher.tsx            # Lanceur du REPL
├── projectOnboardingState.ts   # Etat d'onboarding
├── tasks.ts                    # Gestion des taches
│
├── entrypoints/                # Points d'entree (CLI, daemon, MCP)
├── screens/                    # Ecrans (REPL, Resume, Settings)
├── components/                 # Composants React (~146 fichiers)
├── ink/                        # Moteur de rendu terminal (~50 fichiers)
├── tools/                      # Implementations d'outils (~45 dossiers)
├── commands/                   # Implementations de commandes (~103 fichiers)
├── services/                   # Services (API, analytics, auth)
├── hooks/                      # Hooks React (~87 fichiers)
├── utils/                      # Utilitaires (~331 fichiers)
├── state/                      # Gestion d'etat
├── types/                      # Definitions de types TypeScript
├── constants/                  # Constantes et prompts
├── context/                    # React Context providers
├── bridge/                     # Systeme de controle distant (~33 fichiers)
├── keybindings/                # Gestion des raccourcis clavier
├── memdir/                     # Systeme de memoire persistante
├── migrations/                 # Migrations de schema
├── skills/                     # Systeme de skills
├── tasks/                      # Gestion des taches en arriere-plan
├── query/                      # Modules de requete supplementaires
├── remote/                     # Execution distante
├── server/                     # Serveur HTTP interne
├── plugins/                    # Systeme de plugins
├── schemas/                    # Schemas de validation
├── vim/                        # Mode vim (motions, operators)
├── voice/                      # Mode voix (VOICE_MODE)
├── buddy/                      # Compagnon Tamagotchi (BUDDY)
├── coordinator/                # Orchestration multi-agent
├── assistant/                  # Mode assistant toujours actif (KAIROS)
├── native-ts/                  # Bindings natifs
├── upstreamproxy/              # Proxy HTTP
├── outputStyles/               # Styles de sortie
├── bootstrap/                  # Donnees de bootstrap
├── moreright/                  # Extensions de droits
└── public/                     # Ressources statiques
```

## Principes architecturaux cles

### 1. Feature Gates a la compilation (Bun)
Le projet utilise des feature flags resolus a la compilation via Bun. Cela permet d'eliminer le code mort (dead-code elimination) pour les builds externes vs internes.

```typescript
// Exemple : le module assistant n'est inclus que si KAIROS est actif
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js')
  : null
```

### 2. Chargement paresseux des modules
Les modules lourds sont charges a la demande pour minimiser le temps de demarrage.

```typescript
const messageSelector = () =>
  require('src/components/MessageSelector.js')
```

### 3. Prompts systeme modulaires
Le prompt systeme est compose de sections cachees (stables) et volatiles (dynamiques), optimisant l'utilisation du cache de prompt d'Anthropic.

### 4. Pattern Observer pour l'etat
Les changements d'etat declenchent des listeners qui provoquent des re-rendus et des effets secondaires.

### 5. Registre filtre pour les outils
Tous les outils sont enregistres, puis filtres par feature flags, type d'utilisateur et regles de permission.

## Metriques du projet

| Metrique | Valeur |
|----------|--------|
| Fichiers TypeScript/TSX | ~800+ |
| Outils disponibles | 41+ |
| Commandes | 100+ |
| Hooks React | 87+ |
| Utilitaires | 331+ |
| Composants UI | 146+ |
| Taille du fichier principal (main.tsx) | ~4700 lignes |
