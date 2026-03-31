# 04 - Systeme d'outils (Tools)

## Vue d'ensemble

Le systeme d'outils est l'un des piliers de Claude Code. Il permet a l'IA d'executer des actions concretes : lire/ecrire des fichiers, executer du shell, chercher sur le web, gerer des agents, etc. Le projet compte **41+ outils** organises en categories.

## Architecture

```
Tool.ts                    # Interface de base
    │
tools.ts                   # Registre central (import, filtrage, export)
    │
tools/                     # Implementations (1 dossier par outil)
    ├── BashTool/
    ├── FileReadTool/
    ├── FileEditTool/
    ├── AgentTool/
    └── ... (45 dossiers)
    │
tools/permissions/         # Systeme de permissions
    │
services/tools/            # Execution et orchestration
    ├── StreamingToolExecutor.ts
    ├── toolExecution.ts
    ├── toolOrchestration.ts
    └── toolHooks.ts
```

## Interface de base : `Tool.ts`

```typescript
type ToolInputJSONSchema = {
  type: 'object'
  properties: Record<string, JSONSchema>
  required?: string[]
}

type ToolResult = {
  content: string | ContentBlock[]
  isError?: boolean
}

type Tool = {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema
  execute: (
    input: Record<string, unknown>,
    context: ToolUseContext
  ) => Promise<string | ToolResult>
}
```

### `ToolUseContext`
Le contexte passe a chaque outil lors de l'execution :

```typescript
type ToolUseContext = {
  workingDirectory: string
  permissions: PermissionContext
  abortSignal: AbortSignal
  onProgress: (progress: string) => void
  sessionId: string
  // ... autres champs contextuels
}
```

## Catalogue complet des outils

### Operations sur les fichiers

| Outil | Description | Risque |
|-------|------------|--------|
| **FileReadTool** | Lire le contenu d'un fichier avec support de plages de lignes | LOW |
| **FileWriteTool** | Creer/ecraser un fichier (avec limites de taille) | MEDIUM |
| **FileEditTool** | Edition diff-based avec rollback possible | MEDIUM |
| **GlobTool** | Recherche de fichiers par pattern (utilise `bfs` natif si dispo) | LOW |
| **GrepTool** | Recherche regex dans les fichiers (utilise `ugrep` natif si dispo) | LOW |
| **NotebookEditTool** | Manipulation de cellules Jupyter notebook | MEDIUM |

### Shell et execution de code

| Outil | Description | Risque |
|-------|------------|--------|
| **BashTool** | Execution shell avec sandboxing optionnel | HIGH |
| **PowerShellTool** | Execution PowerShell (Windows) | HIGH |
| **REPLTool** | REPL interactif (interne uniquement) | HIGH |
| **LSPTool** | Communication avec Language Server Protocol | LOW |

### Web et reseau

| Outil | Description | Risque |
|-------|------------|--------|
| **WebFetchTool** | Requete HTTP avec conversion HTML→markdown | LOW |
| **WebSearchTool** | Recherche web via provider configure | LOW |
| **MCPTool** | Acces aux ressources Model Context Protocol | MEDIUM |

### Agents et gestion de taches

| Outil | Description | Risque |
|-------|------------|--------|
| **AgentTool** | Creer des sous-agents avec leur propre contexte | MEDIUM |
| **TeamCreateTool** | Gestion d'equipes d'agents | MEDIUM |
| **TeamDeleteTool** | Suppression d'equipes d'agents | MEDIUM |
| **SendMessageTool** | Messagerie inter-agents | LOW |
| **TaskCreateTool** | Creer une tache en arriere-plan | LOW |
| **TaskGetTool** | Obtenir le statut d'une tache | LOW |
| **TaskListTool** | Lister les taches | LOW |
| **TaskUpdateTool** | Mettre a jour une tache | LOW |

### Navigation et planification

| Outil | Description | Risque |
|-------|------------|--------|
| **EnterPlanModeTool** | Passer en mode planification | LOW |
| **ExitPlanModeV2Tool** | Quitter le mode planification | LOW |
| **EnterWorktreeTool** | Creer un worktree git isole | MEDIUM |
| **ExitWorktreeTool** | Quitter le worktree | MEDIUM |

### Fonctionnalites specialisees

| Outil | Description | Risque |
|-------|------------|--------|
| **SkillTool** | Invoquer des skills utilisateur ou integres | MEDIUM |
| **BriefTool** | Upload/resume de fichiers | LOW |
| **AskUserQuestionTool** | Poser une question a l'utilisateur | LOW |
| **ToolSearchTool** | Decouvrir les outils disponibles | LOW |
| **SleepTool** | Delais asynchrones (KAIROS only) | LOW |
| **RemoteTriggerTool** | Declencher des agents distants | MEDIUM |
| **ScheduleCronTool** | Planification cron (Create/Delete/List) | MEDIUM |
| **ConfigTool** | Modification des settings (interne) | MEDIUM |

## Registre d'outils : `tools.ts`

### Chargement et filtrage

```typescript
// Pseudo-code simplifie du registre
function getAvailableTools(context: ToolContext): Tool[] {
  const allTools = [
    new FileReadTool(),
    new FileWriteTool(),
    new FileEditTool(),
    new BashTool(),
    new GlobTool(),
    new GrepTool(),
    // ...
    // Outils conditionnes par feature flags
    ...(feature('KAIROS') ? [new SleepTool()] : []),
    ...(feature('BRIDGE_MODE') ? [new RemoteTriggerTool()] : []),
  ]

  return allTools.filter(tool => {
    // Filtrage par type d'utilisateur
    if (tool.antOnly && userType !== 'ant') return false
    // Filtrage par permission
    if (isDenied(tool, context.permissions)) return false
    // Filtrage par feature flag
    if (tool.featureGate && !feature(tool.featureGate)) return false
    return true
  })
}
```

### Cache des schemas

```typescript
// Evite la re-serialisation des schemas JSON a chaque appel
const schemaCache = new Map<string, ToolSchema>()

function getToolSchema(tool: Tool): ToolSchema {
  if (schemaCache.has(tool.name)) return schemaCache.get(tool.name)!
  const schema = computeSchema(tool)
  schemaCache.set(tool.name, schema)
  return schema
}
```

## Execution des outils

### `StreamingToolExecutor`

Executeur principal qui gere le streaming des resultats d'outils vers l'API :

```
Appel API retourne tool_use
    │
    ▼
StreamingToolExecutor
    ├── Parse le bloc tool_use
    ├── Identifie l'outil dans le registre
    ├── Verifie les permissions
    │   ├── AUTO → execute directement
    │   ├── ASK  → prompt utilisateur
    │   └── DENY → retourne erreur
    ├── Execute l'outil avec contexte
    ├── Stream les progres (onProgress)
    ├── Collecte le resultat final
    └── Formate pour l'API (tool_result)
```

### Execution parallele

Quand l'API retourne plusieurs `tool_use` dans une meme reponse :

```typescript
// Les outils independants sont executes en parallele
const results = await Promise.all(
  toolCalls.map(call => executeWithPermission(call, context))
)
```

### Hooks d'execution

```
Pre-execution hooks
    ├── Validation des inputs
    ├── Journalisation
    └── Injection de contexte
    │
    ▼
Execution de l'outil
    │
    ▼
Post-execution hooks
    ├── Filtrage de la sortie
    ├── Metriques de performance
    └── Mise a jour de l'etat
```

## Implementation d'un outil : exemple

### Structure d'un dossier d'outil

```
tools/
└── FileReadTool/
    ├── index.ts          # Export principal
    ├── FileReadTool.ts   # Implementation
    └── schema.ts         # Schema JSON (optionnel)
```

### Exemple complet : FileReadTool

```typescript
import { Tool, ToolResult, ToolUseContext } from '../Tool'
import { readFile } from 'fs/promises'

type FileReadInput = {
  file_path: string
  offset?: number
  limit?: number
}

const FileReadTool: Tool = {
  name: 'Read',
  description: 'Read a file from the local filesystem.',
  inputSchema: {
    type: 'object',
    properties: {
      file_path: {
        type: 'string',
        description: 'Absolute path to the file to read'
      },
      offset: {
        type: 'number',
        description: 'Line number to start reading from'
      },
      limit: {
        type: 'number',
        description: 'Number of lines to read'
      }
    },
    required: ['file_path']
  },

  async execute(
    input: FileReadInput,
    context: ToolUseContext
  ): Promise<ToolResult> {
    const { file_path, offset = 0, limit = 2000 } = input

    // Validation du chemin
    if (!isAbsolutePath(file_path)) {
      return { content: 'Error: path must be absolute', isError: true }
    }

    // Verification de traversal
    if (hasPathTraversal(file_path)) {
      return { content: 'Error: path traversal detected', isError: true }
    }

    try {
      const content = await readFile(file_path, 'utf-8')
      const lines = content.split('\n')
      const slice = lines.slice(offset, offset + limit)

      // Format cat -n (avec numeros de ligne)
      const numbered = slice.map(
        (line, i) => `${offset + i + 1}\t${line}`
      ).join('\n')

      return { content: numbered }
    } catch (error) {
      return {
        content: `Error reading file: ${error.message}`,
        isError: true
      }
    }
  }
}

export default FileReadTool
```

## Pour recreer le systeme d'outils

### 1. Definir l'interface Tool

```typescript
interface Tool {
  name: string
  description: string
  inputSchema: JSONSchema
  execute(input: unknown, ctx: Context): Promise<ToolResult>
}
```

### 2. Creer un registre

```typescript
class ToolRegistry {
  private tools = new Map<string, Tool>()

  register(tool: Tool): void {
    this.tools.set(tool.name, tool)
  }

  get(name: string): Tool | undefined {
    return this.tools.get(name)
  }

  getAll(): Tool[] {
    return Array.from(this.tools.values())
  }

  getDefinitions(): ToolDefinition[] {
    return this.getAll().map(t => ({
      name: t.name,
      description: t.description,
      input_schema: t.inputSchema
    }))
  }
}
```

### 3. Implementer l'executeur

```typescript
async function executeTool(
  registry: ToolRegistry,
  call: ToolCall,
  ctx: Context
): Promise<ToolResultBlock> {
  const tool = registry.get(call.name)
  if (!tool) {
    return {
      type: 'tool_result',
      tool_use_id: call.id,
      content: `Unknown tool: ${call.name}`,
      is_error: true
    }
  }

  const result = await tool.execute(call.input, ctx)
  return {
    type: 'tool_result',
    tool_use_id: call.id,
    content: typeof result === 'string' ? result : result.content,
    is_error: result.isError ?? false
  }
}
```

### 4. Integrer dans la boucle de requete

Les resultats des outils sont retournes a l'API comme messages `user` avec le type `tool_result`, creant la boucle de conversation outil.
