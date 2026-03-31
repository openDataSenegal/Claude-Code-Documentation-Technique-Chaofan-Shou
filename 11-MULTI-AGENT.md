# 11 - Multi-agent et orchestration

## Vue d'ensemble

Claude Code supporte l'orchestration de plusieurs agents IA travaillant en parallele. Le systeme comprend l'**AgentTool** (sous-agents), le **mode Coordinator** (orchestration avancee), et le **systeme de taches** (background tasks).

## Architecture

```
coordinator/
└── coordinatorMode.ts    # Prompt et logique du coordinateur

tools/
├── AgentTool/            # Creation de sous-agents
├── TeamCreateTool/       # Gestion d'equipes
├── TeamDeleteTool/       # Suppression d'equipes
├── SendMessageTool/      # Messagerie inter-agents
├── TaskCreateTool/       # Taches en arriere-plan
├── TaskGetTool/          # Statut des taches
├── TaskListTool/         # Liste des taches
├── TaskUpdateTool/       # Mise a jour des taches
├── EnterWorktreeTool/    # Isolation git
└── ExitWorktreeTool/     # Sortie du worktree
```

## AgentTool : sous-agents

### Principe

L'AgentTool permet a l'IA principale de creer des sous-agents qui operent dans leur propre contexte. Chaque sous-agent :
- A sa propre conversation avec l'API
- Peut utiliser un sous-ensemble d'outils
- Retourne un resultat a l'agent parent
- Peut operer dans un worktree git isole

### Cycle de vie d'un sous-agent

```
Agent principal
    │
    ├── Cree AgentTool avec prompt + type
    │
    ▼
Sous-agent demarre
    ├── Recoit le prompt initial
    ├── Obtient ses outils disponibles
    ├── Execute sa tache (boucle query + tools)
    └── Retourne le resultat
    │
    ▼
Agent principal recoit le resultat
    └── Continue son travail
```

### Types de sous-agents

```typescript
type SubagentType =
  | 'general-purpose'  // Agent polyvalent
  | 'Explore'          // Exploration rapide du codebase
  | 'Plan'             // Conception de plans d'implementation
  | 'claude-code-guide' // Guide d'utilisation
```

### Exemple d'utilisation

```typescript
// L'IA principale invoque l'AgentTool
{
  name: "Agent",
  input: {
    description: "Find auth middleware",
    prompt: "Search the codebase for all authentication middleware files and summarize their structure.",
    subagent_type: "Explore"
  }
}
```

### Isolation via worktree

Un sous-agent peut operer dans un git worktree isole :

```
Repo principal (/project)
    │
    ├── Agent 1 → /tmp/worktree-abc123 (branche feature-a)
    ├── Agent 2 → /tmp/worktree-def456 (branche feature-b)
    └── Agent 3 → /project (meme repertoire)
```

Cela permet des modifications paralleles sans conflits.

## Mode Coordinator

### Principe

Le mode Coordinator (feature gate: `COORDINATOR_MODE`) transforme l'agent principal en chef d'orchestre qui delegue le travail a des agents workers.

### Phases d'orchestration

| Phase | Acteur | Objectif |
|-------|--------|----------|
| **Research** | Workers | Investiguer le codebase, collecter des informations |
| **Synthesis** | Coordinator | Comprendre le probleme, rediger les specifications |
| **Implementation** | Workers | Effectuer les modifications ciblees |
| **Verification** | Workers | Tester et valider les changements |

### Flux de travail

```
Utilisateur : "Refactor the auth system to use JWT"
    │
    ▼
COORDINATOR (phase Research)
    ├── Spawn Worker 1 : "Find all auth-related files"
    ├── Spawn Worker 2 : "Analyze current session management"
    └── Spawn Worker 3 : "Review existing JWT usage if any"
    │
    │   (execution parallele)
    │
    ▼
COORDINATOR (phase Synthesis)
    ├── Recoit les resultats de recherche
    ├── Synthetise un plan d'implementation
    └── Decoupe en taches atomiques
    │
    ▼
COORDINATOR (phase Implementation)
    ├── Spawn Worker A : "Create JWT utility module"
    ├── Spawn Worker B : "Update login endpoint"
    └── Spawn Worker C : "Update middleware chain"
    │
    │   (execution parallele, worktrees isoles)
    │
    ▼
COORDINATOR (phase Verification)
    ├── Spawn Worker : "Run test suite"
    ├── Spawn Worker : "Check for regressions"
    └── Review des changements
    │
    ▼
Rapport final a l'utilisateur
```

### Communication inter-agents

```typescript
// Agent notifie le coordinateur
{
  type: 'task-notification',
  agentId: 'worker-1',
  status: 'completed',
  result: 'Found 12 auth-related files in /src/auth/'
}

// Coordinateur envoie une directive
{
  type: 'directive',
  targetAgent: 'worker-2',
  task: 'Update middleware to validate JWT tokens',
  context: { files: [...], specs: '...' }
}
```

### Scratchpad partage

Les agents partagent un espace de notes durables :

```typescript
// Worker ecrit dans le scratchpad
await scratchpad.write('auth-analysis', {
  findings: 'Current auth uses session cookies...',
  risks: ['Session fixation vulnerability', ...],
  files: ['/src/auth/session.ts', ...]
})

// Coordinator lit le scratchpad
const analysis = await scratchpad.read('auth-analysis')
```

## Systeme de taches

### TaskCreateTool

Cree une tache qui s'execute en arriere-plan :

```typescript
{
  name: "TaskCreate",
  input: {
    description: "Run full test suite",
    command: "npm test",
    background: true
  }
}
```

### Cycle de vie d'une tache

```
PENDING → RUNNING → COMPLETED
                  → FAILED
                  → CANCELLED
```

### Suivi des taches

```typescript
// Creer une tache
const task = await TaskCreate({
  description: "Lint check",
  command: "npm run lint"
})
// task.id = "task-123"

// Verifier le statut
const status = await TaskGet({ id: "task-123" })
// status = { state: 'running', progress: '45%' }

// Lister toutes les taches
const tasks = await TaskList()
// [{ id: 'task-123', state: 'running' }, ...]
```

## Gestion des couleurs des agents

Chaque agent recoit une couleur distincte pour l'identification visuelle :

```typescript
const AGENT_COLORS = [
  'cyan',
  'magenta',
  'yellow',
  'green',
  'blue',
  'red',
  'white'
]

function assignColor(agentIndex: number): string {
  return AGENT_COLORS[agentIndex % AGENT_COLORS.length]
}
```

## Pour recreer ce systeme

### 1. Sous-agent basique

```typescript
async function spawnSubagent(
  prompt: string,
  tools: Tool[],
  parentContext: Context
): Promise<string> {
  const subContext = createSubContext(parentContext)

  const messages: Message[] = [
    { role: 'user', content: prompt }
  ]

  // Boucle de requete independante
  while (true) {
    const response = await callAPI({
      messages,
      tools: tools.map(t => t.definition),
      system: buildSubagentSystemPrompt(subContext)
    })

    messages.push({ role: 'assistant', content: response.content })

    const toolCalls = extractToolCalls(response)
    if (toolCalls.length === 0) {
      // L'agent a termine
      return extractTextContent(response)
    }

    const results = await executeTools(toolCalls, subContext)
    messages.push({ role: 'user', content: results })
  }
}
```

### 2. Coordinateur

```typescript
class Coordinator {
  private workers: Map<string, Worker> = new Map()

  async orchestrate(task: string): Promise<void> {
    // Phase 1: Research
    const researchResults = await Promise.all([
      this.spawnWorker('research-1', `Analyze: ${task}`),
      this.spawnWorker('research-2', `Find related code for: ${task}`)
    ])

    // Phase 2: Synthesis
    const plan = await this.synthesize(researchResults)

    // Phase 3: Implementation
    const implResults = await Promise.all(
      plan.tasks.map((subtask, i) =>
        this.spawnWorker(`impl-${i}`, subtask.prompt, {
          worktree: true
        })
      )
    )

    // Phase 4: Verification
    await this.verify(implResults)
  }

  private async spawnWorker(
    id: string,
    prompt: string,
    options?: { worktree?: boolean }
  ): Promise<string> {
    const worker = new Worker(id, options)
    this.workers.set(id, worker)
    return worker.execute(prompt)
  }
}
```

### 3. Systeme de taches

```typescript
class TaskManager {
  private tasks = new Map<string, Task>()

  create(description: string, fn: () => Promise<void>): string {
    const id = generateId()
    const task: Task = {
      id,
      description,
      state: 'pending',
      createdAt: Date.now()
    }

    this.tasks.set(id, task)

    // Executer en arriere-plan
    this.run(id, fn)

    return id
  }

  private async run(id: string, fn: () => Promise<void>): Promise<void> {
    const task = this.tasks.get(id)!
    task.state = 'running'

    try {
      await fn()
      task.state = 'completed'
    } catch (error) {
      task.state = 'failed'
      task.error = error.message
    }
  }

  get(id: string): Task | undefined {
    return this.tasks.get(id)
  }

  list(): Task[] {
    return Array.from(this.tasks.values())
  }
}
```

### Points cles

1. **Parallelisme** : Lancer les agents independants en parallele
2. **Isolation** : Worktrees git pour eviter les conflits
3. **Communication** : Messages structures entre agents
4. **Scratchpad** : Espace partage pour les connaissances durables
5. **Couleurs** : Identification visuelle des agents
6. **Taches async** : Execution en arriere-plan avec suivi d'etat
