# New CLI — Process & Template

Proceso completo para construir un nuevo CLI de extracción desde cero.

---

## Fase 1: Recon del target (ANTES de codear)

Salida: `RESEARCH.md` en la raíz del repo.

### 1.1 Identificar el tipo
Usa el árbol de decisión en `TYPES.md`.

### 1.2 Mapear portales y auth
```bash
# Abrir el target en browser con network tab abierto
agent-browser --headed --session <name> open <url>

# Ver requests mientras navegas manualmente
agent-browser snapshot -i   # ver refs del DOM
```

Documentar en RESEARCH.md:
- Tabla de portales (URL | Auth | CAPTCHA | Contenido)
- Flujo de auth paso a paso con gotchas
- Headers especiales observados
- Endpoints descubiertos con ejemplo de request/response

### 1.3 Identificar anti-bot
- ¿Bloquea headless? → test con `agent-browser open <url>` (sin --headed)
- ¿Tiene Akamai? → buscar `akam/` o `_Incapsula_Resource` en network
- ¿Rate limiting? → intentar 10 requests seguidos, observar 429

---

## Fase 2: Scaffold del CLI

### 2.1 Estructura de carpetas
```
<nombre>-cli/
  index.ts          ← dispatcher
  server.ts         ← REST API (Bun.serve)
  package.json
  RESEARCH.md       ← del Fase 1
  CONTEXT.md        ← para AI agents
  src/
    config.ts
    constants.ts
    http.ts
    formatters.ts
    schemas/
    services/
    validation/
      input.ts
    commands/
    workflows/      ← solo Tipo B/C
    browser/        ← solo Tipo A/B/C
      client.ts
      auth.ts
      cdp.ts        ← solo si hay iframes cross-origin
      captcha.ts    ← solo si hay reCAPTCHA
    ui/
      chalk.ts
      spinner.ts
      table.ts
      banner.ts
    mcp/
      index.ts
```

### 2.2 Instalar bloques cligentic
```bash
# Bloques base obligatorios
bunx shadcn add https://cligentic.dev/r/audit-log.json
bunx shadcn add https://cligentic.dev/r/json-mode.json
bunx shadcn add https://cligentic.dev/r/global-flags.json
bunx shadcn add https://cligentic.dev/r/error-map.json
bunx shadcn add https://cligentic.dev/r/xdg-paths.json

# Bloques según tipo de target
bunx shadcn add https://cligentic.dev/r/trust-ladder.json   # si hay ops mutantes
bunx shadcn add https://cligentic.dev/r/doctor.json         # siempre recomendado
bunx shadcn add https://cligentic.dev/r/killswitch.json     # si maneja datos sensibles
```

### 2.3 package.json mínimo
```json
{
  "name": "<nombre>-cli",
  "version": "0.1.0",
  "bin": { "<nombre>": "./index.ts" },
  "scripts": {
    "dev": "bun run index.ts",
    "build": "bun build index.ts --outdir dist",
    "lint": "biome check src/"
  },
  "dependencies": {
    "zod": "^3.24.0",
    "@clack/prompts": "^1.0.0",
    "chalk": "^5.0.0",
    "commander": "^14.0.0"
  },
  "devDependencies": {
    "@biomejs/biome": "^1.9.0",
    "typescript": "^5.0.0"
  }
}
```

---

## Fase 3: Implementación por tipo

### Tipo A — Bearer token intercept
```typescript
// src/browser/auth.ts
let capturedToken: string | null = null;
page.on('response', async (res) => {
  if (res.url().includes('/auth') && res.status() === 200) {
    const body = await res.json().catch(() => null);
    if (body?.accessToken) capturedToken = body.accessToken;
  }
});
// polling hasta capturar token
const start = Date.now();
while (!capturedToken && Date.now() - start < 30_000)
  await new Promise(r => setTimeout(r, 2000));
await browser.close();
```

### Tipo B — Form POST + session cookie
```typescript
// src/browser/auth.ts
await page.goto('https://portal.example.com/login');
await page.fill('[name="username"]', creds.user);
await page.fill('[name="password"]', creds.pass);
await page.click('[type="submit"]');
await page.waitForURL('**/dashboard**');
const cookies = await context.cookies();
// guardar cookies en config
```

### Tipo B — CDP para input masks
```typescript
// src/browser/cdp.ts
const setter = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value')!.set!;
setter.call(el, value);
el.dispatchEvent(new Event('input', { bubbles: true }));
el.dispatchEvent(new Event('change', { bubbles: true }));
```

### Tipo D — OAuth2 PKCE local callback
```typescript
// src/commands/auth.ts
const code = await new Promise<string>((resolve) => {
  const srv = Bun.serve({
    port: 8888,
    fetch(req) {
      const code = new URL(req.url).searchParams.get('code');
      if (code) { srv.stop(); resolve(code); }
      return new Response('Puedes cerrar esta ventana.');
    }
  });
  open(`https://auth.example.com/oauth?redirect_uri=http://localhost:8888`);
});
```

### Tipo E — Next.js Server Action
```typescript
// src/http.ts
const res = await fetch(BASE_URL + '/dashboard', {
  method: 'POST',
  headers: {
    'Next-Action': ACTION_HASH,  // del recon
    'Content-Type': 'application/json',
  },
  body: JSON.stringify([payload]),
});
// parsear RSC stream — la línea "1:" tiene el JSON real
const lines = (await res.text()).split('\n');
const data = JSON.parse(lines.find(l => l.startsWith('1:'))!.slice(2));
```

---

## Fase 4: Checklist antes de publicar

### Funcionalidad
- [ ] `<cmd> login` funciona y guarda sesión
- [ ] `<cmd> --help` muestra todos los comandos
- [ ] `<cmd> --output json` produce JSON válido
- [ ] `<cmd> doctor` valida deps y auth
- [ ] Comandos mutantes tienen `--dry-run`
- [ ] Comandos destructivos tienen `--yes` o T2/T3 gate

### Calidad
- [ ] `src/validation/input.ts` valida todos los inputs del usuario
- [ ] `TypedError` en `http.ts` con `statusCode` y `isSessionExpired`
- [ ] Audit trail en `~/.config/<cmd>/audit/YYYY-MM-DD.jsonl`
- [ ] Biome lint sin errores

### Distribución
- [ ] `bin` en package.json apunta a `index.ts`
- [ ] `.npmignore` excluye `.env`, `node_modules`, `*.test.ts`
- [ ] `RESEARCH.md` completo (no publicar sin él)
- [ ] `CONTEXT.md` escrito para AI agents
- [ ] `src/mcp/index.ts` expone comandos como tools de Claude

### GitHub
- [ ] Repo en `github.com/klipso-reverse/<nombre>-cli`
- [ ] RESEARCH.md copiado a `_knowledge/targets/<nombre>.md`
- [ ] Agregado a la tabla de targets en `_knowledge/README.md`

---

## RESEARCH.md template

```markdown
# <Target> — Research

Recon realizado: <fecha>. Tool: agent-browser vX.X.X / Playwright.

## Portal Map

| Portal | URL | Auth | Anti-bot | Contiene |
|--------|-----|------|----------|----------|

## Auth Flow

### Paso 1: ...
### Paso 2: ...

### Gotchas descubiertos
- ...

## Anti-bot / WAF

- ¿Bloquea headless? → Sí/No
- ¿WAF? → Akamai / Cloudflare / Ninguno
- ¿Rate limiting? → X requests/min

## Endpoints descubiertos

### POST /api/auth/login
Request:
```json
{ "username": "...", "password": "..." }
```
Response:
```json
{ "accessToken": "eyJ..." }
```

## Notas adicionales
```

---

## CONTEXT.md template

```markdown
# <nombre>-cli — Agent Context

## El agente no es un operador de confianza
Breve descripción del dominio y por qué importa la cautela.

## Arquitectura
- Auth method: ...
- Browser: headed/headless / no browser
- Gotchas críticos: ...

## Auth flow
1. `<cmd> login` → ...
2. ...

## Output format
- `--output json`: siempre disponible
- `--output table`: default en TTY

## Archivos clave
- `src/browser/auth.ts` → ...
- `src/services/` → ...
```
