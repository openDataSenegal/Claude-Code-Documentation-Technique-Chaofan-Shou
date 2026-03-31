# 16 - Guide de recreation

## Objectif

Ce guide permet de recreer un projet similaire a Claude Code : un CLI interactif connecte a une API LLM, avec execution d'outils, interface terminal riche, et systeme de memoire persistante.

## Phase 1 : Fondations (Semaine 1-2)

### 1.1 Initialisation du projet

```bash
# Initialiser avec Bun (ou Node.js + TypeScript)
mkdir my-ai-cli
cd my-ai-cli
bun init

# Installer les dependances essentielles
bun add @anthropic-ai/sdk     # Client API
bun add ink react              # UI terminal (ou implementation custom)
bun add commander              # Parsing CLI
bun add chalk                  # Couleurs terminal
```

### 1.2 Structure de base

```
my-ai-cli/
├── src/
│   ├── index.ts              # Point d'entree CLI
│   ├── repl.tsx              # Interface REPL
│   ├── query-engine.ts       # Moteur de requetes
│   ├── tools/                # Implementations d'outils
│   │   ├── index.ts          # Registre
│   │   ├── types.ts          # Interface Tool
│   │   ├── file-read.ts
│   │   ├── file-write.ts
│   │   ├── file-edit.ts
│   │   ├── bash.ts
│   │   ├── glob.ts
│   │   └── grep.ts
│   ├── commands/             # Commandes slash
│   │   ├── index.ts
│   │   ├── help.ts
│   │   ├── clear.ts
│   │   └── commit.ts
│   ├── state/                # Gestion d'etat
│   │   └── store.ts
│   ├── permissions/          # Systeme de permissions
│   │   └── index.ts
│   ├── config/               # Configuration
│   │   └── index.ts
│   └── utils/                # Utilitaires
│       ├── path-validation.ts
│       └── secret-detection.ts
├── tsconfig.json
├── package.json
└── CLAUDE.md
```

### 1.3 Point d'entree CLI

```typescript
// src/index.ts
import { Command } from 'commander'
import { startREPL } from './repl'
import { loadConfig } from './config'

const program = new Command()
  .name('my-ai-cli')
  .version('1.0.0')
  .description('AI-powered CLI assistant')
  .argument('[query]', 'Initial query')
  .option('-m, --model <model>', 'Model to use', 'claude-sonnet-4-6')
  .option('--resume <session>', 'Resume a previous session')
  .action(async (query, options) => {
    const config = await loadConfig()
    await startREPL({ ...config, ...options, initialQuery: query })
  })

program.parse()
```

## Phase 2 : Moteur de requetes (Semaine 2-3)

### 2.1 Interface Tool

```typescript
// src/tools/types.ts
export interface ToolDefinition {
  name: string
  description: string
  input_schema: {
    type: 'object'
    properties: Record<string, unknown>
    required?: string[]
  }
}

export interface ToolResult {
  content: string
  isError?: boolean
}

export interface Tool {
  definition: ToolDefinition
  execute(input: Record<string, unknown>): Promise<ToolResult>
}
```

### 2.2 Query Engine

```typescript
// src/query-engine.ts
import Anthropic from '@anthropic-ai/sdk'
import type { Tool, ToolResult } from './tools/types'

export class QueryEngine {
  private client: Anthropic
  private tools: Tool[]
  private messages: Anthropic.MessageParam[] = []

  constructor(apiKey: string, tools: Tool[]) {
    this.client = new Anthropic({ apiKey })
    this.tools = tools
  }

  async query(
    userMessage: string,
    systemPrompt: string,
    options: {
      model?: string
      maxTokens?: number
      onText?: (text: string) => void
      onToolUse?: (name: string, input: unknown) => void
      onToolResult?: (name: string, result: ToolResult) => void
      checkPermission?: (name: string, input: unknown) => Promise<boolean>
    } = {}
  ): Promise<string> {
    const {
      model = 'claude-sonnet-4-6',
      maxTokens = 8192,
      onText,
      onToolUse,
      onToolResult,
      checkPermission
    } = options

    this.messages.push({ role: 'user', content: userMessage })

    while (true) {
      // Appel API
      const response = await this.client.messages.create({
        model,
        max_tokens: maxTokens,
        system: systemPrompt,
        messages: this.messages,
        tools: this.tools.map(t => t.definition)
      })

      this.messages.push({ role: 'assistant', content: response.content })

      // Extraire le texte
      let responseText = ''
      const toolCalls: Array<{ id: string; name: string; input: unknown }> = []

      for (const block of response.content) {
        if (block.type === 'text') {
          responseText += block.text
          onText?.(block.text)
        } else if (block.type === 'tool_use') {
          toolCalls.push({
            id: block.id,
            name: block.name,
            input: block.input
          })
        }
      }

      // Si pas d'appels d'outils, on a fini
      if (toolCalls.length === 0) {
        return responseText
      }

      // Executer les outils
      const toolResults: Anthropic.ToolResultBlockParam[] = []

      for (const call of toolCalls) {
        onToolUse?.(call.name, call.input)

        // Verifier la permission
        if (checkPermission) {
          const allowed = await checkPermission(call.name, call.input)
          if (!allowed) {
            toolResults.push({
              type: 'tool_result',
              tool_use_id: call.id,
              content: 'Permission denied by user',
              is_error: true
            })
            continue
          }
        }

        // Executer l'outil
        const tool = this.tools.find(t => t.definition.name === call.name)
        if (!tool) {
          toolResults.push({
            type: 'tool_result',
            tool_use_id: call.id,
            content: `Unknown tool: ${call.name}`,
            is_error: true
          })
          continue
        }

        try {
          const result = await tool.execute(call.input as Record<string, unknown>)
          onToolResult?.(call.name, result)
          toolResults.push({
            type: 'tool_result',
            tool_use_id: call.id,
            content: result.content,
            is_error: result.isError
          })
        } catch (error) {
          toolResults.push({
            type: 'tool_result',
            tool_use_id: call.id,
            content: `Error: ${(error as Error).message}`,
            is_error: true
          })
        }
      }

      // Ajouter les resultats et continuer la boucle
      this.messages.push({ role: 'user', content: toolResults })
    }
  }

  getMessages(): Anthropic.MessageParam[] {
    return [...this.messages]
  }

  setMessages(messages: Anthropic.MessageParam[]): void {
    this.messages = messages
  }
}
```

## Phase 3 : Outils de base (Semaine 3-4)

### 3.1 FileRead

```typescript
// src/tools/file-read.ts
import { readFile } from 'fs/promises'
import { resolve, isAbsolute } from 'path'
import type { Tool } from './types'

export const fileReadTool: Tool = {
  definition: {
    name: 'Read',
    description: 'Read a file from the filesystem',
    input_schema: {
      type: 'object',
      properties: {
        file_path: { type: 'string', description: 'Absolute file path' },
        offset: { type: 'number', description: 'Start line (0-based)' },
        limit: { type: 'number', description: 'Max lines to read' }
      },
      required: ['file_path']
    }
  },

  async execute(input) {
    const filePath = input.file_path as string
    const offset = (input.offset as number) ?? 0
    const limit = (input.limit as number) ?? 2000

    if (!isAbsolute(filePath)) {
      return { content: 'Error: path must be absolute', isError: true }
    }

    try {
      const content = await readFile(filePath, 'utf-8')
      const lines = content.split('\n')
      const slice = lines.slice(offset, offset + limit)
      const numbered = slice
        .map((line, i) => `${offset + i + 1}\t${line}`)
        .join('\n')
      return { content: numbered }
    } catch (err) {
      return { content: `Error: ${(err as Error).message}`, isError: true }
    }
  }
}
```

### 3.2 Bash

```typescript
// src/tools/bash.ts
import { exec } from 'child_process'
import { promisify } from 'util'
import type { Tool } from './types'

const execAsync = promisify(exec)

export const bashTool: Tool = {
  definition: {
    name: 'Bash',
    description: 'Execute a shell command',
    input_schema: {
      type: 'object',
      properties: {
        command: { type: 'string', description: 'Shell command to execute' },
        timeout: { type: 'number', description: 'Timeout in ms (max 120000)' }
      },
      required: ['command']
    }
  },

  async execute(input) {
    const command = input.command as string
    const timeout = Math.min((input.timeout as number) ?? 120000, 120000)

    try {
      const { stdout, stderr } = await execAsync(command, {
        timeout,
        cwd: process.cwd(),
        maxBuffer: 1024 * 1024 * 10  // 10MB
      })
      const output = [stdout, stderr].filter(Boolean).join('\n')
      return { content: output || '(no output)' }
    } catch (err) {
      return { content: `Error: ${(err as Error).message}`, isError: true }
    }
  }
}
```

### 3.3 FileEdit

```typescript
// src/tools/file-edit.ts
import { readFile, writeFile } from 'fs/promises'
import type { Tool } from './types'

export const fileEditTool: Tool = {
  definition: {
    name: 'Edit',
    description: 'Replace a string in a file',
    input_schema: {
      type: 'object',
      properties: {
        file_path: { type: 'string', description: 'Absolute file path' },
        old_string: { type: 'string', description: 'Text to find' },
        new_string: { type: 'string', description: 'Replacement text' }
      },
      required: ['file_path', 'old_string', 'new_string']
    }
  },

  async execute(input) {
    const filePath = input.file_path as string
    const oldStr = input.old_string as string
    const newStr = input.new_string as string

    try {
      const content = await readFile(filePath, 'utf-8')

      const occurrences = content.split(oldStr).length - 1
      if (occurrences === 0) {
        return { content: 'Error: old_string not found in file', isError: true }
      }
      if (occurrences > 1) {
        return {
          content: `Error: old_string found ${occurrences} times. Must be unique.`,
          isError: true
        }
      }

      const newContent = content.replace(oldStr, newStr)
      await writeFile(filePath, newContent)
      return { content: 'File edited successfully' }
    } catch (err) {
      return { content: `Error: ${(err as Error).message}`, isError: true }
    }
  }
}
```

### 3.4 Registre d'outils

```typescript
// src/tools/index.ts
import { fileReadTool } from './file-read'
import { fileEditTool } from './file-edit'
import { bashTool } from './bash'
// ... autres imports

export const ALL_TOOLS = [
  fileReadTool,
  fileEditTool,
  bashTool,
  // globTool, grepTool, fileWriteTool, ...
]
```

## Phase 4 : Interface REPL (Semaine 4-5)

### 4.1 REPL avec Ink

```tsx
// src/repl.tsx
import React, { useState, useCallback } from 'react'
import { render, Box, Text, useInput } from 'ink'
import TextInput from 'ink-text-input'
import { QueryEngine } from './query-engine'
import { ALL_TOOLS } from './tools'
import { buildSystemPrompt } from './prompt'

type Message = {
  role: 'user' | 'assistant' | 'tool'
  content: string
}

function REPL({ config }: { config: Config }) {
  const [messages, setMessages] = useState<Message[]>([])
  const [input, setInput] = useState('')
  const [isLoading, setIsLoading] = useState(false)

  const engine = new QueryEngine(config.apiKey, ALL_TOOLS)

  const handleSubmit = useCallback(async (value: string) => {
    if (!value.trim() || isLoading) return

    setInput('')
    setMessages(prev => [...prev, { role: 'user', content: value }])
    setIsLoading(true)

    try {
      const systemPrompt = await buildSystemPrompt()

      const response = await engine.query(value, systemPrompt, {
        model: config.model,
        onText: (text) => {
          setMessages(prev => {
            const last = prev[prev.length - 1]
            if (last?.role === 'assistant') {
              return [
                ...prev.slice(0, -1),
                { ...last, content: last.content + text }
              ]
            }
            return [...prev, { role: 'assistant', content: text }]
          })
        },
        onToolUse: (name, input) => {
          setMessages(prev => [
            ...prev,
            { role: 'tool', content: `Using ${name}...` }
          ])
        },
        checkPermission: async (name, input) => {
          // Implementer le dialogue de permission ici
          return true  // MVP: tout approuver
        }
      })
    } catch (error) {
      setMessages(prev => [
        ...prev,
        { role: 'assistant', content: `Error: ${(error as Error).message}` }
      ])
    } finally {
      setIsLoading(false)
    }
  }, [isLoading, config])

  return (
    <Box flexDirection="column" padding={1}>
      <Text bold color="blue">My AI CLI</Text>
      <Box flexDirection="column" marginY={1}>
        {messages.map((msg, i) => (
          <Box key={i} marginBottom={1}>
            <Text bold color={msg.role === 'user' ? 'green' : 'cyan'}>
              {msg.role === 'user' ? '> ' : '  '}
            </Text>
            <Text>{msg.content}</Text>
          </Box>
        ))}
      </Box>
      {isLoading && <Text dimColor>Thinking...</Text>}
      <Box>
        <Text bold color="green">{isLoading ? '...' : '>'} </Text>
        <TextInput
          value={input}
          onChange={setInput}
          onSubmit={handleSubmit}
        />
      </Box>
    </Box>
  )
}

export async function startREPL(config: Config) {
  render(<REPL config={config} />)
}
```

## Phase 5 : Prompt systeme (Semaine 5)

### 5.1 Construction du prompt

```typescript
// src/prompt.ts
import { readFile } from 'fs/promises'
import { exec } from 'child_process'
import { promisify } from 'util'

const execAsync = promisify(exec)

export async function buildSystemPrompt(): Promise<string> {
  const parts: string[] = []

  // Section stable (cachable)
  parts.push(ROLE_INSTRUCTIONS)
  parts.push(TOOL_USAGE_RULES)
  parts.push(SECURITY_INSTRUCTIONS)

  // Section dynamique
  parts.push(`Current date: ${new Date().toISOString().split('T')[0]}`)
  parts.push(`Working directory: ${process.cwd()}`)

  // Git status
  try {
    const { stdout } = await execAsync('git status --short')
    parts.push(`Git status:\n${stdout}`)
  } catch {}

  // CLAUDE.md
  try {
    const claudeMd = await readFile('CLAUDE.md', 'utf-8')
    parts.push(`Project instructions (CLAUDE.md):\n${claudeMd}`)
  } catch {}

  // Memoires
  try {
    const memories = await loadMemories()
    if (memories) parts.push(`Memory:\n${memories}`)
  } catch {}

  return parts.join('\n\n')
}

const ROLE_INSTRUCTIONS = `You are an AI coding assistant running in the terminal.
You help developers with software engineering tasks.
You have access to tools for reading files, editing code, running commands, and more.
Always read a file before editing it.
Prefer editing existing files over creating new ones.`

const TOOL_USAGE_RULES = `When using tools:
- Use Read instead of cat/head/tail
- Use Edit instead of sed/awk
- Use Glob instead of find
- Use Grep instead of grep/rg
- Use Bash only for commands that require shell execution`

const SECURITY_INSTRUCTIONS = `Security rules:
- Never execute commands that could damage the system
- Never commit secrets or credentials
- Validate all file paths against traversal attacks
- Ask permission before running potentially dangerous commands`
```

## Phase 6 : Systeme de permissions (Semaine 5-6)

```typescript
// src/permissions/index.ts
const RISK_LEVELS: Record<string, 'low' | 'medium' | 'high'> = {
  'Read': 'low',
  'Glob': 'low',
  'Grep': 'low',
  'Edit': 'medium',
  'Write': 'medium',
  'Bash': 'high',
}

export async function checkPermission(
  toolName: string,
  input: unknown,
  readline: ReadlineInterface
): Promise<boolean> {
  const risk = RISK_LEVELS[toolName] ?? 'high'

  if (risk === 'low') return true

  console.log(`\n⚠️  ${toolName} wants to execute:`)
  console.log(JSON.stringify(input, null, 2))
  console.log(`Risk level: ${risk.toUpperCase()}`)

  const answer = await question(readline, '[a]llow / [d]eny / [s]ession allow: ')

  switch (answer.toLowerCase()) {
    case 'a': return true
    case 's':
      sessionAllowed.add(toolName)
      return true
    default: return false
  }
}
```

## Phase 7 : Memoire persistante (Semaine 6-7)

```typescript
// src/memory/index.ts
import { readFile, writeFile, mkdir } from 'fs/promises'
import { join } from 'path'
import { homedir } from 'os'

const MEMORY_DIR = join(homedir(), '.my-ai-cli', 'memory')

export async function saveMemory(
  name: string,
  type: string,
  description: string,
  content: string
): Promise<void> {
  await mkdir(MEMORY_DIR, { recursive: true })

  const filename = `${type}_${slugify(name)}.md`
  const filepath = join(MEMORY_DIR, filename)

  const file = `---
name: ${name}
description: ${description}
type: ${type}
---

${content}`

  await writeFile(filepath, file)
  await updateIndex(filename, description)
}

export async function loadMemories(): Promise<string> {
  try {
    return await readFile(join(MEMORY_DIR, 'MEMORY.md'), 'utf-8')
  } catch {
    return ''
  }
}

async function updateIndex(filename: string, description: string): Promise<void> {
  const indexPath = join(MEMORY_DIR, 'MEMORY.md')
  let index = ''
  try { index = await readFile(indexPath, 'utf-8') } catch {}

  if (!index.includes(filename)) {
    index += `- [${filename}](${filename}) — ${description}\n`
    await writeFile(indexPath, index)
  }
}
```

## Phase 8 : Commandes slash (Semaine 7)

```typescript
// src/commands/index.ts
import type { QueryEngine } from '../query-engine'

interface Command {
  name: string
  description: string
  execute(args: string[], engine: QueryEngine): Promise<string>
}

const commands: Command[] = [
  {
    name: 'help',
    description: 'Show available commands',
    async execute() {
      return commands.map(c => `/${c.name} - ${c.description}`).join('\n')
    }
  },
  {
    name: 'clear',
    description: 'Clear conversation history',
    async execute(_, engine) {
      engine.setMessages([])
      return 'Conversation cleared'
    }
  },
  {
    name: 'cost',
    description: 'Show session cost',
    async execute(_, engine) {
      // Implementer le calcul des couts
      return 'Cost tracking not yet implemented'
    }
  }
]

export function resolveCommand(input: string): {
  command: Command
  args: string[]
} | null {
  if (!input.startsWith('/')) return null
  const [name, ...args] = input.slice(1).split(' ')
  const cmd = commands.find(c => c.name === name)
  return cmd ? { command: cmd, args } : null
}
```

## Phase 9 : Ameliorations avancees (Semaine 8+)

### 9.1 Streaming des reponses
Remplacer `messages.create` par `messages.stream` pour afficher les reponses en temps reel.

### 9.2 Persistence des sessions
Sauvegarder les messages sur disque pour permettre la reprise avec `--resume`.

### 9.3 Sous-agents
Implementer l'AgentTool pour deleguer des taches a des sous-processus.

### 9.4 Compaction des messages
Implementer la troncature et le resume des anciens messages pour rester dans la fenetre de contexte.

### 9.5 Mode vim
Ajouter le support des motions et operateurs vim pour l'edition.

### 9.6 Bridge WebSocket
Permettre le controle distant via une interface web.

## Checklist de recreation

- [ ] CLI avec parsing d'arguments
- [ ] Connexion a l'API Anthropic
- [ ] Boucle de requete avec tool-use
- [ ] Outils de base (Read, Edit, Write, Bash, Glob, Grep)
- [ ] Interface REPL terminal
- [ ] Prompt systeme avec contexte dynamique
- [ ] Systeme de permissions
- [ ] Memoire persistante
- [ ] Commandes slash
- [ ] Streaming des reponses
- [ ] Persistence des sessions
- [ ] Suivi des couts
- [ ] Sous-agents
- [ ] Gestion de la fenetre de contexte
- [ ] Detection de secrets
- [ ] Validation des chemins
- [ ] Configuration multi-niveaux
- [ ] CLAUDE.md support

## Dependances recommandees

| Package | Usage |
|---------|-------|
| `@anthropic-ai/sdk` | Client API Anthropic |
| `ink` + `react` | Interface terminal React |
| `ink-text-input` | Champ de saisie |
| `commander` | Parsing CLI |
| `chalk` | Couleurs terminal |
| `glob` | Pattern matching de fichiers |
| `chokidar` | Watcher de fichiers (settings) |
| `jsonwebtoken` | JWT pour le bridge |
| `ws` | WebSocket pour le bridge |

## Conseils

1. **Commencer simple** : Un REPL basique avec Read + Bash couvre deja 80% des cas d'usage
2. **Iterer** : Ajouter les fonctionnalites une par une
3. **Tester avec de vrais workflows** : Utiliser l'outil pour de vrais projets des la phase 2
4. **Securite des le debut** : Valider les chemins et detecter les secrets avant d'ajouter des features
5. **Le prompt est le produit** : Investir du temps dans le prompt systeme, c'est ce qui fait la difference
