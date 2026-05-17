# Crafter CLI — Metodología Maestra
> Análisis profundo de 7 repos: whatsapp-cli, bancolombia-cli, ubereats-cli, rappi-cli, sunat-cli, spoti-cli, trii-cli
> Fecha: 2026-04-11

---

## 1. El Framework Central: Recon → Document → Automate

Todo el family Crafter sigue esta metodología de 3 fases:

### Fase 1 — RECON (Descubrimiento)
- Abrir el portal/app con browser real (headless=false)
- Interceptar network requests: endpoints, headers, tokens
- Grabar HAR completo: `recordHar: { path: HAR_PATH, content: "embed" }`
- Snapshot del DOM para mapear formularios y elementos
- Identificar tipo de auth, anti-bot, y formato de respuesta

### Fase 2 — DOCUMENT (RESEARCH.md primero)
Artefacto central antes de escribir una línea de código:
```
## Portal Map       — URLs, auth, CAPTCHA, contenido
## Auth Flow        — bypass discoveries documentados
## Workflows        — paso a paso con gotchas numerados
## Endpoints        — método, payload, respuesta
## Edge Cases       — problemas + soluciones
## File Locations   — config, sessions, audit
```

### Fase 3 — AUTOMATE (CLI)
Arquitectura estandarizada (ver Sección 3).

---

## 2. Taxonomía de Targets (5 Tipos)

| Tipo | Descripción | Ejemplos | Técnica principal |
|------|------------|---------|-------------------|
| **A — API Privada** | API REST no documentada, captura token del browser | rappi-cli, ubereats-cli | Interceptar Bearer/cookie → HTTP puro |
| **B — Form-Based** | Portales con formularios HTML, sin API REST | sunat-cli, whatsapp-cli | Browser automation permanente + CDP |
| **C — Hybrid Banking** | Auth compleja + API directa con headers anti-fraude | bancolombia-cli | Human-in-the-loop + captura de device-id/cookies |
| **D — API Oficial** | OAuth2 documentado, sin reverse engineering | spoti-cli | PKCE + callback server local |
| **E — RSC/Next.js** | Server Actions de Next.js, stream RSC no-JSON | trii-cli | Parsear línea `1:` del RSC stream |

### Cuándo aplica cada tipo:

```
Target tiene API pública documentada?  → Tipo D (OAuth2 PKCE)
Target es Next.js con Server Actions?  → Tipo E (RSC parsing)
Target es un banco con anti-fraude?    → Tipo C (Human-in-the-loop)
Target tiene formularios HTML complejos? → Tipo B (CDP + browser permanente)
Target tiene app móvil/web con API?    → Tipo A (token capture)
```

---

## 3. Stack Técnico Estándar

| Componente | Elección | Razón |
|-----------|---------|-------|
| Runtime | **Bun** | File I/O nativo, sin transpilación, más rápido que Node |
| Lenguaje | **TypeScript strict** | Sin `any`, tipos inferidos de Zod |
| Validación | **Zod** | Runtime + compile-time, `safeParse`, `passthrough()` |
| CLI framework | **Commander.js** o dispatcher custom | Commander para subcommands, custom para simplicidad |
| HTTP server | **Hono** | Ultra-ligero, TypeScript-first, mismo para REST y MCP |
| Browser | **Playwright** | Solo para login capture o form-based permanente |
| UI | **chalk + ora + cli-table3** | Colors, spinners, tablas — copiable entre repos |
| Formatter | **Biome** | Más rápido que ESLint + Prettier |

---

## 4. Arquitectura Estandarizada (La misma en todos los repos)

```
target-cli/
├── index.ts              # Router/dispatcher de comandos
├── server.ts             # Hono REST API (port 32XX)
│
├── src/
│   ├── constants.ts      # BASE_URL, headers, ports, timeouts
│   ├── http.ts           # Cliente HTTP typed con generics
│   ├── config.ts         # Load/save config con Zod validation
│   ├── formatters.ts     # Moneda, fechas, locale-aware
│   │
│   ├── schemas/          # Contratos de datos (Zod)
│   │   ├── config.ts     # Discriminated union DirectConfig | ApiConfig
│   │   ├── *.ts          # Raw schemas (API) + Normalized schemas (CLI)
│   │
│   ├── services/         # Lógica de negocio ← CORAZÓN DEL REPO
│   │   └── *.ts          # Agnósticos de interfaz (CLI/REST/MCP usan los mismos)
│   │
│   ├── commands/         # Comandos CLI (cada uno es un script standalone)
│   │   └── *.ts
│   │
│   ├── api/
│   │   └── app.ts        # Hono routes (READ-ONLY para operaciones sensibles)
│   │
│   ├── mcp/
│   │   └── index.ts      # MCP server para Claude (tools con Zod params)
│   │
│   └── ui/               # Capa de presentación
│       ├── chalk.ts      # Brand colors + semantic helpers (ok, fail, warn)
│       ├── spinner.ts    # withSpinner() wrapper con TTY detection
│       ├── table.ts      # printTable() + printDetail()
│       └── banner.ts     # ASCII art + Chafa logo
```

### Principio clave: Un solo services layer
```
CLI commands  →  |
REST API      →  | → services/ → schemas/ → http.ts
MCP server    →  |
```
Sin duplicar lógica. El mismo `getPositions()` funciona para los tres.

---

## 5. Patrones de Autenticación por Tipo

### Tipo A/C — Token Capture (Playwright)
```typescript
// Capturar token durante login real del usuario
page.on("response", async (res) => {
  if (res.url().includes("oauth2/token") && res.status() === 200) {
    const body = await res.json();
    accessToken = body.data?.accessToken;
  }
});

// Capturar cookies anti-bot
const cookies = await context.cookies();
cookieStr = cookies
  .filter((c) => c.domain.includes("target.com"))
  .map((c) => `${c.name}=${c.value}`)
  .join("; ");
```

### Tipo A — Headers anti-detección
```typescript
const browser = await chromium.launch({
  headless: false,
  channel: "chrome",
  args: ["--disable-blink-features=AutomationControlled", "--no-sandbox"],
});
const context = await browser.newContext({
  userAgent: "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...",
  locale: "es-CO",
  timezoneId: "America/Bogota",
});
```

### Tipo C — Headers anti-fraude bancarios
```typescript
function buildBankHeaders(config: DirectConfig) {
  return {
    Authorization: `Bearer ${config.accessToken}`,
    "message-id": crypto.randomUUID(),     // Único por request
    "request-timestamp": preciseTimestamp(), // Con milisegundos
    "device-id": config.deviceId,           // Del login capture
    "session-tracker": config.sessionTracker,
    Cookie: config.cookies,                 // Cookies Imperva/anti-bot
    "app-version": "4.2.5",                // Spoofed, match deploy
  };
}

function preciseTimestamp(): string {
  const now = new Date();
  const pad = (n: number) => String(n).padStart(2, "0");
  return `${now.getFullYear()}-${pad(now.getMonth()+1)}-${pad(now.getDate())} ` +
         `${pad(now.getHours())}:${pad(now.getMinutes())}:${pad(now.getSeconds())}` +
         `:${String(now.getMilliseconds()).padStart(3, "0")}`;
}
```

### Tipo B — Persistent Browser Context (WhatsApp)
```typescript
// Primera vez: usuario ve el browser y hace login
// Siguientes veces: carga el mismo estado (IndexedDB + cookies)
const context = await pw.launchPersistentContext(stateDir, {
  headless: options.headless ?? true,
  channel: "chrome",
  args: ["--disable-blink-features=AutomationControlled"],
});
```

### Tipo D — OAuth2 PKCE (Spotify)
```typescript
// Sin client secret — el code verifier nunca sale del dispositivo
const verifier = generateCodeVerifier();      // 128 chars, base64url
const challenge = await sha256(verifier);     // Web Crypto API
// Callback server local en 127.0.0.1:8888
// Token refresh automático 60s antes de expirar
```

### Tipo E — RSC Next.js Server Actions (Trii)
```typescript
// POST con headers específicos de Next.js
const res = await fetch(`${WEB_BASE}${path}`, {
  method: "POST",
  headers: {
    "Content-Type": "text/plain;charset=UTF-8",
    "Next-Action": ACTION_HASH,                    // Hash del Server Action
    "Next-Router-State-Tree": buildRouterTree(path), // Árbol de rutas encoded
    Accept: "text/x-component",
    Cookie: config.cookies,
  },
  body: JSON.stringify(args),
});

// Parsear respuesta RSC: el valor útil está en la línea "1:"
function parseRscLine1<T>(rsc: string): T {
  const match = rsc.match(/^1:(.*)$/m);
  return JSON.parse(match![1]);
}
```

---

## 6. Patrones de Código Reutilizables

### 6.1 withSpinner (TTY-aware)
```typescript
export async function withSpinner<T>(message: string, fn: () => Promise<T>): Promise<T> {
  const spinner = ora({
    text: brandColor(message),
    stream: process.stderr,  // stderr para no romper pipes de stdout
    isSilent: !process.stderr.isTTY,
  }).start();
  try {
    const result = await fn();
    spinner.stop();
    return result;
  } catch (err) {
    spinner.stop();
    throw err;
  }
}
```

### 6.2 withBrowser (cleanup garantizado)
```typescript
export async function withBrowser<T>(
  fn: (page: Page) => Promise<T>,
  options?: { headless?: boolean },
): Promise<T> {
  const { context, page } = await launchBrowser(options);
  try {
    return await fn(page);
  } finally {
    await context.close(); // Siempre se ejecuta
  }
}
```

### 6.3 HTTP client typed con generics
```typescript
export async function get<T>(path: string, config: Config, schema: ZodSchema<T>): Promise<T> {
  const res = await fetch(`${BASE_URL}${path}`, { headers: buildHeaders(config) });
  if (!res.ok) handleError(res);
  const json = await res.json();
  return schema.parse(json);
}
```

### 6.4 Config con Discriminated Union
```typescript
const ConfigSchema = z.discriminatedUnion("mode", [
  z.object({ mode: z.literal("direct"), cookies: z.string(), ... }),
  z.object({ mode: z.literal("api"), apiUrl: z.string(), token: z.string() }),
]);
// Los services usan el mismo código para ambos modos
```

### 6.5 Output auto JSON/table (agent-first)
```typescript
export function output(format: "json" | "table" | "auto", data: {
  json: unknown;
  table?: { headers: string[]; rows: string[][] };
}) {
  const resolved = format === "auto" ? (process.stdout.isTTY ? "table" : "json") : format;
  if (resolved === "json") {
    Array.isArray(data.json) ? outputNDJSON(data.json) : console.log(JSON.stringify(data.json, null, 2));
  } else if (data.table) {
    outputTable(data.table.headers, data.table.rows);
  }
}
```

### 6.6 Error con hints accionables
```typescript
class AppError extends Error {
  constructor(message: string, public statusCode: number, public isSessionExpired = false) {
    super(message);
  }
}

// En commands:
catch (e) {
  if (e instanceof AppError && e.isSessionExpired) {
    console.error(`Session expired. Run '${toolName} login' to reconnect.`);
    process.exit(1);
  }
}
```

### 6.7 Audit Trail JSONL
```typescript
export function audit(entry: Omit<AuditEntry, "timestamp">): void {
  const today = new Date().toISOString().split("T")[0];
  const file = join(AUDIT_DIR, `${today}.jsonl`);
  appendFileSync(file, JSON.stringify({ timestamp: new Date().toISOString(), ...entry }) + "\n");
}
// Uso: audit({ command: "login", args: {}, result: "success" | "error", details: {} })
```

### 6.8 Session freshness check
```typescript
const SESSION_MAX_AGE_MS = 18 * 60 * 1000;

function isSessionFresh(path: string): boolean {
  if (!existsSync(path)) return false;
  return Date.now() - statSync(path).mtimeMs < SESSION_MAX_AGE_MS;
}
```

### 6.9 MCP tool pattern
```typescript
server.tool("get_data", "Description for Claude", {
  param: z.string().describe("What this param does"),
}, async ({ param }) => {
  const config = await loadConfig();
  const data = await getData(param, config);
  return {
    content: [{ type: "text", text: JSON.stringify(data, null, 2) }]
  };
});
```

### 6.10 Dry-run + safety cap (fintech)
```typescript
// En comandos de trading/mutación:
if (!opts.confirm) {
  printPreview(preview);
  console.log("Dry run — add --confirm to execute.");
  return;
}
if (preview.total > MAX_AMOUNT_COP && !opts.yolo) {
  console.error(`Amount exceeds safety cap. Use --yolo to bypass.`);
  process.exit(1);
}
```

---

## 7. Zod Patterns Específicos

### Raw → Normalized (separar schemas de API vs CLI)
```typescript
// Raw: lo que devuelve la API (frágil, puede cambiar)
const RawPositionSchema = z.object({
  stock: z.string(),
  average_cost: z.number(),
}).passthrough();  // Acepta campos extra sin romper

// Normalized: contrato interno del CLI (estable)
const PositionSchema = z.object({
  symbol: z.string(),
  avgPrice: z.number(),
  // ... campos renombrados y tipados estrictamente
});

// En el service: mapear raw → normalized
return raw.map(p => PositionSchema.parse({
  symbol: p.stock,
  avgPrice: p.average_cost,
}));
```

### Fallback defensivo en respuestas inestables
```typescript
// Cuando la API puede devolver el dato en varios paths:
const feedItems = data?.data?.feedItems || data?.feedItems || [];
const store = item?.store || item?.analyticsLabel?.store;
```

### Password solo desde ENV (nunca en config)
```typescript
export function getCredentials() {
  const config = loadConfig();
  const password = process.env.APP_PASSWORD; // NUNCA stored en disco
  if (!password) throw new Error("Set APP_PASSWORD env var");
  return { ...config, password };
}
```

---

## 8. Checklist para un Target Nuevo

### RECON
- [ ] Abrir con `chromium.launch({ headless: false })`
- [ ] Grabar HAR: `recordHar: { path: "~/.target/trii.har", content: "embed" }`
- [ ] Interceptar todos los requests: headers, tokens, cookies
- [ ] Identificar tipo de target (A/B/C/D/E)
- [ ] Detectar anti-bot: Cloudflare, Akamai, Imperva, reCAPTCHA
- [ ] Mapear todos los endpoints necesarios

### DOCUMENT (RESEARCH.md PRIMERO)
- [ ] Portal Map con URLs y auth
- [ ] Auth flow completo con bypass discoveries
- [ ] Workflows paso a paso con gotchas
- [ ] Endpoints descubiertos con payload y respuesta
- [ ] Edge cases y soluciones

### SCAFFOLD
- [ ] `git init` + primer commit antes de escribir lógica
- [ ] Copiar scaffold: `src/http.ts`, `src/config.ts`, `src/ui/`
- [ ] Definir schemas Zod (raw + normalized)
- [ ] Implementar auth flow del tipo correspondiente

### IMPLEMENTAR
- [ ] Services layer (agnóstico de interfaz)
- [ ] Commands CLI (thin wrappers de services)
- [ ] REST API con Hono (solo reads)
- [ ] MCP server (sin operaciones destructivas)
- [ ] `--output json/table/auto` en todos los reads
- [ ] `--dry-run` en todas las mutaciones
- [ ] Audit trail JSONL

### TESTEAR
- [ ] Smoke test: login → operación básica → logout
- [ ] `--output json` para todos los comandos de lectura
- [ ] Dry-run de todas las mutaciones
- [ ] MCP tools funcionan en Claude

---

## 9. Principios de Diseño del Family

1. **Agent-first I/O** — `--output json` en todo. Diseñado para ser usado por AI agents, no solo humanos.

2. **RESEARCH.md como memoria persistente** — El conocimiento del recon vive en el repo, reproducible por futuros agentes.

3. **Services layer agnóstico** — CLI, REST API y MCP comparten exactamente el mismo código de negocio.

4. **Human-in-the-loop para auth crítica** — Para bancos y sistemas anti-bot: el humano hace el login real, el CLI solo observa y captura.

5. **--dry-run obligatorio** — Toda operación mutante tiene preview sin ejecutar por defecto.

6. **CDP como escape hatch** — Cuando agent-browser no puede (cross-origin iframes, Angular masks), raw CDP WebSocket resuelve.

7. **Zod end-to-end** — Validación en frontera de entrada (config, API responses, params de MCP).

8. **Separación Raw/Normalized** — Los schemas de la API pueden cambiar. Los schemas del CLI son estables.

9. **MCP con restricciones explícitas** — Claude puede leer todo, pero trading/mutaciones sensibles se excluyen del MCP.

10. **Password solo en ENV** — Credenciales sensibles nunca se almacenan en disco.

---

## 10. Lecciones Específicas por Repo

### rappi-cli
- `app-version` header usa commit hash hardcodeado → rompe con cada deploy de Rappi
- ID de producto debe ser compuesto: `"storeId_productId"`
- Precio necesita 3 campos: `price`, `real_price`, `markup_price`

### sunat-cli
- SOL viejo sin CAPTCHA → redirige a Nueva Plataforma con state OAuth (bypass de reCAPTCHA)
- CDP raw WebSocket para cross-origin iframes con Angular input masks
- `nativeInputValueSetter` para bypassear validators de Angular
- Sesión expira en 20 minutos → siempre verificar freshness antes de operar

### whatsapp-cli
- `launchPersistentContext()` guarda IndexedDB + cookies → no re-auth entre comandos
- Aria labels en inglés Y español: `div[aria-label="Chat list"], div[aria-label="Lista de chats"]`
- Spinners siempre a `stderr`, datos a `stdout` → pipes funcionan

### bancolombia-cli
- Headers anti-fraude: `message-id` (UUID único), `request-timestamp` (con milisegundos)
- Session tracker + device-id capturados durante login → deben coincidir en cada request
- Cookies Imperva en cada request o falla con 403
- Session TTL: ~6 minutos

### ubereats-cli
- Cart identificado por `draftOrderUuid` → persiste en config entre comandos
- Precios en centavos (1299 = $12.99) → dividir por 100 siempre
- API responses inestables → usar fallback chains: `data?.path1 || data?.path2 || []`

### spoti-cli
- PKCE: code verifier se genera local, nunca sale del dispositivo → más seguro que client_secret
- Callback server local en `127.0.0.1:8888` → forzar cierre de conexiones con `conn.destroy()`
- Token refresh proactivo 60s antes de expirar → cero interrupciones mid-command
- Scopes por comando: validar que el token tiene los permisos antes de hacer el request

### trii-cli
- Action hashes de Next.js Server Actions rotan en cada deploy → re-capturar del HAR
- Respuesta RSC: parsear solo la línea que empieza con `1:` → JSON válido
- `Next-Router-State-Tree` header: árbol de rutas encodeado que el server valida
- Fee de Trii: `12,500 COP flat + 19% IVA` para órdenes < 5M COP → computar client-side

---

## 11. Tabla Comparativa de los 7 Repos

| Repo | Tipo | Auth | Browser | MCP | REST API | Output |
|------|------|------|---------|-----|----------|--------|
| rappi-cli | A | Token capture | Solo login | ✅ 14 tools | ✅ | json/table |
| sunat-cli | B | Session cookies | Permanente | ❌ | ❌ | json/table/auto |
| whatsapp-cli | B | Persistent context | Permanente | ✅ 5 tools | ✅ Hono | table |
| bancolombia-cli | C | Human-in-loop + cookies | Solo login | ✅ 6 tools | ✅ Hono | table |
| ubereats-cli | A | Cookie capture | Solo login | ✅ 16 tools | ✅ Hono | table |
| spoti-cli | D | OAuth2 PKCE | Solo callback | ❌ | ❌ | json/table |
| trii-cli | E | RSC + cookies | Solo login | ✅ 9 tools | ✅ Hono | table |

---

## 12. Qué Copiar Directamente Entre Repos

| Componente | Copiar de | Qué cambiar |
|-----------|----------|------------|
| `src/ui/` completo | rappi-cli o ubereats-cli | Brand colors (hex) |
| `src/http.ts` base | ubereats-cli | BASE_URL, headers |
| `src/config.ts` | bancolombia-cli | Schema Zod del config |
| `withBrowser()` | whatsapp-cli | Nada, es genérico |
| `withSpinner()` | cualquiera | Nada, es genérico |
| Audit trail | sunat-cli | Paths de la app |
| MCP server | rappi-cli o trii-cli | Tools y services |
| OAuth2 PKCE | spoti-cli | Client ID, scopes |
| RSC client | trii-cli | Action hashes, paths |
