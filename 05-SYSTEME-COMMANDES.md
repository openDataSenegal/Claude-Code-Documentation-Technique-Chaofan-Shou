# 05 - Systeme de commandes

## Vue d'ensemble

Les commandes sont les actions de haut niveau que l'utilisateur peut invoquer via des commandes slash (`/commit`, `/help`, `/review`, etc.) ou via les arguments CLI. Le projet contient **100+ commandes** organisees en modules independants.

## Architecture

```
commands.ts                # Registre central, import et filtrage
    │
commands/                  # Implementations (103 modules)
    ├── commit.ts
    ├── review.ts
    ├── init.ts
    ├── help.ts
    ├── login.ts
    └── ...
```

## Registre : `commands.ts`

Le registre central importe toutes les commandes, les filtre par disponibilite et les expose au systeme.

```typescript
// Structure simplifiee
type Command = {
  name: string
  aliases?: string[]
  description: string
  isAvailable: (context: CommandContext) => boolean
  action: (args: CommandArgs, context: CommandContext) => Promise<void>
}

function getAvailableCommands(context: CommandContext): Command[] {
  return ALL_COMMANDS.filter(cmd => cmd.isAvailable(context))
}
```

## Categories de commandes

### Commandes fondamentales (toujours disponibles)

| Commande | Description |
|----------|-------------|
| `/commit` | Generation de message de commit par l'IA + git commit |
| `/commit-push-pr` | Workflow complet : commit → push → creation PR |
| `/review` | Revue de code et commentaires sur PR |
| `/init` | Configuration initiale du projet |
| `/login` | Authentification aupres de l'API |
| `/logout` | Deconnexion |
| `/status` | Informations de session et d'environnement |
| `/help` | Reference des commandes |
| `/config` | Modification des parametres |
| `/context` | Gestion des fichiers de contexte attaches |
| `/tasks` | Gestion des taches et de la memoire de session |
| `/memory` | Gestion de la memoire persistante |
| `/cost` | Rapport d'utilisation et de couts |
| `/resume` | Reprendre une session precedente |
| `/clear` | Effacer la conversation |
| `/compact` | Compacter l'historique des messages |

### Commandes conditionnelles (feature flags)

| Commande | Gate | Description |
|----------|------|-------------|
| `/proactive` | PROACTIVE/KAIROS | Assistant toujours actif |
| `/brief` | KAIROS_BRIEF | Sortie concise en mode KAIROS |
| `/assistant` | KAIROS | Gestion de session assistant |
| `/bridge` | BRIDGE_MODE | Controle distant via claude.ai |
| `/voice` | VOICE_MODE | Entree vocale |
| `/workflows` | WORKFLOW_SCRIPTS | Scripts d'automatisation |
| `/force-snip` | HISTORY_SNIP | Forcer la compression des messages |

### Commandes internes (USER_TYPE === 'ant')

| Commande | Description |
|----------|-------------|
| `/agents-platform` | Interface de gestion des agents |
| `/security-review` | Analyse de menaces |
| `/bughunter` | Tests de securite automatises |

## Implementation d'une commande : exemple

### `/commit` - Generation de commit

```typescript
// commands/commit.ts (structure simplifiee)
const commitCommand: Command = {
  name: 'commit',
  aliases: ['c'],
  description: 'Generate a commit message and create a git commit',

  isAvailable: () => true,

  async action(args, context) {
    // 1. Obtenir le diff git
    const diff = await execGit('diff --staged')
    if (!diff) {
      context.log('No staged changes. Stage files first with git add.')
      return
    }

    // 2. Obtenir les commits recents pour le style
    const recentCommits = await execGit('log --oneline -10')

    // 3. Demander a l'IA de generer le message
    const message = await context.query({
      prompt: `Generate a commit message for these changes.
        Diff: ${diff}
        Recent commits for style reference: ${recentCommits}`,
      maxTokens: 500
    })

    // 4. Creer le commit
    await execGit(`commit -m "${escapeMessage(message)}"`)
    context.log(`Committed: ${message}`)
  }
}
```

## Flux de routage des commandes

```
Entree utilisateur : "/commit -m 'fix bug'"
    │
    ▼
Parse du prefixe slash
    ├── Nom de commande : "commit"
    └── Arguments : ["-m", "fix bug"]
    │
    ▼
Recherche dans le registre
    ├── Match exact par nom → commit
    ├── Match par alias → c → commit
    └── Pas de match → traiter comme message normal
    │
    ▼
Verification de disponibilite
    ├── isAvailable(context) → true/false
    └── Si false → "Command not available"
    │
    ▼
Execution
    └── command.action(parsedArgs, context)
```

## Extensibilite

### Skills comme commandes

Le systeme de **skills** (`/skills/`) etend les commandes avec des capacites supplementaires :

```typescript
// Un skill est une commande enrichie avec un prompt
type Skill = {
  name: string
  trigger: string       // Condition de declenchement
  prompt: string        // Prompt injecte dans la conversation
  tools?: string[]      // Outils requis
}
```

Skills integres :
- `commit` - Workflow de commit
- `review-pr` - Revue de PR
- `simplify` - Simplification du code
- `claude-api` - Aide a l'utilisation de l'API Claude
- `update-config` - Configuration du harness
- `keybindings-help` - Aide sur les raccourcis

### Plugins

Le dossier `/plugins/` permet d'ajouter des commandes externes via un systeme de plugins.

## Pour recreer ce systeme

### 1. Interface Command

```typescript
interface Command {
  name: string
  aliases?: string[]
  description: string
  isAvailable(ctx: Context): boolean
  execute(args: string[], ctx: Context): Promise<void>
}
```

### 2. Registre avec resolution

```typescript
class CommandRegistry {
  private commands = new Map<string, Command>()

  register(cmd: Command): void {
    this.commands.set(cmd.name, cmd)
    cmd.aliases?.forEach(a => this.commands.set(a, cmd))
  }

  resolve(input: string): { command: Command; args: string[] } | null {
    if (!input.startsWith('/')) return null
    const [name, ...args] = input.slice(1).split(' ')
    const cmd = this.commands.get(name)
    return cmd ? { command: cmd, args } : null
  }
}
```

### 3. Integration dans la boucle REPL

```typescript
async function handleInput(input: string, ctx: Context): Promise<void> {
  const resolved = commandRegistry.resolve(input)

  if (resolved) {
    if (!resolved.command.isAvailable(ctx)) {
      console.log('Command not available')
      return
    }
    await resolved.command.execute(resolved.args, ctx)
  } else {
    // Traiter comme un message pour l'IA
    await sendToAI(input, ctx)
  }
}
```

### Points cles pour la recreation

1. **Separation des concerns** : Chaque commande est un module independant
2. **Filtrage dynamique** : Les commandes s'activent/desactivent selon le contexte
3. **Alias** : Supporter les raccourcis (ex: `/c` pour `/commit`)
4. **Feature gates** : Certaines commandes ne sont disponibles que sous certaines conditions
5. **Skills** : Permettre l'extension par des prompts plutot que du code
