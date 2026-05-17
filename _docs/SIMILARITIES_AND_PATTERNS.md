# Crafter CLI Family — Similitudes, Metodología y Puntos en Común
> Análisis comparativo de 7 repos: rappi · sunat · whatsapp · bancolombia · ubereats · spoti · trii
> Fecha: 2026-04-11

---

## 1. EL ADN COMPARTIDO — Lo que tienen TODOS sin excepción

Estos patrones aparecen en los 7 repos sin variación:

### A. La misma estructura de carpetas
```
nombre-cli/
├── index.ts          ← dispatcher de comandos
├── server.ts         ← REST API
├── src/
│   ├── constants.ts  ← BASE_URL, ports, timeouts
│   ├── http.ts       ← cliente HTTP typed
│   ├── config.ts     ← load/save con Zod
│   ├── formatters.ts ← moneda, fechas, locale
│   ├── schemas/      ← Zod raw + normalized
│   ├── services/     ← lógica de negocio ← CORAZÓN
│   ├── commands/     ← thin wrappers de services
│   ├── api/app.ts    ← Hono REST routes
│   ├── mcp/index.ts  ← Claude tools
│   └── ui/           ← chalk, spinner, table, banner
```

### B. El mismo stack técnico invariante
| Decisión | Elección | Aparece en |
|---------|---------|-----------|
| Runtime | Bun | 7/7 |
| Lenguaje | TypeScript strict | 7/7 |
| Validación | Zod | 7/7 |
| HTTP server | Hono | 6/7 (spoti no tiene) |
| Claude integration | MCP server | 6/7 (spoti y sunat no tienen) |
| UI colors | Chalk | 7/7 |
| UI spinners | ora/withSpinner | 7/7 |
| UI tablas | cli-table3 | 7/7 |
| Browser automation | Playwright | 6/7 (spoti no necesita) |
| Config format | JSON en home dir | 7/7 |

### C. El mismo patrón de tres interfaces
```
CLI (commands/)  →  |
REST (api/)      →  | → services/ → schemas/ → http/browser
MCP (mcp/)       →  |
```
Un solo `getPositions()` / `search()` / `sendMessage()` que funciona para las tres.
**Nunca hay lógica duplicada entre interfaces.**

### D. Config con Zod validation siempre
```typescript
// Idéntico en los 7 repos, solo cambia el schema
export async function loadConfig(): Promise<AppConfig> {
  if (!existsSync(CONFIG_PATH)) throw new Error("Run: app login");
  const parsed = ConfigSchema.safeParse(JSON.parse(await Bun.file(CONFIG_PATH).text()));
  if (!parsed.success) throw new Error(`Invalid config: ${parsed.error.message}`);
  return parsed.data;
}
```

### E. Brand colors por servicio
Cada repo tiene su propia paleta de colores en `src/ui/chalk.ts`:
```typescript
// rappi-cli
const rappiOrange = chalk.hex("#FF441F");

// bancolombia-cli
const bcoYellow = chalk.hex("#FDDA24");
const bcoBlue   = chalk.hex("#003DA5");

// ubereats-cli
const uberGreen = chalk.hex("#06C167");

// whatsapp-cli
const waGreen = chalk.hex("#25D366");

// trii-cli
const triiTeal = chalk.hex("#00D4AA");
```
Mismo patrón, diferente identidad visual. UX coherente con el servicio que automatizan.

### F. Semantic color helpers idénticos
```typescript
// En los 7 repos, siempre estos helpers:
const ok   = (msg: string) => `  ${success("✓")} ${msg}`;
const fail = (msg: string) => `  ${error("error")} ${msg}`;
const warn = (msg: string) => chalk.yellow(msg);
const dim  = (msg: string) => chalk.dim(msg);
```

### G. Spinner a stderr, datos a stdout
```typescript
// Patrón idéntico en todos:
const spinner = ora({
  stream: process.stderr, // ← SIEMPRE stderr
  isSilent: !process.stderr.isTTY, // ← silencio si pipe
}).start();
```
Esto permite `ubereats cart --output json | jq '.items'` sin romper el spinner.

### H. MCP tools siempre retornan JSON stringify
```typescript
// Idéntico en bancolombia, rappi, ubereats, whatsapp, trii:
return {
  content: [{
    type: "text",
    text: JSON.stringify(data, null, 2)
  }]
};
```

### I. Playwright siempre con los mismos flags anti-detección
```typescript
// Aparece en rappi, bancolombia, ubereats, whatsapp, trii:
chromium.launch({
  headless: false, // o true según el caso
  channel: "chrome",
  args: [
    "--disable-blink-features=AutomationControlled",
    "--no-sandbox",
  ],
});
// + userAgent spoofed como Chrome en macOS
```

---

## 2. EVOLUCIÓN DEL FRAMEWORK — Cómo maduró versión a versión

```
rappi-cli (base)
    ↓ agrega TypedError, dual-mode auth
bancolombia-cli
    ↓ agrega commander.js, audit trail, dry-run, agent-first IO, CDP
sunat-cli
    ↓ agrega persistent context, TUI completo, prefetch, polling
whatsapp-cli
    ↓ mismo patrón que rappi pero más maduro, Promise.allSettled
ubereats-cli
    ↓ agrega PKCE, monorepo, tsup build, scope-based validation
spoti-cli
    ↓ agrega RSC parsing, HAR recording, fee computation, safety caps
trii-cli (más maduro)
```

### Lo que se agregó en cada generación:

| Característica | Introducida en |
|---------------|---------------|
| dispatcher + services + MCP + REST | rappi-cli |
| TypedError con `isSessionExpired` | bancolombia-cli |
| Dual-mode config (direct \| api proxy) | bancolombia-cli |
| `--dry-run` en mutaciones | sunat-cli |
| `--output json/table/auto` | sunat-cli |
| Audit trail JSONL | sunat-cli |
| Commander.js subcommands | sunat-cli |
| CDP raw WebSocket para iframes | sunat-cli |
| Persistent browser context | whatsapp-cli |
| TUI con polling en tiempo real | whatsapp-cli |
| Message deduplication via Set | whatsapp-cli |
| `Promise.allSettled` para calls paralelas | ubereats-cli |
| Draft order UUID en config | ubereats-cli |
| OAuth2 PKCE sin client_secret | spoti-cli |
| Callback server local con force-close | spoti-cli |
| Scope-based command validation | spoti-cli |
| RSC stream parsing (línea `1:`) | trii-cli |
| HAR recording durante recon | trii-cli |
| Fee computation client-side | trii-cli |
| Safety cap + `--yolo` bypass | trii-cli |
| Post-placement order verification | trii-cli |

---

## 3. METODOLOGÍA COMPARTIDA — El proceso siempre igual

### Paso 0: Decidir el tipo de target
```
¿API pública con OAuth2?   → Tipo D (spoti-cli como referencia)
¿Next.js con Server Actions? → Tipo E (trii-cli como referencia)
¿Banco con anti-fraude?    → Tipo C (bancolombia-cli como referencia)
¿Solo forms HTML?          → Tipo B (sunat-cli como referencia)
¿App móvil/web con API?    → Tipo A (rappi-cli como referencia)
```

### Paso 1: RECON primero, siempre
1. `chromium.launch({ headless: false, recordHar: { path: "~/.target/capture.har" } })`
2. Hacer el flujo completo manualmente mientras el CLI observa
3. Extraer del HAR: endpoints, headers, tokens, cookies, payloads
4. Identificar anti-bot: ¿Cloudflare? ¿Akamai? ¿Imperva? ¿reCAPTCHA?

### Paso 2: RESEARCH.md antes del primer archivo .ts
```markdown
## Portal Map        ← URLs + auth + CAPTCHA
## Auth Flow         ← el bypass discovery más valioso
## Endpoints         ← método, payload, respuesta real
## Gotchas           ← lo que rompió durante el recon
## Headers críticos  ← los que fallar = 403/401
```

### Paso 3: Git desde el primer archivo
```bash
git init
git add RESEARCH.md
git commit -m "docs: RESEARCH.md - recon inicial de {target}"
# A partir de aquí: un commit por comando que pasa smoke test
```

### Paso 4: Scaffold base (copiar de repo más cercano)
- `src/ui/` completo → copiar sin cambios, solo brand colors
- `src/config.ts` → copiar, cambiar schema Zod
- `src/http.ts` → copiar, cambiar BASE_URL y buildHeaders()
- `src/constants.ts` → nueva desde cero

### Paso 5: Implementar en este orden
1. Auth (login command) → smoke test manual
2. Primer read command (whoami, balance, etc.)
3. REST API (Hono routes para los reads)
4. MCP server (tools de los reads)
5. Comandos mutantes con `--dry-run`
6. Audit trail

---

## 4. PUNTOS EN COMÚN TÉCNICOS — Los patrones que se repiten

### 4.1 Raw → Normalized siempre en 2 capas
**Todos los repos separan** el schema de la API (frágil) del schema interno (estable):
```typescript
// Capa 1: Raw — lo que devuelve la API (puede cambiar)
const RawOrderSchema = z.object({
  date_creation: z.string(),  // snake_case, nullable, passthrough
  type_transaction: z.string().nullish(),
}).passthrough();

// Capa 2: Normalized — contrato estable del CLI
const OrderSchema = z.object({
  date: z.string(),           // camelCase, requerido, tipado estricto
  side: z.enum(["buy","sell"]),
});

// En el service: mapear raw → normalized
return RawSchema.parse(json).map(raw => NormalizedSchema.parse({
  date: raw.date_creation,
  side: raw.type === "BUY" ? "buy" : "sell",
}));
```

### 4.2 Dual-mode auth en todos los que tienen servidor proxy
rappi, bancolombia, trii — los tres tienen el mismo patrón:
```typescript
// Discriminated union: el service no sabe en qué modo está
const ConfigSchema = z.discriminatedUnion("mode", [
  z.object({ mode: z.literal("direct"), cookies: z.string(), ... }),
  z.object({ mode: z.literal("api"), apiUrl: z.string(), token: z.string() }),
]);

// El service dispara según el modo
if (config.mode === "api") return apiProxyCall(path, args, config);
return directApiCall(path, args, config);
```

### 4.3 Session expiry detection multi-idioma
Todos los que tocan servicios en español/colombiano detectan errores en español:
```typescript
function isExpiredError(body: string): boolean {
  const lower = body.toLowerCase();
  return (
    lower.includes("token inv") ||      // "token inválido"
    lower.includes("sesión") ||         // "sesión expirada"
    lower.includes("sin actividad") ||  // (bancolombia)
    lower.includes("unauthorized") ||
    lower.includes("session expired")
  );
}
```

### 4.4 withSpinner como HOF universal
```typescript
// Todos los repos tienen esta función, prácticamente idéntica:
async function withSpinner<T>(message: string, fn: () => Promise<T>): Promise<T> {
  const spinner = ora({ text: brandColor(message), stream: process.stderr }).start();
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

### 4.5 Error con actionable hint siempre
Nunca un stack trace crudo al usuario:
```typescript
// Patrón idéntico en todos:
const hint = expired ? ` Run '${CLI_NAME} login' to reconnect.` : "";
throw new AppError(`${message}${hint}`, statusCode, expired);
```

### 4.6 Masking de datos sensibles en output
```typescript
// Aparece en bancolombia, trii, sunat:
function maskAccount(number: string): string {
  return "****" + number.slice(-4);
}
// Pero en --output json la API devuelve el valor completo para automatización
```

### 4.7 Locale-aware formatting
Todos los que manejan moneda local la formatean con el locale correcto:
```typescript
// bancolombia + trii (COP colombiano)
`$${amount.toLocaleString("es-CO")}`

// ubereats (USD centavos)
amount.toLocaleString("en-US", { minimumFractionDigits: 2 })

// sunat (PEN soles peruanos)
`S/ ${amount.toFixed(2)}`
```

### 4.8 Operaciones mutantes siempre con confirmación
```typescript
// Seco en trii, bancolombia, sunat — mismo patrón:
if (!opts.confirm) {
  printPreview(preview);
  console.log(`\n  Dry run. Add --confirm to execute.\n`);
  return;
}
// [opcional] interactivo
const answer = await readline.question('Type "yes" to confirm: ');
if (answer !== "yes") process.exit(0);
```

---

## 5. LO QUE VARÍA — Solo lo específico del dominio

Lo único que cambia entre repos es **el conocimiento del dominio**:

| Dominio | Conocimiento específico |
|---------|------------------------|
| Bancolombia | device-id + session-tracker + milisegundos en timestamp |
| SUNAT | SOL viejo sin CAPTCHA → redirect a Nueva Plataforma |
| Rappi | app-version con commit hash rota por deploy |
| Trii | Action hashes rotan por deploy, RSC línea "1:", fee formula |
| Uber Eats | draftOrderUuid en config, precios en centavos |
| Spotify | PKCE, refresh proactivo 60s antes, scopes por comando |
| WhatsApp | Aria labels en inglés y español, polling 3s con dedup |

**Conclusión**: El framework es siempre el mismo. Lo que se invierte tiempo aprendiendo es el target.

---

## 6. CÓMO GENERAN LOS CLI — El proceso exacto paso a paso

### Fase 0 — Recon con HAR (30-60 min)
```typescript
// Grabar HAR completo durante el login y flujos principales
const context = await browser.newContext({
  recordHar: { path: "~/.app/capture.har", content: "embed" }
});
// El HAR tiene: todos los requests, headers, payloads, responses
// Analizarlo con: jq, har-analyzer, o abrirlo en Chrome DevTools
```

### Fase 1 — Extraer del HAR (30 min)
1. Filtrar requests al dominio target
2. Identificar el endpoint de auth (token o cookies)
3. Copiar los headers completos de un request autenticado
4. Guardar ejemplos de response para definir schemas Zod

### Fase 2 — Scaffolding (1 hora)
```bash
mkdir my-cli && cd my-cli
bun init
# Copiar ui/, config.ts template, http.ts template
git init && git add . && git commit -m "chore: scaffold"
```

### Fase 3 — Auth command (1-2 horas)
- Abrir browser con Playwright
- Capturar token/cookies durante login real del usuario
- Guardar en `~/.my-config.json`
- Smoke test: `bun run src/commands/login.ts`

### Fase 4 — Primer read command (30 min por comando)
```
service → schema → command → ui formatting
```

### Fase 5 — REST + MCP (1-2 horas)
- Copiar estructura de `api/app.ts` de cualquier repo
- Copiar estructura de `mcp/index.ts` de rappi o trii
- Reemplazar por tus services

### Fase 6 — Comandos mutantes con safety (30 min cada uno)
- Preview/dry-run primero
- Confirmación interactiva
- Post-execution verification

---

## 7. TABLA RESUMEN FINAL

| Aspecto | rappi | sunat | whatsapp | bancolombia | ubereats | spoti | trii |
|---------|-------|-------|----------|-------------|----------|-------|------|
| Tipo | A | B | B | C | A | D | E |
| Browser | login | siempre | siempre | login | login | callback | login |
| MCP tools | 14 | ❌ | 5 | 6 | 16 | ❌ | 9 |
| REST API | ✅ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Dry-run | ❌ | ✅ | N/A | ❌ | ❌ | N/A | ✅ |
| Audit trail | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Output json | ✅ | ✅ | parcial | ✅ | ✅ | ✅ | ✅ |
| Dual-mode auth | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| Token refresh | ❌ | N/A | N/A | manual | ❌ | automático | manual |
| Commander.js | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Safety caps | ❌ | ❌ | N/A | ❌ | ❌ | N/A | ✅ |

---

## 8. LAS 5 LECCIONES QUE ENSEÑAN TODOS

1. **Services layer primero** — Si los services están bien, CLI + REST + MCP son gratis
2. **RESEARCH.md antes de código** — El conocimiento del recon ES el diferencial, no el código
3. **Zod en frontera de entrada** — Nunca confiar en la API, siempre parsear y validar
4. **Spinners a stderr** — Permite pipes sin romper la UX
5. **El dominio es lo único que varía** — El framework es siempre el mismo

---

## 9. CÓMO REPLICARLO EN UN TARGET NUEVO (Guía de 1 página)

```
1. Identifica el tipo (A/B/C/D/E) con la tabla de la Sección 1
2. git clone del repo más similar como referencia
3. Abre el target con browser + HAR recording, haz el flujo manual
4. Escribe RESEARCH.md con lo que encontraste
5. Copia ui/, config.ts, http.ts del repo referencia
6. Implementa login command que captura token/cookies
7. Un service por operación (getX, createX, deleteX)
8. Commands thin (10 líneas que llaman al service)
9. Agrega REST y MCP copiando la estructura existente
10. --dry-run en todo lo que muta
```

Costo: **1-3 días por target nuevo** si el framework ya está internalizado.
