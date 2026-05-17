# WhatsApp Web — Research

Tipo: B — SPA puro, DOM automation permanente (sin API REST)
Recon via: análisis de código (whatsapp-cli)

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| WhatsApp Web | https://web.whatsapp.com | QR scan (browser visible) | Chrome automation detection | Chat list, mensajes, historial |

## Auth Flow

### Mecanismo
- Tipo de auth: **Browser session persistence** (Playwright persistent context)
- No hay tokens — la sesión vive en IndexedDB + cookies del browser
- Dónde se guarda: 
  - `~/.whatsapp-cli/browser-state/` — contexto completo de Playwright (cookies, localStorage, IndexedDB)
  - `~/.whatsapp-config.json` → `{ pushName, connectedAt, browserStateDir }`

### Flujo de login
1. `whatsapp login` → abre browser **visible** (`headless: false`)
2. WhatsApp Web carga y muestra QR code
3. Usuario escanea QR con su teléfono
4. CLI detecta `div[aria-label="Chat list"]` → login exitoso
5. Extrae `pushName` de `header span[title]` → guarda en config
6. **Timeout: 5 minutos** (`LOGIN_TIMEOUT_MS = 300_000`)
7. Comandos posteriores: reutilizan browser state con `headless: true`

### Gotchas descubiertos en el código
- La sesión vive en IndexedDB — no es un token transferible
- Selector `div[aria-label="Chat list"]` es frágil — puede romperse si WhatsApp cambia markup
- Browser state se guarda en disco → re-login solo cuando WhatsApp revoca la sesión
- **No hay API REST** — todo es DOM automation (clicks, text extraction)

## Headers críticos

Ninguno explícito — todo ocurre a nivel de sesión del navegador (cookies de WhatsApp).

## Endpoints / Selectores DOM

No hay API REST. El CLI usa selectores DOM:

| Selector | Para qué |
|----------|----------|
| `div[aria-label="Chat list"]` | Detectar login exitoso |
| `header span[title]` | Extraer pushName del usuario |

## Anti-bot / Protecciones

- `--disable-blink-features=AutomationControlled` — elimina `navigator.webdriver = true`
- `--no-sandbox` — permisos Chrome en algunos entornos
- User-Agent Chrome real (no Playwright default)
- Viewport fijo `1280x900` — evita detección de viewport no estándar
- Locale `en-US`

## Notas críticas de implementación

- **Persistent context**: `launchPersistentContext()` reutiliza el estado guardado entre invocaciones del CLI
- Login: headed (`headless: false`). Comandos: headless (`headless: true`) con estado anterior
- No hay logout controlado — simplemente se elimina el browser-state directory
- **Fragilidad DOM**: cualquier cambio de markup en WhatsApp Web rompe los selectores
- Sin API REST — el CLI es dependiente de que WhatsApp Web no cambie su UI
