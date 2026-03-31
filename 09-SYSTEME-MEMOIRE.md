# 09 - Systeme de memoire

## Vue d'ensemble

Claude Code dispose d'un systeme de memoire persistante permettant de retenir des informations entre les sessions. Il comprend deux sous-systemes : la **memoire fichier** (lecture/ecriture manuelle) et le **systeme autoDream** (consolidation automatique).

## Architecture

```
memdir/                          # Systeme de memoire
├── memdir.ts                    # Chargement et parsing
├── memoryTypes.ts               # Types de memoire
├── memoryFile.ts                # Operations sur fichiers memoire
├── memoryIndex.ts               # Gestion de l'index MEMORY.md
└── ...

services/autoDream/              # Consolidation automatique
├── config.ts                    # Configuration des seuils
├── consolidationPrompt.ts       # Prompt de consolidation
├── consolidationLock.ts         # Verrou d'execution
└── ...

~/.claude/projects/<project>/memory/  # Stockage sur disque
├── MEMORY.md                    # Index des memoires
├── user_role.md                 # Memoire utilisateur
├── feedback_testing.md          # Retour utilisateur
├── project_goals.md             # Objectifs projet
└── ...
```

## Types de memoire

```typescript
type MemoryType = 'user' | 'feedback' | 'project' | 'reference'
```

### `user` - Informations sur l'utilisateur
- Role, competences, preferences
- Style de collaboration souhaite
- Niveau d'expertise par domaine

### `feedback` - Retours sur l'approche de travail
- Corrections ("ne fais pas X")
- Validations ("oui, continue comme ca")
- Preferences de workflow

### `project` - Contexte du projet
- Decisions architecturales
- Contraintes et deadlines
- Etat des initiatives en cours

### `reference` - Pointeurs vers des ressources externes
- Liens vers des outils (Linear, Jira, Grafana)
- Documentation externe
- Channels Slack ou contacts

## Format d'un fichier memoire

```markdown
---
name: User Role
description: Senior backend developer focused on Go microservices
type: user
---

User is a senior backend developer with 8 years of experience.
Primary stack: Go, PostgreSQL, Kubernetes.
Prefers concise responses with code examples.
Frame frontend explanations in terms of backend analogues.
```

## Index : `MEMORY.md`

L'index est un fichier Markdown avec des liens vers chaque memoire :

```markdown
- [User Role](user_role.md) — Senior Go backend dev, prefers concise responses
- [Testing Feedback](feedback_testing.md) — Use real DB in integration tests, not mocks
- [Auth Rewrite](project_auth.md) — Driven by legal compliance, not tech debt
- [Bug Tracker](reference_linear.md) — Pipeline bugs tracked in Linear "INGEST"
```

**Limite** : ~200 lignes maximum (les lignes au-dela sont tronquees).

## Cycle de vie de la memoire

### Ecriture (manuelle ou par l'IA)

```
1. L'utilisateur dit quelque chose de memorable
   "Je suis data scientist, je travaille sur l'observabilite"
    │
    ▼
2. L'IA detecte l'information utile
    │
    ▼
3. Verifier si une memoire existante couvre ce sujet
   ├── Oui → Mettre a jour le fichier existant
   └── Non → Creer un nouveau fichier
    │
    ▼
4. Ecrire le fichier memoire (avec frontmatter)
    │
    ▼
5. Mettre a jour MEMORY.md (ajouter ou modifier l'entree)
```

### Lecture (a chaque session)

```
Demarrage de session
    │
    ▼
Charger MEMORY.md
    │
    ▼
Injecter dans le prompt systeme
    ├── Contenu de MEMORY.md (index)
    └── Fichiers memoire pertinents
    │
    ▼
L'IA a le contexte des sessions precedentes
```

## Systeme autoDream ("Claude Dreams")

### Principe

Le systeme autoDream consolide automatiquement les memoires en arriere-plan. Il s'execute quand certaines conditions sont remplies.

### Conditions de declenchement (gates)

Toutes les conditions doivent etre satisfaites :

| Gate | Condition |
|------|-----------|
| **Time gate** | >= 24 heures depuis la derniere consolidation |
| **Session gate** | >= 5 sessions depuis la derniere consolidation |
| **Lock gate** | Acquisition du verrou de consolidation (evite les doublons) |

### Phases de consolidation

```
Phase 1 : ORIENT
    │ Lire MEMORY.md et les fichiers existants
    │ Scanner les sujets couverts
    │
    ▼
Phase 2 : GATHER
    │ Collecter les nouvelles informations
    │ Lire les journaux quotidiens
    │ Analyser les transcripts recents
    │
    ▼
Phase 3 : CONSOLIDATE
    │ Ecrire ou mettre a jour les fichiers memoire
    │ Fusionner les informations redondantes
    │ Creer de nouveaux fichiers pour les nouveaux sujets
    │
    ▼
Phase 4 : PRUNE
    │ Garder l'index < 200 lignes
    │ Supprimer les memoires obsoletes
    │ Nettoyer les fichiers orphelins
```

### Implementation

```typescript
// Pseudo-code du systeme autoDream
async function runConsolidation(context: ConsolidationContext): Promise<void> {
  // Verifier les gates
  if (!await checkTimeGate()) return
  if (!await checkSessionGate()) return
  if (!await acquireLock()) return

  try {
    // Phase 1: Orient
    const existingMemories = await loadMemoryIndex()
    const topics = extractTopics(existingMemories)

    // Phase 2: Gather
    const dailyLogs = await readDailyLogs(since: lastConsolidation)
    const newSignals = extractSignals(dailyLogs)

    // Phase 3: Consolidate
    for (const signal of newSignals) {
      const matchingTopic = findMatchingTopic(signal, topics)
      if (matchingTopic) {
        await updateMemoryFile(matchingTopic, signal)
      } else {
        await createMemoryFile(signal)
      }
    }

    // Phase 4: Prune
    await pruneIndex(maxLines: 200)
    await removeStaleMemories()
  } finally {
    await releaseLock()
  }
}
```

### Journaux quotidiens

Le systeme maintient des journaux append-only :

```
~/.claude/memdir/daily/
├── 2026-03-31.md
├── 2026-03-30.md
└── 2026-03-29.md
```

Ces journaux capturent les evenements significatifs de chaque session.

## Integration dans le prompt systeme

Les memoires sont injectees dans le prompt systeme sous forme de contenu attache :

```
[system prompt]
...
As you answer the user's questions, you can use the following context:

## Memory
Contents of MEMORY.md:
- [User Role](user_role.md) — Senior Go dev
- [Testing Feedback](feedback_testing.md) — Use real DB

## Memory Files
Contents of user_role.md:
---
name: User Role
...
---
User is a senior backend developer...
```

## Pour recreer ce systeme

### 1. Structure de stockage

```typescript
const MEMORY_DIR = path.join(homedir(), '.my-tool', 'memory')
const INDEX_PATH = path.join(MEMORY_DIR, 'MEMORY.md')

interface MemoryFile {
  name: string
  description: string
  type: 'user' | 'feedback' | 'project' | 'reference'
  content: string
  filePath: string
}
```

### 2. Operations CRUD

```typescript
async function saveMemory(memory: MemoryFile): Promise<void> {
  const frontmatter = `---
name: ${memory.name}
description: ${memory.description}
type: ${memory.type}
---

${memory.content}`

  await writeFile(memory.filePath, frontmatter)
  await updateIndex(memory)
}

async function loadMemories(): Promise<MemoryFile[]> {
  const indexContent = await readFile(INDEX_PATH, 'utf-8')
  const links = parseMarkdownLinks(indexContent)

  return Promise.all(
    links.map(async link => {
      const content = await readFile(
        path.join(MEMORY_DIR, link.href), 'utf-8'
      )
      return parseMemoryFile(content, link.href)
    })
  )
}
```

### 3. Injection dans le contexte

```typescript
function buildMemoryContext(memories: MemoryFile[]): string {
  if (memories.length === 0) return ''

  const index = memories
    .map(m => `- [${m.name}](${path.basename(m.filePath)}) — ${m.description}`)
    .join('\n')

  const details = memories
    .map(m => `## ${m.name}\n${m.content}`)
    .join('\n\n')

  return `## Memory Index\n${index}\n\n## Memory Details\n${details}`
}
```

### 4. Consolidation automatique (simplifie)

```typescript
async function maybeConsolidate(): Promise<void> {
  const lastRun = await getLastConsolidationTime()
  const sessionCount = await getSessionCountSince(lastRun)

  if (Date.now() - lastRun < 24 * 60 * 60 * 1000) return
  if (sessionCount < 5) return

  // Lancer la consolidation en arriere-plan
  await consolidateMemories()
  await setLastConsolidationTime(Date.now())
}
```

### Points cles

1. **Fichiers plats** : Les memoires sont des fichiers Markdown simples
2. **Index legers** : MEMORY.md est un index rapide a scanner
3. **Frontmatter** : Metadata structuree dans chaque fichier
4. **Consolidation paresseuse** : Ne s'execute que quand necessaire
5. **Verrouillage** : Evite les consolidations concurrentes
6. **Elagage** : L'index reste compact (< 200 lignes)
