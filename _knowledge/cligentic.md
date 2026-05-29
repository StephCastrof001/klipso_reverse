# cligentic — Meta-framework

Repo: `klipso_reverse/cligentic/` (Railly/cligentic)
Site: https://cligentic.railly.dev
Modelo: shadcn para CLIs — copy-paste sin runtime dependency

## Concepto central

cligentic no es una librería. Es un registry de bloques TypeScript que se copian al proyecto via `bunx shadcn@latest add <url>`. Una vez copiado, el código es tuyo — edítalo libremente, sin lock-in.

```bash
bunx shadcn@latest add https://cligentic.railly.dev/r/<block>.json
# → crea src/cli/<category>/<block>.ts en tu proyecto
```

## Los 22 bloques

### Agent (primitivas de seguridad — must-have para CLIs agenticos)

| Bloque | Descripción | Cuándo instalar |
|--------|-------------|-----------------|
| `json-mode` | Dual output: TTY=human, piped/--json=NDJSON. Disciplina stdout/stderr. | SIEMPRE |
| `next-steps` | Hints estructurados a stderr para que agentes sepan qué ejecutar después | SIEMPRE en CLI agentico |
| `trust-ladder` | Gate T2/T3: confirm prompt en TTY, error estructurado en --json | Si hay ops mutantes o destructivas |
| `error-map` | `AppError` con `code`, `name`, `human`, `hint`. Mapea excepciones HTTP a errores accionables | SIEMPRE |
| `doctor` | Pre-flight checks (deps, auth, conectividad) con diagnóstico estructurado | SIEMPRE |
| `killswitch` | Archivo `~/.app/KILLSWITCH` bloquea todos los writes. Emergencia. | Si CLI maneja datos sensibles o money |
| `api-key-wizard` | First-run wizard para API key: masked prompt + validación live | Para Tipo D (API key auth) |
| `skill-installer-prompt` | Post-init hook que ofrece instalar companion skill para Claude Code / Cursor | Al productizar CLI |

### Foundation (plumbing — obligatorios)

| Bloque | Descripción | Cuándo instalar |
|--------|-------------|-----------------|
| `audit-log` | JSONL append-only, rotación diaria en `~/.config/<app>/audit/YYYY-MM-DD.jsonl` | Ops con escritura |
| `audit-lifecycle` | Wrappers begin/end alrededor de audit-log para ops retryables/peligrosas | Ops T2/T3 |
| `atomic-write` | write→temp→fsync→rename. Sin partial writes en crash | Si escribes config/cache |
| `xdg-paths` | XDG Base Dir Spec: `~/.config`, `~/.local/share`, `~/.cache`. macOS y Windows awareness | SIEMPRE para paths de config |
| `config` | JSON config reader/writer encima de xdg-paths, profile-aware | Si hay login/config persistente |
| `session` | Token persistence: load on boot, save after login, clear on logout | Auth con sesión |
| `error-map` | (también en Agent — instalado una vez, usado en ambos contextos) | — |
| `argv` | POSIX argv parser sin framework. Zero deps. | Si no usas commander/yargs |
| `global-flags` | `--json`, `--dry-run`, `--profile`, `--verbose` estandarizados | SIEMPRE |
| `telemetry` | Opt-out via `CLI_NO_TELEMETRY=1` o `DO_NOT_TRACK=1` | Al productizar |
| `banner` | ASCII wordmark con gradiente. Mostrado en bare invoke o --help | Opcional, cosmético |

### Platform (cross-OS helpers)

| Bloque | Descripción | Cuándo instalar |
|--------|-------------|-----------------|
| `detect` | `isWsl`, `isCi`, `isHeadlessLinux`, `hasCommand`, `detectMode`. Auto-instalado como dep | Auto (dependencia) |
| `open-url` | Abre URL en browser nativo: macOS/Linux/Windows/WSL/CI. Fallback chain. Never throws | OAuth2 PKCE (Tipo D) |
| `copy-clipboard` | pbcopy/xclip/xsel/wl-copy/clip.exe auto-detectado | Copiar tokens/resultados |
| `notify-os` | Desktop notification: osascript/notify-send/powershell | Tareas largas en background |

## Trust Ladder: cuándo usar qué

| Mecanismo | Qué hace | Cuándo |
|-----------|----------|--------|
| `--dry-run` | Simula la operación, muestra preview, NO ejecuta | Ops mutantes reversibles (Fase 2 del preview) |
| T2 gate (`trust-ladder`) | Pide confirmación antes de ejecutar. En --json lanza AppError (no cuelga) | Ops con efecto secundario moderado |
| T3 + intent token | Requiere token single-use generado en paso anterior | Ops destructivas irreversibles (delete, wipe) |
| `killswitch` | Bloquea TODOS los writes via archivo de emergencia | Kill de emergencia sin código |

**Regla**: `--dry-run` ≠ trust ladder. `--dry-run` es un preview; T2/T3 es una autorización. Un CLI bien diseñado tiene ambos en ops destructivas: `--dry-run` primero para ver qué pasaría, T2/T3 para ejecutar.

## Cómo instalar en nuevo CLI

### ✅ Forma rápida — copiar desde el monorepo local (recomendado)

Los bloques ya están en `klipso_reverse/cligentic/registry/`. No hace falta internet ni `bunx shadcn`.

```bash
# Crear estructura de destino
mkdir -p src/cli/agent src/cli/foundation src/cli/platform src/cli/safety

# Copiar mínimo viable (9 bloques obligatorios)
REGISTRY="C:/Users/HP SUPPORT/klipso_reverse/cligentic/registry"
cp "$REGISTRY/agent/json-mode.ts"         src/cli/agent/
cp "$REGISTRY/agent/next-steps.ts"        src/cli/agent/
cp "$REGISTRY/agent/doctor.ts"            src/cli/agent/
cp "$REGISTRY/foundation/error-map.ts"    src/cli/foundation/
cp "$REGISTRY/foundation/global-flags.ts" src/cli/foundation/
cp "$REGISTRY/foundation/xdg-paths.ts"    src/cli/foundation/
cp "$REGISTRY/foundation/audit-log.ts"    src/cli/foundation/
cp "$REGISTRY/platform/detect.ts"         src/cli/platform/   # dep de json-mode

# Condicionales
cp "$REGISTRY/agent/trust-ladder.ts"      src/cli/agent/      # si hay ops mutantes
cp "$REGISTRY/safety/killswitch.ts"       src/cli/safety/     # si maneja dinero/datos sensibles

# Instalar dep de picocolors (json-mode y next-steps lo usan)
bun add picocolors

# Correr lint y auto-fix (los bloques vienen con estilo propio)
bun x biome check --write --unsafe src/cli/
```

### Alternativa — descargar desde registry online

```bash
bunx shadcn@latest add https://cligentic.railly.dev/r/json-mode.json
# ... etc. Más lento, requiere internet. Solo usar si el monorepo no está disponible.
```

Los bloques se instalan en `src/cli/<category>/`. La dependencia `detect.ts` debe copiarse siempre junto con `json-mode.ts`.

## prototype-consumer (case study)

`cligentic/prototype-consumer/` es un CLI de prueba que dogfoodea el registry. Ver ahí para ejemplos de uso real de cada bloque. Incluye patrones de:
- Integrar `trust-ladder` con `commander.js`
- `audit-log` en comandos que hacen writes
- `doctor` con checks custom

## CLIs que usan cligentic (confirmados)

| CLI | Bloques usados |
|-----|---------------|
| v0-cli | audit-log, killswitch, doctor, next-steps, api-key-wizard (copiados a src/lib/trust/) |
| sunat-cli | config, xdg-paths, json-mode, next-steps, telemetry |
| webctl | xdg-paths, config, session |
| broker-cli (privado) | killswitch, audit-log, session, atomic-write, trust-ladder |

## Tests — patrón del framework

**Runner: `bun:test`** (nativo de Bun). No usar vitest.

```bash
# package.json
"test": "bun test"

# Correr
bun test
```

```typescript
// tests/unit/price-extraction.test.ts
import { test, expect } from "bun:test";
import { normalizeVtexSearch } from "../../src/schemas/product.js";

test("extrae precio OH desde Teasers", () => {
  const raw = [/* mock VTEX response */];
  const result = normalizeVtexSearch(raw);
  expect(result[0].prices.oh).toBe(12.50);
});
```

**Qué testear:** funciones puras de schemas y utils — NO comandos, NO servicios con HTTP.
- Extracción de precios (dual Teasers/Installments para OH)
- Normalización de orderForm (centavos → soles)
- Parsing de args CLI
- configAgeLabel(), configExists()

**Referencia:** `v0-cli/tests/unit/` — único CLI del framework con tests (10 unit tests).
bancolombia-cli, rappi-cli, sunat-cli: sin tests.

---

## Diferencia con ui/ manual (rappi-cli, bancolombia-cli)

Los CLIs anteriores al framework tienen `src/ui/chalk.ts`, `src/ui/spinner.ts`, etc. escritos a mano. cligentic reemplaza eso con bloques estandarizados. Los CLIs nuevos deben usar cligentic en lugar de reinventar ui/.
