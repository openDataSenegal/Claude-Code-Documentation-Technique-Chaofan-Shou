# 02 - Points d'entree et demarrage

## Vue d'ensemble du demarrage

Claude Code supporte plusieurs modes d'execution, chacun avec son propre point d'entree. Le bootstrap commun effectue l'authentification, le chargement de configuration et l'initialisation des services.

## Points d'entree (`entrypoints/`)

### `cli.tsx` - Point d'entree principal

C'est le fichier execute au lancement de `claude-code`. Il implemente un systeme de fast-paths pour les operations legeres.

```
Lancement CLI
    │
    ├─→ --version           → Affiche version, exit
    ├─→ --dump-system-prompt → Affiche prompt, exit
    ├─→ --computer-use-mcp  → Lance serveur MCP Computer Use
    ├─→ --claude-in-chrome   → Lance serveur MCP Chrome
    ├─→ DAEMON worker mode  → Lance worker daemon
    ├─→ BRIDGE mode         → Lance bridge WebSocket
    │
    └─→ Mode normal         → Import dynamique de main.tsx
```

**Optimisations de demarrage :**
- Profilage du temps d'evaluation des modules (`startupProfiler.ts`)
- Pre-chargement parallele du keychain macOS
- Lecture parallele des settings MDM
- Import dynamique de `main.tsx` (lazy loading)

### Modes d'execution

| Mode | Declencheur | Description |
|------|------------|-------------|
| **REPL normal** | `claude-code "query"` | Interface interactive standard |
| **Bridge** | `CLAUDE_CODE_BRIDGE=1` | Serveur WebSocket pour claude.ai |
| **Daemon worker** | `--daemon-worker` | Sous-processus lean pour agents |
| **MCP Computer Use** | `--computer-use-mcp` | Serveur MCP standalone |
| **MCP Chrome** | `--claude-in-chrome-mcp` | Serveur MCP pour extension Chrome |
| **Remote CCR** | `CLAUDE_CODE_REMOTE=true` | Execution dans conteneur distant |

## `main.tsx` - Orchestration principale

Fichier de ~4700 lignes qui orchestre l'ensemble de l'application.

### Sequence d'initialisation

```
1. Parse des arguments CLI (commander/yargs)
    │
2. Initialisation telemetrie + analytics
    │
3. Prefetch du keychain (async, non-bloquant)
    │
4. Chargement des feature flags (GrowthBook)
    │
5. Verification de l'authentification API
    │
6. Chargement de la configuration (user + project)
    │
7. Initialisation du registre de commandes
    │
8. Initialisation du registre d'outils
    │
9. Demarrage des serveurs MCP configures
    │
10. Lancement du REPL
     │
     ├─→ Nouvelle session    → Ecran REPL vierge
     ├─→ Resume session      → Ecran ResumeConversation
     └─→ Mode bridge         → BridgeMain
```

### Gestion des sessions

```typescript
// Pseudo-code simplifie
if (args.resume) {
  // Charger session depuis le disque
  const session = await loadSession(args.resume)
  renderResumeScreen(session)
} else if (args.continue) {
  // Continuer la derniere session
  const lastSession = await getLastSession()
  renderResumeScreen(lastSession)
} else {
  // Nouvelle session
  renderREPL()
}
```

## `replLauncher.tsx` - Lanceur du REPL

Point d'entree pour le rendu de l'interface terminal interactive. Il :
1. Initialise le renderer Ink personnalise
2. Monte l'arbre React des composants
3. Configure les providers de contexte
4. Demarre la boucle d'entree clavier

## `setup.ts` - Configuration initiale

Gere la premiere utilisation et les mises a jour de configuration :
- Creation du dossier `~/.claude/`
- Migration des anciens formats de configuration
- Validation de la cle API
- Initialisation du systeme de memoire

## Diagramme de sequence du demarrage

```
Utilisateur          CLI             main.tsx          REPL
    │                 │                │                 │
    │── claude-code ─→│                │                 │
    │                 │── fast-path? ─→│                 │
    │                 │   (non)        │                 │
    │                 │── import() ───→│                 │
    │                 │                │── init telemetry│
    │                 │                │── load config   │
    │                 │                │── verify auth   │
    │                 │                │── load tools    │
    │                 │                │── load commands │
    │                 │                │── start MCP     │
    │                 │                │── render() ────→│
    │                 │                │                 │── mount React
    │                 │                │                 │── start input loop
    │←─────────────── prompt affiché ──────────────────│
```

## Gestion des erreurs au demarrage

Le bootstrap implemente une strategie de degradation gracieuse :

1. **Erreur d'authentification** → Propose `claude-code login`
2. **Erreur reseau** → Mode offline limite (lecture seule)
3. **Version obsolete** → Avertissement + lien de mise a jour
4. **Config corrompue** → Reset avec sauvegarde de l'ancienne config
5. **Feature flags indisponibles** → Utilise le cache local (stale-acceptable)

## Variables d'environnement cles

| Variable | Effet |
|----------|-------|
| `CLAUDE_CODE_BRIDGE` | Active le mode bridge |
| `CLAUDE_CODE_REMOTE` | Active le mode conteneur distant |
| `ANTHROPIC_API_KEY` | Cle API directe (bypass keychain) |
| `CLAUDE_CODE_DEBUG` | Active les logs de debug |
| `CLAUDE_CODE_MODEL` | Override du modele par defaut |

## Pour recreer cette partie

1. Creer un CLI avec un parser d'arguments (ex: `commander`, `yargs`)
2. Implementer des fast-paths pour les operations sans UI
3. Utiliser l'import dynamique pour le chargement paresseux des modules lourds
4. Profiler le temps de demarrage et optimiser les chemins critiques
5. Supporter plusieurs modes d'execution via des variables d'environnement
6. Implementer une degradation gracieuse pour chaque dependance externe
