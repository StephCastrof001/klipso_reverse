# klipso_reverse — CLAUDE.md

Este repo es una fábrica de CLIs de extracción de APIs privadas.
Objetivo: dado un target (portal web / app), producir un CLI funcional con MCP server.

---

## Antes de tocar código

1. Leer `_knowledge/_knowledge/README.md` — índice de targets + árbol de decisión de tipos
2. Identificar el tipo del target nuevo (A/B/C/D/E/B+D) — ver árbol abajo
3. Leer el `.md` del target más similar en `_knowledge/targets/`
4. Crear `RESEARCH.md` en el nuevo repo **antes de escribir una sola línea de código**

Regla: sin RESEARCH.md completo → no hay implementación.

---

## Árbol de decisión: tipo de target

```
¿Tiene Akamai/PerimeterX/MFA obligatoria?
  → Tipo C — ref: bancolombia-cli

¿Next.js con Server Actions (header Next-Action)?
  → Tipo E — ref: trii-cli

¿Dos backends con auth diferente?
  → Tipo B+D — ref: wiener-cli

¿API Key o OAuth2 PKCE documentado?
  → Tipo D — ref: v0-cli (API key) | spoti-cli (OAuth2 PKCE)

¿Login devuelve Bearer token (SPA + fetch)?
  → Tipo A — ref: rappi-cli | ubereats-cli

¿Login es <form> HTML con POST / Moodle / portal legacy?
  → Tipo B — ref: sunat-cli (más maduro) | unir-cli | whatsapp-cli
```

---

## Stack invariante (todos los CLIs)

- Runtime: **Bun** + TypeScript
- Linter: **Biome** (`biome check src/`)
- UI primitivas: bloques de **cligentic** (NO copiar ui/ de repos viejos — ver abajo)
- MCP server: `src/mcp/index.ts` expone comandos como tools para Claude Code
- Output: siempre `--output json/table/auto` (agent-first IO)
- Comandos mutantes: siempre `--dry-run`

---

## Bloques cligentic obligatorios (instalar primero)

```bash
bunx shadcn@latest add https://cligentic.railly.dev/r/json-mode.json
bunx shadcn@latest add https://cligentic.railly.dev/r/next-steps.json
bunx shadcn@latest add https://cligentic.railly.dev/r/error-map.json
bunx shadcn@latest add https://cligentic.railly.dev/r/global-flags.json
bunx shadcn@latest add https://cligentic.railly.dev/r/xdg-paths.json
bunx shadcn@latest add https://cligentic.railly.dev/r/audit-log.json
bunx shadcn@latest add https://cligentic.railly.dev/r/doctor.json
```

Condicionales:
```bash
# Tipos A/B/C/D con ops destructivas
bunx shadcn@latest add https://cligentic.railly.dev/r/trust-ladder.json
bunx shadcn@latest add https://cligentic.railly.dev/r/killswitch.json

# Tipo D con API key
bunx shadcn@latest add https://cligentic.railly.dev/r/api-key-wizard.json

# OAuth2 PKCE (Tipo D)
bunx shadcn@latest add https://cligentic.railly.dev/r/open-url.json
```

Doc completo de bloques: `_knowledge/cligentic.md`

---

## Proceso por fase

### Fase 1 — Recon (output: RESEARCH.md)
- Abrir target en browser con network tab
- Capturar: auth flow completo, cookies/tokens, headers especiales, endpoints
- Detectar anti-bot: headless block, WAF, rate limiting
- Documentar gotchas numerados
- Plantilla: `_knowledge/framework/TEMPLATE.md` → sección RESEARCH.md template

### Fase 2 — Scaffold
- Crear repo en `Cli-propios/<nombre>-cli/`
- Instalar bloques cligentic (lista arriba)
- Copiar estructura de carpetas del CLI de referencia para ese tipo
- Plantilla detallada: `_knowledge/framework/TEMPLATE.md` → Fase 2

### Fase 3 — Implementar por tipo

| Tipo | Patrón clave | Archivo a copiar/adaptar |
|------|-------------|--------------------------|
| A | `page.on('response')` intercept Bearer token | `rappi-cli/src/browser/auth.ts` |
| B | Form POST + session cookie + CDP para input masks | `sunat-cli/src/browser/` |
| C | Human-in-the-loop + device fingerprint headers | `bancolombia-cli/src/workflows/` |
| D (API key) | api-key-wizard + http.ts directo | `v0-cli/src/lib/trust/` + `src/lib/api/` |
| D (OAuth2) | Local callback server `localhost:8888` | `spoti-cli/src/commands/auth.ts` |
| E | POST con `Next-Action: <hash>`, parsear línea `1:` del RSC stream | `trii-cli/src/http.ts` |
| B+D | Dos clients independientes, dos session stores | `wiener-cli/src/lib/api/` |

### Fase 4 — Checklist antes de publicar
Ver `_knowledge/framework/TEMPLATE.md` → Fase 4 (checklist completo)

---

## Estructura de archivos de un CLI terminado

```
<nombre>-cli/
  index.ts          ← dispatcher (Record<cmd, filepath> + Bun.spawn)
  server.ts         ← REST API (Bun.serve wrapping services/)
  RESEARCH.md       ← recon del portal (NUNCA borrar)
  CONTEXT.md        ← instrucciones para AI agents
  package.json      ← bin: { "<nombre>": "./index.ts" }
  src/
    config.ts       ← loadConfig/saveConfig/removeConfig + Zod
    constants.ts    ← BASE_URL, CONFIG_FILENAME
    http.ts         ← get/post/put/del + TypedError + parseErrorBody
    schemas/        ← Zod por dominio
    services/       ← lógica de negocio
    validation/
      input.ts      ← validar inputs antes de API
    commands/       ← UN archivo por comando
    browser/        ← solo Tipos A/B/C
    workflows/      ← solo Tipos B/C
    ui/             ← REEMPLAZADO por bloques cligentic
    mcp/
      index.ts      ← MCP server
    cli/            ← bloques cligentic instalados aquí
```

---

## Anti-patrones — NO hacer

| Anti-patrón | Por qué falla | Corrección |
|-------------|---------------|------------|
| Copiar `src/ui/` de repos viejos | Reinventa lo que cligentic resuelve | Instalar bloques cligentic |
| Hardcodear XPaths o IDs de DOM | Rompe con cada deploy del frontend | Buscar por `text` / `type` / atributos semánticos |
| Hardcodear `app-version` o `accept-language` | Rompe con cada release | Extraer dinámicamente o rotar |
| Sin RESEARCH.md antes de codear | Endpoints desconocidos, auth incompleta | Recon primero, siempre |
| Sin `--dry-run` en ops mutantes | Usuario no puede hacer preview | Obligatorio |
| Sin `--output json` | CLI no usable por agentes | Usar json-mode de cligentic |
| Leer todo el HTML cuando hay REST API | Frágil, lento | Usar la REST API si existe |
| Sin `TypedError` en http.ts | Errores no manejables por agentes | Copiar patrón de bancolombia-cli |

---

## Dónde está el conocimiento

| Qué buscar | Dónde |
|------------|-------|
| Tipo de target + árbol decisión | `_knowledge/framework/TYPES.md` |
| Proceso completo nuevo CLI | `_knowledge/framework/TEMPLATE.md` |
| Auth flow + endpoints de un target | `_knowledge/targets/<nombre>.md` |
| Bloques cligentic (cuáles, cuándo) | `_knowledge/cligentic.md` |
| Metodología completa (7 repos originales) | `_docs/MASTER_METHODOLOGY.md` |
| WAF bypass lessons | memoria: `reference_waf_lessons.md` |
| Todos los targets en una tabla | `_knowledge/README.md` |

---

## CLIs de referencia disponibles en este repo

```
rappi-cli/          Tipo A — más simple, base del framework
ubereats-cli/       Tipo A — cookies en vez de Bearer
bancolombia-cli/    Tipo C — TypedError, Zod v4, dual-mode config
sunat-cli/          Tipo B — el más maduro, template base para B
whatsapp-cli/       Tipo B — QR scan, sesión browser permanente
unir-cli/           Tipo B — Moodle + Panopto, watch+notify
spoti-cli/          Tipo D — OAuth2 PKCE completo
trii-cli/           Tipo E — RSC stream parser
wiener-cli/         Tipo B+D — dual-session, dos clients independientes
v0-cli/             Tipo D — API key + trust ladder T0-T3 + intent tokens + background workers
cligentic/          META — 22 bloques copy-paste, instalar en todo CLI nuevo
```

CLIs propios (producción): `Cli-propios/`
