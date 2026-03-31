# 08 - Systeme de permissions

## Vue d'ensemble

Le systeme de permissions controle quels outils l'IA peut executer et dans quelles conditions. C'est un composant critique de securite qui protege l'utilisateur contre les actions non desirees.

## Architecture

```
tools/permissions/       # Definitions et regles
hooks/useCanUseTool.tsx  # Hook de decision
services/tools/          # Integration dans l'execution
```

## Niveaux de risque

Chaque outil est classifie par niveau de risque :

| Niveau | Description | Exemples |
|--------|-------------|----------|
| **LOW** | Operations en lecture seule, sans effet de bord | FileRead, Glob, Grep, WebSearch |
| **MEDIUM** | Creation/modification de fichiers, installations | FileWrite, FileEdit, NotebookEdit |
| **HIGH** | Operations systeme, shell, git destructif | Bash, PowerShell, operations git |

## Modes de permission

### Mode `default` (standard)

L'utilisateur est prompt pour chaque action a risque MEDIUM ou HIGH :

```
┌────────────────────────────────────────┐
│ Claude wants to run: bash              │
│ Command: npm install express           │
│                                        │
│ [Allow] [Allow for session] [Deny]     │
└────────────────────────────────────────┘
```

### Mode `auto` (approbation ML)

Un classificateur ML evalue automatiquement les actions :
- Actions jugees sures → auto-approuvees
- Actions douteuses → prompt utilisateur
- Actions dangereuses → toujours prompt

### Mode `bypass`

Toutes les actions sont auto-approuvees. Reserve aux environnements controles (CI/CD, tests).

## Flux de decision

```
Requete d'outil
    │
    ▼
Classification du risque
    │
    ├── LOW → Auto-approuve
    │
    ├── MEDIUM / HIGH
    │   │
    │   ▼
    │   Fichier protege ?
    │   ├── Oui → DENY (auto)
    │   │
    │   ├── Non → Verifier le mode
    │   │   │
    │   │   ├── bypass → APPROVE
    │   │   │
    │   │   ├── auto → Classificateur ML
    │   │   │   ├── Sur → APPROVE
    │   │   │   ├── Incertain → ASK_USER
    │   │   │   └── Dangereux → ASK_USER
    │   │   │
    │   │   └── default → ASK_USER
    │   │
    │   ▼
    │   [Permission en cache pour cette session ?]
    │   ├── Oui → utiliser la decision en cache
    │   └── Non → prompt utilisateur
    │
    ▼
Decision finale : APPROVE | DENY
```

## Fichiers proteges

Certains fichiers sont automatiquement proteges contre l'edition :

```typescript
const PROTECTED_FILES = [
  '.gitconfig',
  '.bashrc',
  '.zshrc',
  '.bash_profile',
  '.zprofile',
  '.mcp.json',
  '.claude.json',
  '.ssh/*',
  'credentials.json',
  '.env',
  '.env.*',
]
```

Toute tentative d'edition de ces fichiers est automatiquement refusee avec un message explicatif.

## Permission Explainer

Quand l'utilisateur est prompt, le systeme genere une explication du risque :

```typescript
// Appel LLM separe pour expliquer le risque
async function explainPermission(toolCall: ToolCall): Promise<string> {
  return await quickQuery({
    prompt: `Explain in 1-2 sentences why this tool call
      might be risky: ${JSON.stringify(toolCall)}`,
    maxTokens: 100
  })
}
```

Exemple de sortie :
> "This command will install a package from npm. It will modify your
> node_modules directory and package.json file."

## Cache de permissions

Les decisions utilisateur sont cachees pour la session :

```typescript
type PermissionCache = {
  // Permission pour un outil specifique
  toolLevel: Map<string, 'allow' | 'deny'>
  // Permission pour un outil + pattern d'argument
  patternLevel: Map<string, 'allow' | 'deny'>
}

// Exemples de cache
{
  toolLevel: {
    'Bash': 'allow'  // "Allow for session"
  },
  patternLevel: {
    'FileWrite:/src/*': 'allow'  // "Allow writes to /src/"
  }
}
```

## Validation des inputs

Avant l'execution, les inputs sont valides :

### Prevention de traversal de chemin

```typescript
function hasPathTraversal(path: string): boolean {
  // URL-encoded traversal
  if (path.includes('%2e') || path.includes('%2E')) return true
  // Unicode normalization
  if (path !== path.normalize('NFC')) return true
  // Double dots
  if (path.includes('..')) return true
  // Backslash injection (Windows)
  if (path.includes('\\..')) return true
  return false
}
```

### Validation des commandes shell

```typescript
function validateBashCommand(command: string): ValidationResult {
  // Detecter les commandes dangereuses
  const dangerous = [
    /rm\s+-rf\s+\//,           // rm -rf /
    /mkfs/,                     // format disk
    /dd\s+if=.*of=\/dev/,      // disk destruction
    /:(){ :|:& };:/,           // fork bomb
  ]

  for (const pattern of dangerous) {
    if (pattern.test(command)) {
      return { valid: false, reason: 'Dangerous command detected' }
    }
  }

  return { valid: true }
}
```

## Classificateur YOLO (auto-approval ML)

Quand active, un modele ML leger evalue les appels d'outils :

```
Input: {
  tool: "Bash",
  command: "git status",
  context: { workingDir: "/project", recentActions: [...] }
}

Output: {
  decision: "approve",  // approve | deny | ask_user
  confidence: 0.95,
  reasoning: "Read-only git command, no side effects"
}
```

Le classificateur est entraine sur des patterns courants de developpement.

## Pour recreer ce systeme

### 1. Classification des outils

```typescript
enum RiskLevel { LOW, MEDIUM, HIGH }

const TOOL_RISK: Record<string, RiskLevel> = {
  'FileRead': RiskLevel.LOW,
  'Glob': RiskLevel.LOW,
  'Grep': RiskLevel.LOW,
  'FileWrite': RiskLevel.MEDIUM,
  'FileEdit': RiskLevel.MEDIUM,
  'Bash': RiskLevel.HIGH,
}
```

### 2. Systeme de permission

```typescript
class PermissionSystem {
  private cache = new Map<string, boolean>()

  async checkPermission(
    tool: string,
    input: unknown
  ): Promise<'approve' | 'deny' | 'ask'> {
    const risk = TOOL_RISK[tool] ?? RiskLevel.HIGH

    if (risk === RiskLevel.LOW) return 'approve'

    if (this.isProtectedTarget(tool, input)) return 'deny'

    const cacheKey = this.getCacheKey(tool, input)
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)! ? 'approve' : 'deny'
    }

    return 'ask'
  }

  grant(tool: string, scope: 'once' | 'session'): void {
    if (scope === 'session') {
      this.cache.set(tool, true)
    }
  }

  deny(tool: string): void {
    this.cache.set(tool, false)
  }

  private isProtectedTarget(tool: string, input: unknown): boolean {
    if (tool === 'FileWrite' || tool === 'FileEdit') {
      const path = (input as { file_path: string }).file_path
      return PROTECTED_FILES.some(p => matchGlob(path, p))
    }
    return false
  }
}
```

### 3. Dialogue de permission (terminal)

```typescript
async function promptPermission(
  tool: string,
  input: unknown
): Promise<'approve' | 'approve_session' | 'deny'> {
  console.log(`\nClaude wants to use: ${tool}`)
  console.log(`Input: ${JSON.stringify(input, null, 2)}`)
  console.log('\n[a] Allow  [s] Allow for session  [d] Deny')

  const key = await readKey()
  switch (key) {
    case 'a': return 'approve'
    case 's': return 'approve_session'
    case 'd': return 'deny'
    default: return 'deny'
  }
}
```

### Points cles

1. **Defense en profondeur** : Plusieurs couches de validation
2. **Principe du moindre privilege** : Seuls les outils necessaires sont autorises
3. **Transparence** : L'utilisateur voit exactement ce qui sera execute
4. **Cache de session** : Eviter de demander la meme chose a repetition
5. **Fichiers proteges** : Liste noire des fichiers sensibles
