# 15 - Securite

## Vue d'ensemble

La securite est un pilier fondamental de Claude Code. Le systeme implemente une defense en profondeur avec plusieurs couches de protection contre les actions non autorisees, les fuites de donnees et les attaques.

## Couches de securite

```
┌───────────────────────────────────────────────────┐
│ Couche 1 : Prompt systeme (cyber risk instruction)│
│ Instructions de securite dans le prompt IA        │
├───────────────────────────────────────────────────┤
│ Couche 2 : Systeme de permissions                 │
│ Approbation utilisateur pour les actions risquees │
├───────────────────────────────────────────────────┤
│ Couche 3 : Validation des inputs                  │
│ Path traversal, injection, sanitization           │
├───────────────────────────────────────────────────┤
│ Couche 4 : Fichiers proteges                      │
│ Liste noire des fichiers sensibles                │
├───────────────────────────────────────────────────┤
│ Couche 5 : Sandboxing                             │
│ Isolation des commandes shell                     │
├───────────────────────────────────────────────────┤
│ Couche 6 : Undercover mode                        │
│ Protection des informations internes Anthropic    │
└───────────────────────────────────────────────────┘
```

## Cyber Risk Instruction

Le fichier `constants/cyberRiskInstruction.ts` contient les instructions de securite injectees dans le prompt systeme :

```
⚠️ DO NOT MODIFY WITHOUT SAFEGUARDS TEAM REVIEW
Owned by: Safeguards Team (Anthropic)
```

Contenu (resume) :
- L'IA doit assister uniquement pour des tests de securite autorises
- Contextes autorises : pentesting, CTF, recherche en securite, defense
- Interdit : attaques destructives, DoS, ciblage de masse, evasion de detection
- Les outils dual-use requierent un contexte d'autorisation clair

## Validation des inputs

### Prevention de traversal de chemin

```typescript
function validatePath(inputPath: string): ValidationResult {
  // 1. URL-encoded traversal
  if (/%2e/i.test(inputPath)) {
    return { valid: false, reason: 'URL-encoded path traversal' }
  }

  // 2. Unicode normalization attack
  if (inputPath !== inputPath.normalize('NFC')) {
    return { valid: false, reason: 'Unicode normalization mismatch' }
  }

  // 3. Double-dot traversal
  if (inputPath.includes('..')) {
    return { valid: false, reason: 'Parent directory traversal' }
  }

  // 4. Backslash injection (Windows paths)
  if (/\\\.\./.test(inputPath)) {
    return { valid: false, reason: 'Backslash path traversal' }
  }

  // 5. Null byte injection
  if (inputPath.includes('\0')) {
    return { valid: false, reason: 'Null byte in path' }
  }

  // 6. Case-insensitive manipulation (macOS/Windows)
  const resolved = path.resolve(inputPath)
  if (resolved.toLowerCase() !== path.resolve(inputPath).toLowerCase()) {
    return { valid: false, reason: 'Case manipulation detected' }
  }

  return { valid: true }
}
```

### Validation des commandes shell

```typescript
function validateCommand(command: string): ValidationResult {
  // Patterns dangereux
  const DANGEROUS_PATTERNS = [
    /rm\s+-rf\s+\/(?!\w)/,     // rm -rf / (mais pas rm -rf /tmp/dir)
    /mkfs\./,                   // mkfs.ext4 etc.
    /dd\s+.*of=\/dev/,         // Ecriture directe sur device
    /:()\s*{\s*:|:&\s*};/,     // Fork bomb
    />\s*\/dev\/sd/,            // Redirection vers disque
    /chmod\s+-R\s+777\s+\//,   // Permissions recursives sur /
  ]

  for (const pattern of DANGEROUS_PATTERNS) {
    if (pattern.test(command)) {
      return {
        valid: false,
        reason: `Potentially dangerous command pattern detected`
      }
    }
  }

  return { valid: true }
}
```

## Fichiers proteges

Liste des fichiers automatiquement proteges contre l'edition :

```typescript
const PROTECTED_FILES = [
  // Configuration shell
  '.bashrc', '.bash_profile', '.zshrc', '.zprofile',
  '.profile', '.login',

  // Git
  '.gitconfig', '.git/config',

  // SSH
  '.ssh/authorized_keys', '.ssh/config',
  '.ssh/id_*', '.ssh/known_hosts',

  // Claude Code config
  '.mcp.json', '.claude.json',

  // Secrets
  '.env', '.env.*',
  'credentials.json',
  '*.pem', '*.key',

  // Systeme
  '/etc/*', '/usr/*', '/bin/*',
]
```

Toute tentative de modification est **automatiquement refusee** avec un message explicatif.

## Sandboxing des commandes shell

### Principe

Le BashTool peut etre execute dans un sandbox qui limite les operations :

```typescript
type SandboxConfig = {
  allowNetwork: boolean
  allowFileWrite: boolean
  writableDirectories: string[]
  readOnlyDirectories: string[]
  maxExecutionTime: number  // ms
  maxMemory: number         // bytes
}
```

### Niveaux de sandboxing

| Niveau | Reseau | Ecriture | Lecture | Usage |
|--------|--------|----------|---------|-------|
| `none` | Oui | Oui | Oui | Mode bypass |
| `basic` | Oui | Projet uniquement | Tout | Mode default |
| `strict` | Non | Projet uniquement | Projet uniquement | Mode securise |

## Undercover Mode

### Principe

Quand Claude Code est utilise dans un depot public, le mode "undercover" s'active pour eviter la fuite d'informations internes Anthropic.

### Activation

```typescript
function shouldActivateUndercover(): boolean {
  // Actif par defaut sauf dans les repos internes
  if (isInternalRepo()) return false
  if (process.env.CLAUDE_CODE_UNDERCOVER === 'false') return false
  return true
}
```

### Protections

Quand actif, le mode undercover :
- **Masque les noms de modeles internes** (codenames comme Fennec, Capybara)
- **Supprime les references a Slack** (channels internes)
- **Evite les mentions de projets internes** (Tengu, Kairos, etc.)
- **Filtre les URLs internes** (staging, internal APIs)
- **Utilise des noms generiques** dans les commits et PRs

## Gestion des secrets

### Principes

1. **Jamais dans le code** : Les secrets vont dans les variables d'environnement ou le keychain
2. **Jamais dans les logs** : Scrubbing automatique des patterns de cles API
3. **Jamais dans les commits** : Detection et avertissement avant commit
4. **Stockage securise** : Keychain macOS pour les cles API

### Detection de secrets

```typescript
const SECRET_PATTERNS = [
  /sk-ant-[a-zA-Z0-9]{20,}/,           // Cle API Anthropic
  /sk-[a-zA-Z0-9]{20,}/,               // Cle API generique
  /ghp_[a-zA-Z0-9]{36}/,               // Token GitHub
  /-----BEGIN\s+(?:RSA\s+)?PRIVATE KEY/, // Cle privee
  /AIza[a-zA-Z0-9_-]{35}/,             // Cle Google
  /AKIA[A-Z0-9]{16}/,                   // Cle AWS
]

function containsSecret(text: string): boolean {
  return SECRET_PATTERNS.some(p => p.test(text))
}
```

## Securite des communications

### API
- Toutes les communications API utilisent HTTPS/TLS
- Les cles API sont transmises via header `x-api-key`
- L'attestation client verifie l'identite du binaire

### Bridge
- Authentification JWT avec expiration
- Tokens de confiance pour les appareils appaires
- WebSocket sur TLS (WSS)

## OWASP Top 10 - Mesures

| Risque OWASP | Mesure |
|--------------|--------|
| A01 - Broken Access Control | Systeme de permissions, fichiers proteges |
| A02 - Cryptographic Failures | TLS, keychain, pas de secrets en clair |
| A03 - Injection | Validation des paths, sanitization des commandes |
| A04 - Insecure Design | Defense en profondeur, principe du moindre privilege |
| A05 - Security Misconfiguration | Defaults securises, MDM pour l'entreprise |
| A06 - Vulnerable Components | Dependencies a jour, build reproductible |
| A07 - Auth Failures | JWT, keychain, verification de cle |
| A08 - Data Integrity | Validation des inputs, checksums |
| A09 - Logging Failures | Analytics avec PII scrubbing |
| A10 - SSRF | Validation des URLs, pas de fetch arbitraire |

## Pour recreer ce systeme

### 1. Validation des chemins

```typescript
function safePath(inputPath: string, allowedBase: string): string | null {
  const resolved = path.resolve(inputPath)

  // Verifier que le chemin reste dans le repertoire autorise
  if (!resolved.startsWith(path.resolve(allowedBase))) {
    return null  // Path traversal detecte
  }

  // Verifier contre la liste noire
  const basename = path.basename(resolved)
  if (PROTECTED_FILES.includes(basename)) {
    return null  // Fichier protege
  }

  return resolved
}
```

### 2. Scrubbing de secrets

```typescript
function scrubSecrets(text: string): string {
  let scrubbed = text
  for (const pattern of SECRET_PATTERNS) {
    scrubbed = scrubbed.replace(pattern, '<REDACTED>')
  }
  return scrubbed
}
```

### 3. Defense en profondeur

```typescript
async function executeToolSafely(
  tool: Tool,
  input: unknown,
  context: Context
): Promise<ToolResult> {
  // Couche 1 : Validation des inputs
  const validation = validateInput(tool.name, input)
  if (!validation.valid) return error(validation.reason)

  // Couche 2 : Permissions
  const permission = await checkPermission(tool, input, context)
  if (permission === 'deny') return error('Permission denied')

  // Couche 3 : Fichiers proteges
  if (isProtectedTarget(tool, input)) return error('Protected file')

  // Couche 4 : Sandboxing
  const sandboxed = wrapInSandbox(tool, context.sandboxConfig)

  // Couche 5 : Execution avec timeout
  return withTimeout(
    sandboxed.execute(input, context),
    context.maxExecutionTime
  )
}
```

### Points cles

1. **Defense en profondeur** : Ne jamais se fier a une seule couche
2. **Fail-safe** : En cas de doute, refuser l'action
3. **Transparence** : L'utilisateur voit ce qui est execute
4. **Principe du moindre privilege** : Minimum de droits necessaires
5. **Validation a chaque couche** : Ne jamais supposer que l'input est sur
