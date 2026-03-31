# 07 - Gestion d'etat

## Vue d'ensemble

Claude Code utilise un pattern de gestion d'etat combine : **React Context API** pour la distribution des donnees + un **Store centralise** pour la mutation et l'observation des changements. L'etat est immutable et versionne.

## Architecture

```
state/
├── AppState.tsx         # Context Provider React
├── AppStateStore.ts     # Store central (getState/setState)
├── onChangeAppState.ts  # Listeners de changement
└── ...

context/
├── AppStateProvider     # Distribution de l'etat
├── StatsProvider        # Statistiques de session
├── FpsMetricsProvider   # Metriques FPS terminal
├── MailboxProvider      # Messagerie inter-composants
├── VoiceProvider        # Mode voix (KAIROS)
├── NotificationProvider # Toasts et modales
└── QueuedMessageContext # File d'attente de messages
```

## AppState : l'objet d'etat central

```typescript
// Structure simplifiee de l'etat principal
type AppState = {
  // Conversation
  messages: Message[]
  selectedMessages: Set<string>
  isStreaming: boolean

  // Modele
  currentModel: string
  thinkingEnabled: boolean
  effortLevel: 'low' | 'medium' | 'high'

  // Outils
  toolPermissionContext: ToolPermissionContext
  activeToolCalls: Map<string, ToolCallState>

  // Session
  sessionId: string
  startTime: number

  // Couts
  costState: CostState
  modelUsage: ModelUsage

  // UI
  inputMode: 'normal' | 'vim' | 'plan'
  scrollPosition: number
  focusedElement: string | null

  // Config
  settings: UserSettings
  featureFlags: FeatureFlags
}
```

## Pattern Store

### AppStateStore

```typescript
class AppStateStore {
  private state: AppState
  private listeners: Set<(state: AppState) => void> = new Set()
  private version = 0

  getState(): Readonly<AppState> {
    return this.state
  }

  setState(updater: (prev: AppState) => Partial<AppState>): void {
    const updates = updater(this.state)
    this.state = { ...this.state, ...updates }
    this.version++
    this.notify()
  }

  subscribe(listener: (state: AppState) => void): () => void {
    this.listeners.add(listener)
    return () => this.listeners.delete(listener)
  }

  private notify(): void {
    for (const listener of this.listeners) {
      listener(this.state)
    }
  }
}
```

### Utilisation dans les composants

```tsx
function MessageList() {
  const { messages, isStreaming } = useAppState()

  return (
    <Box flexDirection="column">
      {messages.map(msg => (
        <MessageView key={msg.id} message={msg} />
      ))}
      {isStreaming && <StreamingIndicator />}
    </Box>
  )
}
```

## Types de messages

```typescript
type Message =
  | UserMessage              // Entree utilisateur
  | AssistantMessage         // Reponse de l'IA
  | ToolUseSummaryMessage    // Resume compact d'outils
  | AttachmentMessage        // Fichiers de memoire/contexte
  | SystemLocalCommandMessage // Sortie d'execution d'outil
  | TombstoneMessage         // Messages supprimes (placeholder)
  | ProgressMessage          // Mises a jour de progression
```

### UserMessage

```typescript
type UserMessage = {
  type: 'user'
  id: string
  content: string | ContentBlock[]
  timestamp: number
  attachments?: Attachment[]
}
```

### AssistantMessage

```typescript
type AssistantMessage = {
  type: 'assistant'
  id: string
  content: ContentBlock[]  // text, tool_use, thinking
  timestamp: number
  model: string
  usage: TokenUsage
  stopReason: 'end_turn' | 'tool_use' | 'max_tokens'
}
```

### ToolUseSummaryMessage

Quand l'historique est compacte, les resultats d'outils detailles sont remplaces par des resumes :

```typescript
type ToolUseSummaryMessage = {
  type: 'tool_use_summary'
  id: string
  toolName: string
  summary: string  // "Read file /src/app.ts (200 lines)"
  originalIds: string[]
}
```

## Context Providers

### Hierarchie des providers

```tsx
<AppStateProvider store={store}>
  <StatsProvider>
    <FpsMetricsProvider>
      <NotificationProvider>
        <MailboxProvider>
          <QueuedMessageContext.Provider>
            <App />
          </QueuedMessageContext.Provider>
        </MailboxProvider>
      </NotificationProvider>
    </FpsMetricsProvider>
  </StatsProvider>
</AppStateProvider>
```

### StatsProvider
Suit les statistiques de session en temps reel :
- Nombre de messages echanges
- Tokens utilises (input/output)
- Outils executes
- Duree de la session

### MailboxProvider
Permet la communication entre composants distants dans l'arbre :

```typescript
// Composant A envoie un message
const { send } = useMailbox()
send('tool-panel', { type: 'highlight', toolId: '123' })

// Composant B recoit le message
const messages = useMailbox('tool-panel')
```

### NotificationProvider
Gere les toasts et modales :

```typescript
const { notify } = useNotification()
notify({
  type: 'success',
  message: 'File saved successfully',
  duration: 3000
})
```

## Listeners de changement : `onChangeAppState.ts`

Les listeners reagissent aux changements d'etat pour produire des effets secondaires :

```typescript
// Pseudo-code
store.subscribe((newState) => {
  // Sauvegarder la session sur disque
  if (stateChanged(newState, 'messages')) {
    saveSessionToDisk(newState)
  }

  // Mettre a jour le titre du terminal
  if (stateChanged(newState, 'currentModel')) {
    setTerminalTitle(`Claude Code - ${newState.currentModel}`)
  }

  // Tracker les couts
  if (stateChanged(newState, 'modelUsage')) {
    updateCostTracker(newState.modelUsage)
  }
})
```

## Pour recreer cette partie

### 1. Store minimal

```typescript
type Listener<T> = (state: T) => void

function createStore<T>(initial: T) {
  let state = initial
  const listeners = new Set<Listener<T>>()

  return {
    getState: () => state,
    setState: (updater: (s: T) => Partial<T>) => {
      state = { ...state, ...updater(state) }
      listeners.forEach(l => l(state))
    },
    subscribe: (l: Listener<T>) => {
      listeners.add(l)
      return () => listeners.delete(l)
    }
  }
}
```

### 2. React Context integration

```tsx
const StoreContext = createContext<ReturnType<typeof createStore>>(null!)

function useAppState() {
  const store = useContext(StoreContext)
  const [state, setState] = useState(store.getState())

  useEffect(() => {
    return store.subscribe(setState)
  }, [store])

  return state
}
```

### 3. Points cles

- **Immutabilite** : Ne jamais muter l'etat directement
- **Listeners** : Permettre aux side-effects de reagir aux changements
- **Selecteurs** : Permettre aux composants de s'abonner a des sous-parties de l'etat
- **Serialisation** : L'etat doit etre serialisable pour la persistence de session
