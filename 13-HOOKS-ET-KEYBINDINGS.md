# 13 - Hooks et keybindings

## Vue d'ensemble

Le projet utilise deux types de hooks distincts :
1. **Hooks React** (~87 fichiers) : Logique reactive pour l'interface terminal
2. **Hooks d'execution** : Callbacks du cycle de vie de l'IA (pre/post sampling)

## Hooks React (`hooks/`)

### Categories

#### Permission et securite

| Hook | Description |
|------|-------------|
| `useCanUseTool` | Moteur de decision pour les permissions d'outils |
| `useApiKeyVerification` | Validation des credentials API |

#### Entree et interaction

| Hook | Description |
|------|-------------|
| `useArrowKeyHistory` | Navigation dans l'historique avec les fleches |
| `useCommandKeybindings` | Gestion des raccourcis de commande |
| `useGlobalKeybindings` | Raccourcis globaux (Ctrl+C, etc.) |
| `useExitOnCtrlCD` | Fermeture propre sur Ctrl+C/D |
| `useDoublePress` | Detection de double-appui |

#### Navigation et UI

| Hook | Description |
|------|-------------|
| `useInboxPoller` | Poll des messages d'agents pairs |
| `useBackgroundTaskNavigation` | Navigation dans les taches de fond |
| `useIDEIntegration` | Connexion avec l'IDE |
| `useIdeSelection` | Synchronisation de la selection IDE |
| `useHistorySearch` | Recherche dans l'historique des sessions |

#### Notifications et communication

| Hook | Description |
|------|-------------|
| `useChromeExtensionNotification` | Notifications via extension Chrome |
| `useLogMessages` | Journalisation des messages |
| `useLspPluginRecommendation` | Suggestions de plugins LSP |
| `useClaudeCodeHintRecommendation` | Suggestions intelligentes |

#### Donnees et configuration

| Hook | Description |
|------|-------------|
| `useSettingsChange` | Watcher des fichiers de settings |
| `useDynamicConfig` | Changements de config a runtime |
| `useManagePlugins` | Installation/suppression de plugins |
| `useDiffInIDE` | Integration diff dans l'IDE |

#### Avances

| Hook | Description |
|------|-------------|
| `useAfterFirstRender` | Initialisation differee |
| `useMergedClients` | Fusion de clients multiples |
| `useMailboxBridge` | Routage de messages |
| `useMainLoopModel` | Suivi du modele actif |

### Exemple : `useArrowKeyHistory`

```tsx
function useArrowKeyHistory() {
  const [historyIndex, setHistoryIndex] = useState(-1)
  const history = useHistory()

  useInput((input, key) => {
    if (key.upArrow) {
      const newIndex = Math.min(historyIndex + 1, history.length - 1)
      setHistoryIndex(newIndex)
      setInputValue(history[newIndex])
    }
    if (key.downArrow) {
      const newIndex = Math.max(historyIndex - 1, -1)
      setHistoryIndex(newIndex)
      setInputValue(newIndex === -1 ? '' : history[newIndex])
    }
  })

  return { historyIndex }
}
```

### Exemple : `useGlobalKeybindings`

```tsx
function useGlobalKeybindings() {
  useInput((input, key) => {
    // Ctrl+L : Effacer l'ecran
    if (key.ctrl && input === 'l') {
      clearScreen()
    }

    // Ctrl+R : Recherche dans l'historique
    if (key.ctrl && input === 'r') {
      toggleHistorySearch()
    }

    // Escape : Annuler l'action en cours
    if (key.escape) {
      cancelCurrentAction()
    }
  })
}
```

## Hooks d'execution (lifecycle hooks)

Ces hooks s'executent a differentes etapes du cycle de vie de l'IA :

### Types de hooks

```
Pre-sampling (avant l'appel API)
    ├── Preprocessing de l'input
    ├── Augmentation du contexte
    ├── Verification des permissions
    └── Injection de prompt systeme

Post-sampling (apres la reponse API)
    ├── Filtrage de la sortie
    ├── Redaction de contenu
    ├── Enforcement de structured output
    └── Traitement du extended thinking

Stop/Failure (en cas d'echec)
    ├── Strategies de fallback
    ├── Decisions de retry
    ├── Recovery d'erreur
    └── Notifications utilisateur
```

### Configuration des hooks

Les hooks sont configures dans `settings.json` :

```json
{
  "hooks": {
    "pre-tool-use": [
      {
        "matcher": "Bash",
        "command": "echo 'About to run bash command'"
      }
    ],
    "post-tool-use": [
      {
        "matcher": "*",
        "command": "log-tool-usage.sh"
      }
    ]
  }
}
```

### Execution des hooks

```typescript
async function runHooks(
  phase: 'pre-tool-use' | 'post-tool-use',
  context: HookContext
): Promise<HookResult> {
  const hooks = getHooksForPhase(phase)
  const matchingHooks = hooks.filter(h =>
    matchesPattern(h.matcher, context.toolName)
  )

  for (const hook of matchingHooks) {
    const result = await executeShellCommand(hook.command, {
      env: {
        TOOL_NAME: context.toolName,
        TOOL_INPUT: JSON.stringify(context.input),
        SESSION_ID: context.sessionId
      }
    })

    if (result.exitCode !== 0) {
      return { blocked: true, message: result.stderr }
    }
  }

  return { blocked: false }
}
```

## Systeme de keybindings (`keybindings/`)

### Architecture

```
keybindings/
├── keybindings.ts       # Registre et resolution
├── defaults.ts          # Raccourcis par defaut
├── parser.ts            # Parsing des sequences
└── ...

~/.claude/keybindings.json   # Configuration utilisateur
```

### Raccourcis par defaut

| Raccourci | Action |
|-----------|--------|
| `Enter` | Soumettre le message |
| `Ctrl+C` | Annuler / Quitter |
| `Ctrl+D` | Quitter |
| `Ctrl+L` | Effacer l'ecran |
| `Ctrl+R` | Rechercher dans l'historique |
| `Up/Down` | Naviguer dans l'historique |
| `Escape` | Annuler l'action en cours |
| `Tab` | Auto-completion |
| `Ctrl+Z` | Annuler la derniere action |

### Configuration personnalisee

```json
// ~/.claude/keybindings.json
{
  "bindings": [
    {
      "keys": ["ctrl+s"],
      "command": "submit"
    },
    {
      "keys": ["ctrl+k", "ctrl+c"],
      "command": "commit",
      "type": "chord"
    },
    {
      "keys": ["ctrl+shift+p"],
      "command": "command-palette"
    }
  ]
}
```

### Chords (sequences de touches)

Le systeme supporte les sequences de touches (chords) :

```
Ctrl+K → (attente 500ms) → Ctrl+C = Execute "commit"
```

```typescript
class ChordHandler {
  private pendingChord: string | null = null
  private chordTimeout: NodeJS.Timeout | null = null

  handleKey(key: string): string | null {
    if (this.pendingChord) {
      const chord = `${this.pendingChord},${key}`
      this.clearPending()

      const binding = this.findBinding(chord)
      return binding?.command ?? null
    }

    // Verifier si c'est le debut d'un chord
    const chordStart = this.findChordStart(key)
    if (chordStart) {
      this.pendingChord = key
      this.chordTimeout = setTimeout(
        () => this.clearPending(),
        500
      )
      return null
    }

    // Raccourci simple
    return this.findBinding(key)?.command ?? null
  }
}
```

## Mode Vim (`vim/`)

Claude Code integre un mode vim pour l'edition :

```
vim/
├── motions.ts       # Mouvements (w, b, e, $, 0, gg, G)
├── operators.ts     # Operateurs (d, c, y, p)
├── textObjects.ts   # Objets texte (iw, aw, i", a{)
├── transitions.ts   # Transitions d'etat (normal, insert, visual)
└── types.ts         # Types
```

### Modes vim

| Mode | Touches | Description |
|------|---------|-------------|
| Normal | `Escape` | Navigation et commandes |
| Insert | `i`, `a`, `o` | Saisie de texte |
| Visual | `v`, `V` | Selection |
| Command | `:` | Commandes ex |

## Pour recreer ces systemes

### 1. Hook React simple

```tsx
function useKeyboardShortcuts(
  shortcuts: Record<string, () => void>
) {
  useInput((input, key) => {
    for (const [combo, handler] of Object.entries(shortcuts)) {
      if (matchesCombo(combo, input, key)) {
        handler()
        break
      }
    }
  })
}

// Utilisation
useKeyboardShortcuts({
  'ctrl+c': () => process.exit(0),
  'ctrl+l': () => clearScreen(),
  'ctrl+r': () => searchHistory(),
})
```

### 2. Systeme de hooks d'execution

```typescript
type HookPhase = 'pre-tool' | 'post-tool' | 'pre-query' | 'post-query'

class HookSystem {
  private hooks = new Map<HookPhase, HookHandler[]>()

  register(phase: HookPhase, handler: HookHandler): void {
    const existing = this.hooks.get(phase) ?? []
    this.hooks.set(phase, [...existing, handler])
  }

  async run(phase: HookPhase, context: unknown): Promise<void> {
    const handlers = this.hooks.get(phase) ?? []
    for (const handler of handlers) {
      await handler(context)
    }
  }
}
```

### 3. Keybindings configurable

```typescript
class KeybindingManager {
  private bindings: Keybinding[] = []

  loadDefaults(): void {
    this.bindings = DEFAULT_BINDINGS
  }

  loadUserConfig(path: string): void {
    const config = JSON.parse(readFileSync(path, 'utf-8'))
    // Les config utilisateur ont priorite
    this.bindings = [...this.bindings, ...config.bindings]
  }

  resolve(key: KeyEvent): string | null {
    const match = this.bindings.find(b => matchesKey(b.keys, key))
    return match?.command ?? null
  }
}
```
