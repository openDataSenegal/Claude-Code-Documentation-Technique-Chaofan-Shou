# 10 - Bridge (Controle distant)

## Vue d'ensemble

Le systeme Bridge permet de controler une instance Claude Code CLI depuis l'interface web de claude.ai. Il etablit une connexion WebSocket authentifiee par JWT entre le navigateur et le terminal local.

## Architecture

```
claude.ai (navigateur)
    │
    │ JWT + mode de travail
    ▼
Bridge HTTP API (bridgeApi.ts)
    │
    │ Upgrade vers WebSocket
    ▼
REPL Bridge Transport (replBridgeTransport.ts)
    │
    │ Frames de messages
    ▼
Remote Bridge Core (remoteBridgeCore.ts)
    │
    │ Routage des messages
    ▼
REPL local (replBridge.ts)
    │
    ▼
Etat local de l'application
```

## Fichiers principaux

```
bridge/
├── bridgeMain.ts            # Cycle de vie du bridge (~115KB)
├── replBridge.ts            # Integration REPL (~100KB)
├── remoteBridgeCore.ts      # Routage des messages (~39KB)
├── bridgeApi.ts             # Couche HTTP API (~18KB)
├── bridgeUI.ts              # Synchronisation UI (~16KB)
├── replBridgeTransport.ts   # Transport de messages (~15KB)
└── ... (27 autres fichiers)
```

## Modes de travail

| Mode | Description |
|------|-------------|
| `single-session` | Une seule session partagee entre web et CLI |
| `worktree` | Chaque session web cree un git worktree isole |
| `same-dir` | Sessions multiples dans le meme repertoire |

## Flux d'authentification

```
1. L'utilisateur lance `claude-code bridge` dans le terminal
    │
    ▼
2. Le bridge genere un code d'appairage
    │ Affiche : "Pairing code: ABCD-1234"
    │
    ▼
3. L'utilisateur entre le code sur claude.ai
    │
    ▼
4. claude.ai envoie une requete avec le code
    │ + JWT token signe
    │
    ▼
5. Le bridge valide le JWT
    │ ├── Verifie la signature
    │ ├── Verifie l'expiration
    │ └── Verifie le code d'appairage
    │
    ▼
6. Connexion etablie → Upgrade WebSocket
    │
    ▼
7. Session de travail commence
```

### Tokens de confiance

Pour les appareils deja appaires, un **trusted device token** evite de re-authentifier :

```typescript
type TrustedDeviceToken = {
  deviceId: string
  issuedAt: number
  expiresAt: number
  securityTier: 'basic' | 'elevated'
}
```

## Protocole de communication

### Format des frames

```typescript
type BridgeFrame = {
  type: 'message' | 'tool_result' | 'status' | 'attachment' | 'error'
  id: string
  payload: unknown
  timestamp: number
}
```

### Types de messages

| Type | Direction | Description |
|------|-----------|-------------|
| `user_message` | Web → CLI | Message utilisateur depuis le navigateur |
| `assistant_message` | CLI → Web | Reponse de l'IA vers le navigateur |
| `tool_progress` | CLI → Web | Progression d'un outil en cours |
| `tool_result` | CLI → Web | Resultat d'execution d'outil |
| `permission_request` | CLI → Web | Demande de permission a l'utilisateur |
| `permission_response` | Web → CLI | Reponse de permission |
| `attachment_upload` | Web → CLI | Fichier uploade depuis le navigateur |
| `session_status` | CLI → Web | Statut de la session (actif, idle) |
| `heartbeat` | Bidirectionnel | Keep-alive |

### Synchronisation de l'UI

Le bridge synchronise l'etat UI entre le terminal et le web :

```
CLI State                    Web UI
    │                           │
    ├── messages[] ────────────→│ Historique de conversation
    ├── isStreaming ───────────→│ Indicateur de streaming
    ├── activeTools[] ─────────→│ Outils en cours
    ├── permissions[] ─────────→│ Dialogues de permission
    │                           │
    │←──── user input ─────────┤ Saisie utilisateur
    │←──── file upload ────────┤ Pieces jointes
    │←──── permission ─────────┤ Decisions de permission
```

## Gestion des sessions

### Polling de session

Le bridge poll periodiquement pour detecter les nouvelles sessions :

```typescript
async function pollForSessions(): Promise<void> {
  while (bridgeActive) {
    const sessions = await fetchPendingSessions()
    for (const session of sessions) {
      await handleNewSession(session)
    }
    await sleep(POLL_INTERVAL)  // ~2 secondes
  }
}
```

### Lifecycle d'une session

```
CONNECTING → AUTHENTICATING → ACTIVE → IDLE → DISCONNECTED
                                │
                                ├── TOOL_EXECUTING
                                ├── WAITING_PERMISSION
                                └── STREAMING
```

## Gestion des erreurs

| Erreur | Action |
|--------|--------|
| WebSocket deconnecte | Reconnexion automatique avec backoff |
| JWT expire | Re-authentification via code d'appairage |
| Timeout de heartbeat | Fermeture propre + notification |
| Erreur de session | Logging + notification web |

## Pour recreer ce systeme

### 1. Serveur WebSocket local

```typescript
import { WebSocketServer } from 'ws'

const wss = new WebSocketServer({ port: 8765 })

wss.on('connection', (ws, req) => {
  // Valider le JWT
  const token = extractToken(req)
  if (!validateJWT(token)) {
    ws.close(4001, 'Unauthorized')
    return
  }

  ws.on('message', (data) => {
    const frame = JSON.parse(data.toString()) as BridgeFrame
    handleFrame(frame, ws)
  })
})
```

### 2. Protocole de messages

```typescript
function handleFrame(frame: BridgeFrame, ws: WebSocket): void {
  switch (frame.type) {
    case 'user_message':
      // Injecter le message dans la session REPL
      replSession.addUserMessage(frame.payload as string)
      break

    case 'permission_response':
      // Resoudre la permission en attente
      pendingPermissions.get(frame.id)?.resolve(frame.payload)
      break

    case 'attachment_upload':
      // Sauvegarder le fichier localement
      saveAttachment(frame.payload as AttachmentData)
      break
  }
}

function sendToWeb(ws: WebSocket, type: string, payload: unknown): void {
  ws.send(JSON.stringify({
    type,
    id: generateId(),
    payload,
    timestamp: Date.now()
  }))
}
```

### 3. Synchronisation d'etat

```typescript
// Observer les changements d'etat et les transmettre au web
store.subscribe((state) => {
  sendToWeb(ws, 'session_status', {
    messages: state.messages.length,
    isStreaming: state.isStreaming,
    activeTools: state.activeToolCalls.size
  })
})
```

### Points cles

1. **JWT pour l'authentification** : Ne jamais transmettre de credentials en clair
2. **WebSocket pour le temps reel** : Plus efficace que le polling HTTP
3. **Heartbeat** : Detecter les deconnexions fantomes
4. **Reconnexion automatique** : Avec backoff exponentiel
5. **Synchronisation unidirectionnelle** : L'etat CLI est la source de verite
