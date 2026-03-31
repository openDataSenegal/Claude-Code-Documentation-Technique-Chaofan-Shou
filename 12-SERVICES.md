# 12 - Services

## Vue d'ensemble

La couche services fournit les fonctionnalites transversales : communication API, analytics, authentification, suivi des couts, et gestion de configuration.

## Architecture

```
services/
├── api/
│   └── claude.ts            # Client API Anthropic (~3419 lignes)
├── analytics/
│   ├── index.ts             # Evenements et telemetrie
│   └── growthbook.ts        # Feature flags runtime
├── tools/
│   ├── StreamingToolExecutor.ts  # Execution des outils
│   ├── toolExecution.ts     # Logique d'execution
│   ├── toolOrchestration.ts # Orchestration multi-outils
│   └── toolHooks.ts         # Hooks pre/post execution
├── autoDream/
│   ├── config.ts            # Configuration consolidation
│   ├── consolidationPrompt.ts # Prompt de consolidation
│   └── consolidationLock.ts # Verrou d'execution
└── diagnosticTracking.ts    # Metriques de performance
```

## Client API : `services/api/claude.ts`

### Responsabilites

1. **Envoi de messages en streaming** vers l'API Anthropic
2. **Suivi de l'utilisation** (tokens, cache, cout)
3. **Gestion des erreurs** avec categorisation et retry
4. **Headers de fonctionnalites beta** (thinking, structured outputs)
5. **Attestation client** pour la verification d'identite
6. **Metadata de facturation**

### Appel API type

```typescript
async function sendMessage(params: {
  model: string
  messages: APIMessage[]
  system: SystemPromptPart[]
  tools: ToolDefinition[]
  maxTokens: number
  stream: true
  thinking?: { type: 'enabled'; budget_tokens: number }
}): Promise<StreamingResponse> {
  const response = await anthropic.messages.create({
    ...params,
    // Headers supplementaires
    headers: {
      'x-anthropic-billing-header': billingMetadata,
      ...(nativeAttestation && {
        'x-anthropic-client-attestation': attestationToken
      })
    }
  })

  return response
}
```

### Streaming

Le client utilise le streaming SSE (Server-Sent Events) :

```
Client                          API
  │                               │
  │── POST /messages (stream) ──→│
  │                               │
  │←── event: message_start ─────│
  │←── event: content_block_start│
  │←── event: content_block_delta│  (x N, texte incremental)
  │←── event: content_block_stop │
  │←── event: message_delta ─────│
  │←── event: message_stop ──────│
```

### Suivi de l'utilisation

```typescript
type TokenUsage = {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens: number
  cache_read_input_tokens: number
}

// Accumulation par session
class UsageTracker {
  private totalUsage: TokenUsage = { /* zeros */ }

  accumulate(usage: TokenUsage): void {
    this.totalUsage.input_tokens += usage.input_tokens
    this.totalUsage.output_tokens += usage.output_tokens
    this.totalUsage.cache_creation_input_tokens +=
      usage.cache_creation_input_tokens
    this.totalUsage.cache_read_input_tokens +=
      usage.cache_read_input_tokens
  }

  getCost(): number {
    return calculateCost(this.totalUsage, this.model)
  }
}
```

### Categorisation des erreurs

```typescript
function categorizeError(error: APIError): ErrorCategory {
  switch (error.status) {
    case 400: return 'invalid_request'    // Non-retryable
    case 401: return 'authentication'     // Re-auth needed
    case 403: return 'forbidden'          // Non-retryable
    case 404: return 'not_found'          // Non-retryable
    case 429: return 'rate_limited'       // Retryable
    case 500: return 'server_error'       // Retryable
    case 502: return 'bad_gateway'        // Retryable
    case 503: return 'overloaded'         // Retryable
    default:  return 'unknown'            // Retryable
  }
}
```

## Suivi des couts : `cost-tracker.ts`

### Metriques suivies

```typescript
type CostMetrics = {
  // Tokens
  inputTokens: number
  outputTokens: number
  cacheCreationTokens: number
  cacheReadTokens: number

  // Requetes
  webSearchRequests: number

  // Fichiers
  linesAdded: number
  linesRemoved: number

  // Duree
  totalDuration: number
  durationWithoutRetries: number

  // Outils
  toolExecutionDuration: number
}
```

### Calcul des couts : `utils/modelCost.ts`

```typescript
const MODEL_PRICING: Record<string, ModelPricing> = {
  'claude-opus-4-6': {
    input: 15.00,   // $ per million tokens
    output: 75.00,
    cacheCreation: 18.75,
    cacheRead: 1.50
  },
  'claude-sonnet-4-6': {
    input: 3.00,
    output: 15.00,
    cacheCreation: 3.75,
    cacheRead: 0.30
  },
  'claude-haiku-4-5': {
    input: 0.80,
    output: 4.00,
    cacheCreation: 1.00,
    cacheRead: 0.08
  }
}

function calculateCost(usage: TokenUsage, model: string): number {
  const pricing = MODEL_PRICING[model]
  return (
    (usage.input_tokens * pricing.input / 1_000_000) +
    (usage.output_tokens * pricing.output / 1_000_000) +
    (usage.cache_creation_input_tokens * pricing.cacheCreation / 1_000_000) +
    (usage.cache_read_input_tokens * pricing.cacheRead / 1_000_000)
  )
}
```

### Affichage : `costHook.ts`

A la fin de la session, affiche un resume des couts :

```
Session Summary:
  Model: claude-sonnet-4-6
  Duration: 12m 34s
  Input tokens: 45,230
  Output tokens: 8,901
  Cache reads: 32,100
  Total cost: $0.28
```

## Analytics et telemetrie

### Evenements

```typescript
type AnalyticsEvent = {
  name: string
  properties: Record<string, unknown>
  timestamp: number
  sessionId: string
}

// Exemples d'evenements
track('session_start', { model: 'claude-sonnet-4-6' })
track('tool_executed', { tool: 'Bash', duration: 1200 })
track('command_used', { command: 'commit' })
track('error_occurred', { type: 'rate_limited', retried: true })
```

### Scrubbing PII

Avant l'envoi, les evenements sont nettoyes :

```typescript
function scrubPII(event: AnalyticsEvent): AnalyticsEvent {
  const scrubbed = { ...event }
  // Supprimer les chemins absolus
  scrubbed.properties = mapValues(scrubbed.properties, (v) => {
    if (typeof v === 'string') {
      return v.replace(/\/Users\/[^/]+/g, '/Users/<redacted>')
    }
    return v
  })
  return scrubbed
}
```

### GrowthBook (Feature Flags)

```typescript
// Initialisation
const gb = new GrowthBook({
  apiHost: 'https://cdn.growthbook.io',
  clientKey: 'sdk-xxx',
  enableDevMode: isDev
})

// Utilisation
if (gb.isOn('new_compaction_strategy')) {
  useReactiveCompact()
} else {
  useLegacyCompact()
}
```

**Cache stale-acceptable** : Les valeurs en cache (jusqu'a 10% de stale) sont utilisees pour eviter de bloquer le demarrage.

## Authentification : `utils/auth.ts`

### Methodes d'authentification

1. **Variable d'environnement** : `ANTHROPIC_API_KEY`
2. **Keychain macOS** : Stockage securise natif
3. **Fichier de configuration** : `~/.claude/config.json`
4. **OAuth** : Pour les deployments entreprise

### Flux d'authentification

```
Demarrage
    │
    ▼
Verifier ANTHROPIC_API_KEY env var
    ├── Present → Utiliser
    │
    ├── Absent → Verifier keychain
    │   ├── Present → Utiliser
    │   │
    │   └── Absent → Verifier config file
    │       ├── Present → Utiliser
    │       │
    │       └── Absent → Demander login
    │           └── claude-code login
```

### Pre-chargement du keychain

Le keychain macOS est lent (~200ms). Le systeme le pre-charge en parallele :

```typescript
// Lance pendant l'import des modules
const keychainPromise = readFromKeychain('anthropic-api-key')

// Utilise plus tard quand necessaire
const apiKey = await keychainPromise
```

## Pour recreer ces services

### 1. Client API minimal

```typescript
import Anthropic from '@anthropic-ai/sdk'

class APIService {
  private client: Anthropic
  private usage = { input: 0, output: 0 }

  constructor(apiKey: string) {
    this.client = new Anthropic({ apiKey })
  }

  async *stream(params: MessageParams): AsyncGenerator<StreamEvent> {
    const stream = this.client.messages.stream(params)

    for await (const event of stream) {
      yield event

      if (event.type === 'message_delta' && event.usage) {
        this.usage.input += event.usage.input_tokens ?? 0
        this.usage.output += event.usage.output_tokens ?? 0
      }
    }
  }

  getCost(model: string): number {
    return calculateCost(this.usage, model)
  }
}
```

### 2. Tracker de couts

```typescript
class CostTracker {
  private metrics: CostMetrics = { /* zeros */ }

  recordAPICall(usage: TokenUsage): void {
    this.metrics.inputTokens += usage.input_tokens
    this.metrics.outputTokens += usage.output_tokens
  }

  recordToolExecution(duration: number): void {
    this.metrics.toolExecutionDuration += duration
  }

  getSummary(model: string): string {
    const cost = calculateCost(this.metrics, model)
    return `Tokens: ${this.metrics.inputTokens}in / ${this.metrics.outputTokens}out | Cost: $${cost.toFixed(4)}`
  }
}
```

### 3. Analytics simple

```typescript
class Analytics {
  private events: AnalyticsEvent[] = []

  track(name: string, properties: Record<string, unknown> = {}): void {
    this.events.push({
      name,
      properties: scrubPII(properties),
      timestamp: Date.now(),
      sessionId: this.sessionId
    })
  }

  async flush(): Promise<void> {
    if (this.events.length === 0) return
    await sendBatch(this.events)
    this.events = []
  }
}
```
