# Best Practices — Extraídas de los CLIs de referencia

Patrones concretos con fuentes. Copiar directamente en nuevos CLIs.

---

## 1. Error handling — patrón bancolombia-cli (más completo)

```typescript
// src/http.ts
export class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly isSessionExpired: boolean = false,
  ) {
    super(message);
    this.name = "AppError";
  }
}

function parseErrorBody(body: string): string {
  try {
    const json = JSON.parse(body);
    // Formato anidado: { errors: [{ message: '{"title":"...","description":"..."}' }] }
    if (json.errors?.[0]?.message) {
      try {
        const inner = JSON.parse(json.errors[0].message);
        if (inner.description) return inner.description;
        if (inner.title) return inner.title;
      } catch {
        return json.errors[0].reason || json.errors[0].message;
      }
    }
    if (json.message) return json.message;
    if (json.error) return json.error;
  } catch {}
  return body.substring(0, 200);
}

function isExpiredError(body: string): boolean {
  const lower = body.toLowerCase();
  return (
    lower.includes("token inv") ||
    lower.includes("session expired") ||
    lower.includes("unauthorized")
  );
}

// En cada command: catch isSessionExpired → mensaje específico
if (e instanceof AppError && e.isSessionExpired) {
  console.error(`Sesión expirada. Corre: <app> login`);
  process.exit(1);
}
```

**Fuente:** `bancolombia-cli/src/http.ts`, `bancolombia-cli/src/commands/*.ts`

---

## 2. Session status display — `whoami` command (bancolombia-cli)

Comando que muestra estado de sesión + cuentas si está activa. Evita que el usuario tenga que adivinar si el login sigue válido.

```typescript
// src/commands/whoami.ts
const expiresAt = new Date(config.expiresAt);
const isExpired = expiresAt < new Date();
const minutesLeft = Math.max(0, Math.round((expiresAt.getTime() - Date.now()) / 60000));

printDetail("Session Info", [
  ["Expires", expiresAt.toLocaleString("es-PE")],
  ["Status", isExpired ? warn("Expired") : success(`Active (${minutesLeft}min left)`)],
]);
```

**Cuándo usar:** cualquier CLI con TTL < 1h. bancolombia 6min, trii 1h, interbank 5min.
**Fuente:** `bancolombia-cli/src/commands/whoami.ts`

---

## 3. Dual-mode config (direct vs API proxy) — bancolombia-cli

Permite usar el CLI tanto con browser directo como con un proxy backend. Útil cuando el cliente no puede/quiere ejecutar Playwright.

```typescript
// src/schemas/config.ts
const DirectConfig = z.object({
  mode: z.literal("direct"),
  accessToken: z.string(),
  deviceId: z.string(),
  sessionTracker: z.string(),
  ip: z.string(),
  cookies: z.string(),
  expiresAt: z.string(),
});

const ApiConfig = z.object({
  mode: z.literal("api"),
  token: z.string(),
  apiUrl: z.string(),
  expiresAt: z.string(),
});

// En http.ts: resolveUrl() y resolveHeaders() eligen según config.mode
```

**Fuente:** `bancolombia-cli/src/schemas/config.ts`, `bancolombia-cli/src/http.ts`

---

## 4. Zod response validation — bancolombia-cli

```typescript
// get/post aceptan schema Zod opcional para validar response
export async function get<T>(path: string, config: Config, schema?: z.ZodType<T>): Promise<T> {
  const data = await res.json();
  if (schema) return schema.parse(data);  // lanza ZodError si response inesperado
  return data as T;
}
```

**Por qué:** Si el portal cambia su response structure, falla con error claro en lugar de `undefined is not a function` random.
**Fuente:** `bancolombia-cli/src/http.ts`

---

## 5. Anti-fraud headers computados en runtime — bancolombia-cli

```typescript
// Timestamp exacto en milisegundos para cada request
const ts = `${year}-${mo}-${day} ${h}:${m}:${s}:${ms}`;
headers["request-timestamp"] = ts;
headers["message-id"] = crypto.randomUUID();  // UUID fresco por request
headers["device-id"] = config.deviceId;       // persistente en config
headers["session-tracker"] = config.sessionTracker; // capturado en login
```

**Cuándo usar:** bancos con anti-fraud headers. Estos valores se capturan en login via `page.on('request')` y se replican exactamente.
**Fuente:** `bancolombia-cli/src/http.ts` → `bancolombiaHeaders()`

---

## 6. REST server con Hono — trii-cli (más maduro que Bun.serve)

```typescript
// server.ts — en lugar de Bun.serve raw
import { Hono } from "hono";
import { cors } from "hono/cors";

const app = new Hono<{ Variables: { config: AppConfig } }>();

app.use("*", cors());
app.use("/api/*", async (c, next) => {
  const config = await loadConfig();
  c.set("config", config);
  await next();
});

app.get("/api/positions", async (c) => c.json(await getPositions(c.get("config"))));
app.get("/api/whoami", async (c) => {
  const { expiresAt } = c.get("config");
  return c.json({ expired: new Date(expiresAt) < new Date(), expiresAt });
});

export default app;
```

**Ventaja sobre Bun.serve raw:** middleware tipado, CORS integrado, routing limpio, compatible con Cloudflare Workers si se migra.
**Fuente:** `trii-cli/src/api/app.ts`

---

## 7. Audit trail two-phase — v0-cli

Para ops destructivas: registrar inicio + fin con duración y resultado.

```typescript
// src/lib/audit/jsonl.ts
const auditId = await auditStart({ command: 'deploy delete', chatId, trustLevel: 'T3' });
try {
  const result = await deleteDeployment(id);
  await auditEnd(auditId, { status: 'ok', deployId: result.id });
} catch (e) {
  await auditEnd(auditId, { status: 'error', error: e.message });
  throw e;
}
```

JSONL en `~/.config/<app>/audit/YYYY-MM-DD.jsonl`:
```json
{"auditId":"abc","command":"deploy delete","startedAt":"...","duration":1234,"status":"ok"}
```

**Fuente:** `v0-cli/src/lib/audit/jsonl.ts`

---

## 8. Cache-first con login-sync — interbank-cli

Para targets con TTL < 10 min. Login vuelca todo, queries leen caché local.

```
login-sync → browser abre → usuario hace login → CLI llama TODOS los endpoints → guarda en ~/.config/<app>/cache.json → browser cierra
accounts   → lee cache.json (sin red, instantáneo)
```

```typescript
// src/commands/login-sync.ts — después de capturar auth cookies:
const [accounts, movements, profile] = await Promise.all([
  fetchAccounts(cookies),
  fetchMovements(cookies),
  fetchProfile(cookies),
]);
await saveCache({ accounts, movements, profile, cachedAt: new Date().toISOString() });
```

**Cuándo usar:** TTL < 10 min (bancolombia 6min, interbank 5min, bbva 10-15min).
**Fuente:** `Cli-propios/interbank-cli/src/commands/login-sync.ts`

---

## 9. agent-browser session persistence — sunat-cli

Para Tipo B (browser permanente), persistir sesión entre comandos con `--session name`:

```bash
agent-browser --headed --session sunat open "https://e-menu.sunat.gob.pe/..."
agent-browser --session sunat state save ~/.sunat/sessions/sol.json
# siguiente comando:
agent-browser --session sunat state load ~/.sunat/sessions/sol.json
```

Sessions en `~/.sunat/sessions/` sobreviven entre invocaciones. Re-login solo cuando `invalidsesskey` response.
**Fuente:** `_knowledge/targets/sunat.md`

---

## 10. `maskSensitive` en output — bancolombia-cli

```typescript
// src/formatters.ts
export function maskAccount(number: string): string {
  return number.slice(0, 4) + "****" + number.slice(-4);
}
// Output: "2003****0700" en lugar de "200332560187"
```

**Regla:** números de cuenta, DNI, emails siempre maskeados en stdout. Logs incluidos.
**Fuente:** `bancolombia-cli/src/formatters.ts`

---

## 11. Cligentic blocks en unir-cli — adopción más completa

unir-cli usa el set más completo de bloques cligentic hasta ahora:

**Agent:** `doctor`, `json-mode`, `next-steps`, `skill-installer-prompt`, `trust-ladder`
**Foundation:** `argv`, `atomic-write`, `audit-log`, `banner`, `config`, `error-map`, `global-flags`, `session`, `xdg-paths`

Usar unir-cli como referencia de integración completa de cligentic en CLI de Tipo B.
**Fuente:** `unir-cli/src/cli/agent/` + `unir-cli/src/cli/foundation/`

---

## 12. `--dry-run` en ops mutantes — v0-cli

```typescript
// src/commands/deploy.ts
.option('--dry-run', 'preview the deployment without creating it')
.action(async (opts) => {
  if (opts.dryRun) {
    renderPreview('deploy preview (dry-run)', preview);
    return;  // ← no ejecuta nada
  }
  await createDeployment(opts);
});
```

**Regla:** cualquier comando que modifica estado (crear, borrar, pagar, enviar) → `--dry-run` obligatorio.
**Fuente:** `v0-cli/src/commands/deploy.ts`

---

## Resumen: qué copiar de dónde

| Necesitas | Copiar de |
|-----------|-----------|
| Error handling + isSessionExpired | `bancolombia-cli/src/http.ts` |
| whoami con TTL countdown | `bancolombia-cli/src/commands/whoami.ts` |
| Anti-fraud headers runtime | `bancolombia-cli/src/http.ts` → `bancolombiaHeaders()` |
| Zod response validation | `bancolombia-cli/src/http.ts` → `get<T>(path, config, schema)` |
| REST server maduro | `trii-cli/src/api/app.ts` (Hono) |
| Audit two-phase | `v0-cli/src/lib/audit/jsonl.ts` |
| Cache-first TTL corto | `Cli-propios/interbank-cli/src/commands/login-sync.ts` |
| Browser session persistence | `sunat-cli` + `_knowledge/targets/sunat.md` |
| maskAccount / formato sensible | `bancolombia-cli/src/formatters.ts` |
| Cligentic integración completa | `unir-cli/src/cli/` |
| dry-run en mutantes | `v0-cli/src/commands/deploy.ts` |
