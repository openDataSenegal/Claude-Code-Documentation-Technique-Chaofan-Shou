# 03 - Systeme de requetes (Query Engine)

## Vue d'ensemble

Le Query Engine est le coeur du systeme. Il gere le cycle de vie complet d'une interaction avec l'API Claude : preparation des messages, envoi, reception en streaming, execution des outils, et boucle de continuation.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  query.ts (~1729 lignes)             │
│  Preparation des messages, normalisation, contexte  │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│              QueryEngine.ts (~1295 lignes)           │
│  Cycle de vie, boucle tool-use, retry, permissions  │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────┐
│           services/api/claude.ts (~3419 lignes)     │
│  Communication API, streaming, usage tracking       │
└─────────────────────────────────────────────────────┘
```

## `query.ts` - Preparation des messages

### Responsabilites
- Normaliser les messages utilisateur au format API Anthropic
- Gerer les fichiers attaches (memoire, contexte, images)
- Generer des resumes compacts des executions d'outils precedentes
- Gerer la fenetre de contexte (budget de tokens, compaction)
- Controle du cache de prompt (invalidation strategique)
- Selection du modele a runtime

### Pipeline de preparation

```
Input utilisateur brut
    │
    ▼
processUserInput()
    ├── Parse des commandes slash (/commit, /help, etc.)
    ├── Expansion des patterns glob dans les mentions de fichiers
    ├── Resolution des pieces jointes (fichiers, images, clipboard)
    └── Gestion du collage d'images
    │
    ▼
fetchSystemPromptParts()
    ├── Sections statiques (cachables par l'API)
    │   ├── Instructions de role
    │   ├── Instructions de securite
    │   ├── Regles d'utilisation des outils
    │   └── Style et ton
    │
    ├── Sections dynamiques (volatiles)
    │   ├── Statut git actuel
    │   ├── Fichiers CLAUDE.md
    │   ├── Memoires chargees (MEMORY.md)
    │   ├── Date courante
    │   └── Configuration du projet
    │
    └── Cache-break signals (separateurs cache/volatile)
    │
    ▼
normalizeMessages()
    ├── Conversion des messages internes → format API
    ├── Injection des resumes d'outils (compaction)
    ├── Gestion des messages tombstone (supprimes)
    └── Troncature si depassement du budget tokens
    │
    ▼
Messages prets pour l'API
```

### Strategie de cache de prompt

L'API Anthropic supporte le caching des prefixes de prompt. Le systeme optimise cela en :

1. **Sections stables en premier** : Instructions de role, securite, regles d'outils
2. **Point de rupture du cache** : Signal explicite entre stable et volatile
3. **Sections volatiles en dernier** : Git status, memoires, date

Cela maximise le taux de cache hit et reduit les couts.

## `QueryEngine.ts` - Moteur d'execution

### Cycle de vie d'une requete

```
                    ┌─────────────────────┐
                    │  Construire messages │
                    │  systeme + user      │
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │  Configurer thinking │
                    │  (si extended)       │
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │  Appel API streaming │◄──────────┐
                    └──────────┬──────────┘           │
                               ▼                       │
                    ┌─────────────────────┐           │
                    │  Recevoir reponse    │           │
                    └──────────┬──────────┘           │
                               ▼                       │
                    ┌─────────────────────┐           │
                    │  Contient tool_use? │           │
                    └──────┬───────┬──────┘           │
                      Non  │       │ Oui              │
                           ▼       ▼                   │
                    ┌──────┐ ┌──────────────┐         │
                    │ FIN  │ │ Verifier     │         │
                    └──────┘ │ permissions  │         │
                             └──────┬───────┘         │
                                    ▼                  │
                             ┌──────────────┐         │
                             │ Executer     │         │
                             │ outil(s)     │         │
                             └──────┬───────┘         │
                                    ▼                  │
                             ┌──────────────┐         │
                             │ Collecter    │         │
                             │ resultats    │─────────┘
                             └──────────────┘
```

### Boucle tool-use

Le QueryEngine implemente une boucle de continuation automatique :

1. L'API retourne un bloc `tool_use` dans la reponse
2. Le moteur verifie les permissions pour chaque outil
3. Si approuve : execute l'outil et collecte le resultat
4. Si refuse : retourne un message d'erreur de permission
5. Les resultats sont ajoutes aux messages et un nouvel appel API est fait
6. La boucle continue jusqu'a ce que l'API retourne du texte sans tool_use

### Gestion des permissions dans la boucle

```typescript
// Pseudo-code simplifie
for (const toolCall of response.toolCalls) {
  const permission = await checkPermission(toolCall)

  switch (permission) {
    case 'approved':
      const result = await executeTool(toolCall)
      results.push({ toolCallId: toolCall.id, result })
      break

    case 'denied':
      results.push({
        toolCallId: toolCall.id,
        result: 'Permission denied by user'
      })
      break

    case 'ask_user':
      const userDecision = await promptUser(toolCall)
      // ... traiter la decision
      break
  }
}
```

### Retry avec backoff exponentiel

```
Tentative 1 → Echec → Attente 1s
Tentative 2 → Echec → Attente 2s
Tentative 3 → Echec → Attente 4s
Tentative 4 → Echec → Attente 8s
Tentative 5 → Echec → Erreur fatale
```

Les erreurs sont categoriees :
- **Retryable** : 429 (rate limit), 500, 502, 503
- **Non-retryable** : 400, 401, 403, 404

## `services/api/claude.ts` - Communication API

### Responsabilites
- Envoi des messages en streaming
- Suivi de l'utilisation (tokens, cache, cout)
- Categorisation des erreurs et strategies de retry
- Headers de fonctionnalites beta (thinking, structured outputs)
- Attestation client
- Metadata de facturation

### Format d'appel API

```typescript
// Structure simplifiee d'un appel
const response = await anthropic.messages.create({
  model: 'claude-opus-4-6',
  max_tokens: 16384,
  system: systemPromptParts,
  messages: normalizedMessages,
  tools: filteredToolDefinitions,
  stream: true,
  // Options conditionnelles
  ...(thinkingEnabled && {
    thinking: { type: 'enabled', budget_tokens: 10000 }
  })
})
```

### Suivi de l'utilisation

Chaque reponse API contient des metriques d'utilisation :

```typescript
type Usage = {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens: number
  cache_read_input_tokens: number
}
```

Le systeme accumule ces metriques par session pour le calcul des couts.

## Gestion de la fenetre de contexte

### Compaction des messages

Quand la conversation approche de la limite de contexte :

1. **Detection** : Estimation du nombre de tokens des messages
2. **Selection** : Identification des messages compressibles (anciens outils, resultats volumineux)
3. **Compression** : Remplacement par des resumes compacts
4. **Verification** : Re-estimation pour confirmer que le budget est respecte

### Strategies de compaction

| Strategie | Declencheur | Action |
|-----------|-------------|--------|
| **Tool summarization** | Resultat d'outil > N tokens | Resume en 1-2 lignes |
| **Message tombstone** | Message utilisateur ancien | Remplacement par placeholder |
| **History snip** | Conversation tres longue | Suppression des messages les plus anciens |
| **Reactive compact** | Approche limite critique | Compaction aggressive |

## Pour recreer cette partie

### Etape 1 : Client API
```typescript
import Anthropic from '@anthropic-ai/sdk'

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY })
```

### Etape 2 : Boucle de requete basique
```typescript
async function queryLoop(messages: Message[], tools: Tool[]) {
  while (true) {
    const response = await client.messages.create({
      model: 'claude-sonnet-4-6',
      max_tokens: 8192,
      messages,
      tools: tools.map(t => t.definition),
      stream: true
    })

    // Accumuler la reponse streamee
    const assistantMessage = await accumulateStream(response)
    messages.push({ role: 'assistant', content: assistantMessage.content })

    // Verifier s'il y a des appels d'outils
    const toolCalls = assistantMessage.content.filter(
      block => block.type === 'tool_use'
    )

    if (toolCalls.length === 0) break // Fin de la boucle

    // Executer les outils
    const toolResults = await Promise.all(
      toolCalls.map(async call => ({
        type: 'tool_result',
        tool_use_id: call.id,
        content: await executeTool(call.name, call.input)
      }))
    )

    messages.push({ role: 'user', content: toolResults })
  }
}
```

### Etape 3 : Gestion du prompt systeme
```typescript
function buildSystemPrompt(context: Context): string {
  return [
    ROLE_INSTRUCTIONS,        // Stable, cachable
    TOOL_USAGE_RULES,         // Stable, cachable
    SECURITY_INSTRUCTIONS,    // Stable, cachable
    // --- cache break ---
    `Date: ${new Date().toISOString()}`,  // Volatile
    `Git status: ${context.gitStatus}`,   // Volatile
    context.claudeMd,                     // Volatile
    context.memories                      // Volatile
  ].join('\n\n')
}
```

### Etape 4 : Retry logic
```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 5
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      if (!isRetryable(error) || attempt === maxRetries - 1) throw error
      await sleep(Math.pow(2, attempt) * 1000)
    }
  }
  throw new Error('Unreachable')
}
```
