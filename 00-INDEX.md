# Documentation Technique - Claude Code

## Table des matieres

Ce dossier contient la documentation technique complete du projet **Claude Code**, le CLI officiel d'Anthropic pour interagir avec Claude. Cette documentation est concue pour permettre a toute personne de comprendre l'architecture et de recreer un projet similaire.

### Documents

| # | Fichier | Contenu |
|---|---------|---------|
| 01 | [Vue d'ensemble](01-VUE-D-ENSEMBLE.md) | Presentation du projet, stack technique, structure des fichiers |
| 02 | [Points d'entree et demarrage](02-ENTRYPOINTS-ET-DEMARRAGE.md) | Bootstrap, modes d'execution, initialisation |
| 03 | [Systeme de requetes (Query Engine)](03-QUERY-ENGINE.md) | Cycle de vie d'une requete, communication API, gestion des tokens |
| 04 | [Systeme d'outils (Tools)](04-SYSTEME-OUTILS.md) | Architecture des outils, registre, permissions, execution |
| 05 | [Systeme de commandes](05-SYSTEME-COMMANDES.md) | Commandes utilisateur, routage, extensibilite |
| 06 | [Interface terminal (Ink)](06-INTERFACE-TERMINAL-INK.md) | Rendu React dans le terminal, composants, hooks UI |
| 07 | [Gestion d'etat](07-GESTION-ETAT.md) | AppState, Store, Context Providers, types de messages |
| 08 | [Systeme de permissions](08-SYSTEME-PERMISSIONS.md) | Niveaux de risque, approbation, classification |
| 09 | [Systeme de memoire](09-SYSTEME-MEMOIRE.md) | Memoire persistante, consolidation (Dream), MEMORY.md |
| 10 | [Bridge (Controle distant)](10-BRIDGE.md) | Connexion WebSocket, synchronisation avec claude.ai |
| 11 | [Multi-agent et orchestration](11-MULTI-AGENT.md) | Mode coordinateur, sous-agents, taches paralleles |
| 12 | [Services](12-SERVICES.md) | API, analytics, telemetrie, cout, authentification |
| 13 | [Hooks et keybindings](13-HOOKS-ET-KEYBINDINGS.md) | Hooks React, hooks d'execution, raccourcis clavier |
| 14 | [Configuration et feature flags](14-CONFIGURATION.md) | Settings, feature gates, GrowthBook, MDM |
| 15 | [Securite](15-SECURITE.md) | Protection des fichiers, undercover mode, validation |
| 16 | [Guide de recreation](16-GUIDE-DE-RECREATION.md) | Guide pas-a-pas pour recreer un projet similaire |

### Conventions

- Les chemins de fichiers sont relatifs a la racine du projet (`/`)
- Les tailles de fichiers et nombres de lignes sont approximatifs
- Les feature flags sont indiques entre parentheses ex: `(KAIROS)`
