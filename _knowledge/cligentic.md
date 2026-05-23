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

```bash
# Mínimo viable para CLI agentico
bunx shadcn@latest add https://cligentic.railly.dev/r/json-mode.json
bunx shadcn@latest add https://cligentic.railly.dev/r/next-steps.json
bunx shadcn@latest add https://cligentic.railly.dev/r/error-map.json
bunx shadcn@latest add https://cligentic.railly.dev/r/global-flags.json
bunx shadcn@latest add https://cligentic.railly.dev/r/xdg-paths.json
bunx shadcn@latest add https://cligentic.railly.dev/r/audit-log.json

# Si hay ops mutantes
bunx shadcn@latest add https://cligentic.railly.dev/r/trust-ladder.json

# Siempre recomendado
bunx shadcn@latest add https://cligentic.railly.dev/r/doctor.json

# Si CLI maneja dinero o datos sensibles
bunx shadcn@latest add https://cligentic.railly.dev/r/killswitch.json

# Para Tipo D (API key auth)
bunx shadcn@latest add https://cligentic.railly.dev/r/api-key-wizard.json
```

Los bloques se instalan en `src/cli/<category>/`. Las dependencias entre bloques se resuelven automáticamente via `registryDependencies` — instalar `next-steps` trae `json-mode` sin pedirlo.

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

## Diferencia con ui/ manual (rappi-cli, bancolombia-cli)

Los CLIs anteriores al framework tienen `src/ui/chalk.ts`, `src/ui/spinner.ts`, etc. escritos a mano. cligentic reemplaza eso con bloques estandarizados. Los CLIs nuevos deben usar cligentic en lugar de reinventar ui/.
