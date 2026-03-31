# 14 - Configuration et feature flags

## Vue d'ensemble

Le systeme de configuration de Claude Code opere a plusieurs niveaux : configuration utilisateur, configuration projet, variables d'environnement, MDM (gestion d'appareils), et feature flags runtime.

## Niveaux de configuration

```
Priorite (du plus haut au plus bas) :

1. Arguments CLI        (--model opus)
2. Variables d'env      (ANTHROPIC_API_KEY)
3. Config projet        (.claude.json dans le repo)
4. Config utilisateur   (~/.claude/config.json)
5. MDM                  (policies macOS gerees)
6. Defaults             (valeurs par defaut du code)
```

## Configuration utilisateur : `~/.claude/config.json`

```json
{
  "model": "claude-sonnet-4-6",
  "apiKey": "sk-ant-...",
  "theme": "dark",
  "maxTokens": 16384,
  "thinking": {
    "enabled": true,
    "budgetTokens": 10000
  },
  "permissions": {
    "defaultMode": "default",
    "trustedDirectories": ["/home/user/projects"]
  },
  "hooks": {
    "pre-tool-use": [],
    "post-tool-use": []
  },
  "editor": "code",
  "shell": "/bin/zsh"
}
```

## Configuration projet : `.claude.json`

```json
{
  "tools": {
    "allowed": ["FileRead", "FileEdit", "Glob", "Grep", "Bash"],
    "denied": ["PowerShell"]
  },
  "context": {
    "include": ["CLAUDE.md", "docs/architecture.md"],
    "exclude": ["node_modules", ".env"]
  },
  "memory": {
    "directory": ".claude/memory"
  }
}
```

## CLAUDE.md : Instructions projet

Le fichier `CLAUDE.md` (a la racine du repo) contient des instructions en langage naturel :

```markdown
# CLAUDE.md

## Stack technique
- TypeScript + React
- PostgreSQL + Prisma ORM
- Tests avec Vitest

## Conventions
- Nommage camelCase pour les variables
- PascalCase pour les composants React
- Pas de `any` en TypeScript

## Regles
- Toujours ecrire des tests pour les nouvelles fonctions
- Ne pas modifier les migrations existantes
```

Ce fichier est injecte dans le prompt systeme a chaque requete.

## Variables d'environnement

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Cle API Anthropic |
| `CLAUDE_CODE_MODEL` | Override du modele |
| `CLAUDE_CODE_DEBUG` | Active les logs debug |
| `CLAUDE_CODE_BRIDGE` | Active le mode bridge |
| `CLAUDE_CODE_REMOTE` | Mode conteneur distant |
| `CLAUDE_CODE_MAX_TOKENS` | Limite de tokens |
| `CLAUDE_CODE_THEME` | Theme (dark/light/auto) |
| `HTTP_PROXY` / `HTTPS_PROXY` | Configuration proxy |

## Feature Flags (compile-time)

### Mecanisme Bun

Les feature flags sont resolus a la compilation par Bun, permettant le dead-code elimination :

```typescript
// A la compilation, Bun remplace feature('KAIROS') par true ou false
// Le bundler elimine ensuite le code mort
if (feature('KAIROS')) {
  const assistant = require('./assistant')
  // Ce code est SUPPRIME du bundle si KAIROS = false
}
```

### Liste des feature flags

| Flag | Description | Build externe |
|------|-------------|---------------|
| `PROACTIVE` / `KAIROS` | Mode assistant toujours actif | OFF |
| `KAIROS_BRIEF` | Commande brief | OFF |
| `BRIDGE_MODE` | Controle distant | OFF |
| `DAEMON` | Workers daemon | OFF |
| `VOICE_MODE` | Entree vocale | OFF |
| `WORKFLOW_SCRIPTS` | Scripts d'automatisation | OFF |
| `COORDINATOR_MODE` | Orchestration multi-agent | OFF |
| `TRANSCRIPT_CLASSIFIER` | Mode AFK auto-approval | OFF |
| `BUDDY` | Compagnon Tamagotchi | OFF |
| `NATIVE_CLIENT_ATTESTATION` | Verification client | OFF |
| `HISTORY_SNIP` | Compression avancee | OFF |
| `EXPERIMENTAL_SKILL_SEARCH` | Decouverte de skills | OFF |
| `DUMP_SYSTEM_PROMPT` | Extraction du prompt | ON |
| `CHICAGO_MCP` | Computer Use | OFF |
| `REACTIVE_COMPACT` | Compaction avancee | OFF |
| `CONTEXT_COLLAPSE` | Optimisation contexte | OFF |

### Builds externe vs interne

| Aspect | Build externe | Build interne (ant) |
|--------|--------------|---------------------|
| Feature flags | Minimal | Tous actifs |
| Undercover mode | Actif | Desactive |
| Outils | Standard | Standard + internes |
| Commandes | Standard | Standard + internes |
| API staging | Non | Oui |
| Debug modes | Non | Oui |

## Feature Flags runtime (GrowthBook)

### Principe

GrowthBook permet de gater des fonctionnalites a runtime sans redeployer :

```typescript
import { GrowthBook } from '@growthbook/growthbook'

const gb = new GrowthBook({
  apiHost: 'https://cdn.growthbook.io',
  clientKey: 'sdk-...',
  attributes: {
    userId: getUserId(),
    plan: getSubscriptionPlan(),
    version: getVersion()
  }
})

// Verification a runtime
if (gb.isOn('new_compaction_algorithm')) {
  useNewCompaction()
} else {
  useLegacyCompaction()
}

// Feature avec valeur
const maxAgents = gb.getFeatureValue('max_parallel_agents', 3)
```

### Cache stale-acceptable

Pour ne pas bloquer le demarrage :

```typescript
async function loadFeatureFlags(): Promise<GrowthBook> {
  const cached = readCachedFlags()

  // Utiliser le cache immediatement
  const gb = new GrowthBook({ features: cached })

  // Rafraichir en arriere-plan
  refreshFlagsAsync().then(fresh => {
    gb.setFeatures(fresh)
    writeCachedFlags(fresh)
  })

  return gb
}
```

## MDM (Managed Device Management)

Sur macOS, les organisations peuvent imposer des configurations via MDM :

```typescript
// Lecture des settings MDM (via plist)
async function readMDMSettings(): Promise<MDMSettings> {
  const result = await exec(
    'defaults read com.anthropic.claude-code'
  )
  return parsePlist(result)
}
```

Parametres MDM supportes :
- Modele impose
- API endpoint personnalise
- Outils interdits
- Mode de permission force
- Politique de retention des logs

## Migrations : `migrations/`

Le systeme de migrations gere l'evolution des schemas de configuration :

```
migrations/
├── 001_initial.ts
├── 002_add_thinking_config.ts
├── 003_rename_model_field.ts
└── ...
```

```typescript
// Exemple de migration
const migration002 = {
  version: 2,
  up(config: ConfigV1): ConfigV2 {
    return {
      ...config,
      thinking: {
        enabled: false,
        budgetTokens: 10000
      }
    }
  }
}
```

## Pour recreer ce systeme

### 1. Configuration multi-niveaux

```typescript
class Config {
  private layers: ConfigLayer[] = []

  addLayer(name: string, values: Record<string, unknown>): void {
    this.layers.push({ name, values })
  }

  get<T>(key: string, defaultValue: T): T {
    // Chercher du plus prioritaire au moins prioritaire
    for (const layer of this.layers.reverse()) {
      if (key in layer.values) return layer.values[key] as T
    }
    return defaultValue
  }
}

// Initialisation
const config = new Config()
config.addLayer('defaults', DEFAULTS)
config.addLayer('user', loadUserConfig())
config.addLayer('project', loadProjectConfig())
config.addLayer('env', parseEnvVars())
config.addLayer('cli', parseCLIArgs())
```

### 2. Feature flags

```typescript
class FeatureFlags {
  private flags: Record<string, boolean>

  constructor(flags: Record<string, boolean>) {
    this.flags = flags
  }

  isEnabled(flag: string): boolean {
    return this.flags[flag] ?? false
  }

  // Pour le dead-code elimination a la compilation
  static compileTime(flag: string): boolean {
    // Replace par le bundler
    return process.env[`FEATURE_${flag}`] === 'true'
  }
}
```

### 3. CLAUDE.md loader

```typescript
async function loadClaudeMd(projectDir: string): Promise<string | null> {
  const candidates = [
    path.join(projectDir, 'CLAUDE.md'),
    path.join(projectDir, '.claude', 'CLAUDE.md'),
  ]

  for (const candidate of candidates) {
    try {
      return await readFile(candidate, 'utf-8')
    } catch {
      continue
    }
  }

  return null
}
```

### Points cles

1. **Cascade de priorite** : CLI > env > projet > utilisateur > defaults
2. **Feature flags a deux niveaux** : Compile-time (DCE) + runtime (GrowthBook)
3. **CLAUDE.md** : Instructions en langage naturel, injectees dans le prompt
4. **Migrations** : Evolution propre des schemas de configuration
5. **MDM** : Support entreprise pour la gestion centralisee
