# Target Types — Klipso Reverse Framework

Taxonomía de targets según cómo está construida la app y qué método de extracción aplica.

---

## Tipo A — API Privada (SPA + Bearer token)

**Cómo está construida la app:**
- React/Vue SPA que llama a una API REST interna
- Auth por Bearer token en header `Authorization`
- Token se genera en el login y se almacena en localStorage/memoria

**Cómo extraer:**
- Playwright intercepta la respuesta del endpoint de auth (`page.on('response', ...)`)
- Se captura el Bearer token en vuelo, antes de que la SPA lo almacene
- Después: HTTP directo con ese token (sin browser)

**Stack CLI:**
- `src/browser/auth.ts` → Playwright intercept del token
- `src/http.ts` → GET/POST directos con Bearer
- Login: una sola vez, token guardado en config

**Repos de referencia:** `rappi-cli`, `ubereats-cli`

**Señales del target para identificarlo:**
- Network tab muestra `Authorization: Bearer eyJ...` en cada request
- Login hace POST a `/auth` o `/login` y devuelve `{ accessToken, refreshToken }`
- La SPA no tiene SSR — todo es fetch del cliente

---

## Tipo B — Forms HTML (browser automation permanente)

**Cómo está construida la app:**
- Portal legacy: PHP, ASP.NET, JSP, o Moodle
- Auth por session cookie (no Bearer)
- Formularios HTML tradicionales con POST
- Puede tener CAPTCHA, input masks, cross-origin iframes

**Cómo extraer:**
- Browser automation permanente (agent-browser o Playwright con `--headed`)
- CDP directo para cross-origin iframes y React/Angular input masks
- cheerio para parsear HTML de respuestas
- La cookie de sesión se renueva automáticamente

**Stack CLI:**
- `src/browser/client.ts` → wrapper de agent-browser
- `src/browser/cdp.ts` → Raw CDP para casos difíciles
- `src/browser/auth.ts` → login con formulario
- `src/browser/captcha.ts` → bypass via mouse coordinates (CDP)

**Repos de referencia:** `sunat-cli`, `whatsapp-cli`, `unir-cli`

**Señales del target:**
- Network tab muestra `Cookie: PHPSESSID=...` o `ASP.NET_SessionId=...`
- Login es un `<form>` HTML con POST a `/login.php` o similar
- Las URLs tienen extensiones: `.asp`, `.php`, `.jsp`
- Headless Chrome falla (ERR_CONNECTION_RESET) → siempre `--headed`

---

## Tipo C — Hybrid Banking (human-in-the-loop)

**Cómo está construida la app:**
- App bancaria con múltiples capas de anti-fraude
- Headers especiales: `device-id`, `session-tracker`, timestamps en milisegundos
- OTP / segundo factor que requiere intervención humana
- Fingerprinting de dispositivo (Akamai, PerimeterX, DataDome)

**Cómo extraer:**
- Fase 1 automática: captura de sesión inicial
- Fase 2 humana: usuario ingresa OTP en browser abierto
- Fase 3 automática: CLI retoma con sesión establecida
- Headers anti-fraude se replican exactamente desde intercepción

**Stack CLI:**
- `src/browser/` → maneja fase 1 y 3
- `src/workflows/` → orquesta el human-in-the-loop
- Config guarda device-id persistente entre sesiones

**Repos de referencia:** `bancolombia-cli`

**Señales del target:**
- Akamai Bot Manager (`/_Incapsula_Resource` o `akam/` en requests)
- Headers con `x-device-fingerprint`, `x-session-id`, `x-request-id` custom
- MFA/OTP obligatorio en cada sesión nueva

---

## Tipo D — API Oficial (OAuth2 / API Key)

**Cómo está construida la app:**
- API documentada o semi-documentada con auth estándar
- OAuth2 PKCE con callback local, o API Key simple en header

**Cómo extraer:**
- OAuth2 PKCE: servidor local en `localhost:3000` captura el `code` del redirect
- API Key: guardada en config, enviada como `Authorization: Bearer <key>`
- No se necesita browser para requests regulares

**Stack CLI:**
- `src/commands/auth.ts` → flujo OAuth + local callback server
- `src/http.ts` → requests directos con API key

**Repos de referencia:** `spoti-cli`, `v0-cli`

**Señales del target:**
- Tiene documentación pública de API
- Login page tiene "Create API Key" o "Developer settings"
- Auth URL tiene `response_type=code&code_challenge=` (PKCE)

---

## Tipo E — Next.js RSC (Server Actions)

**Cómo está construida la app:**
- Next.js App Router con React Server Components
- Mutaciones via Server Actions (POST a la misma URL con hash)
- Respuesta en formato RSC stream (líneas `0:`, `1:`, `2:...`)

**Cómo extraer:**
- Interceptar el POST a la Server Action (URL + `Next-Action: <hash>`)
- Parsear la línea `1:` del stream RSC que contiene el JSON real
- El hash del Server Action cambia con cada deploy → re-recon necesario

**Stack CLI:**
- `src/http.ts` → POST con header `Next-Action: <hash>`
- `src/services/` → parsea RSC stream

**Repos de referencia:** `trii-cli`

**Señales del target:**
- Network tab muestra POST a la misma URL con `Next-Action: <40-char-hash>`
- Response Content-Type: `text/x-component`
- Respuesta empieza con `0:"$@1"\n1:[...]`

---

## Tipo B+D — Dual-Session (dos backends distintos)

**Cómo está construida la app:**
- Dos sistemas independientes integrados (intranet legacy + LMS moderno)
- Cada sistema tiene su propio auth y endpoints
- El usuario necesita acceso a ambos simultáneamente

**Cómo extraer:**
- Dos clients independientes en `src/lib/api/`
- Dos stores de sesión separados
- CLI decide qué client usar según el comando

**Repos de referencia:** `wiener-cli` (ASP intranet + Canvas LMS)

**Señales del target:**
- Dos dominios distintos con auth diferente
- SSO incompleto — el usuario tiene dos passwords
- Un sistema legacy + un LMS/CRM moderno comprado por la empresa

---

## Cómo identificar el tipo de un nuevo target

```
¿Tiene Akamai/PerimeterX/MFA obligatoria?
  → Tipo C

¿La app es Next.js con Server Actions?
  → Tipo E

¿Tiene dos backends con auth diferente?
  → Tipo B+D

¿Tiene API Key o OAuth2 documentado?
  → Tipo D

¿El login devuelve Bearer token (SPA)?
  → Tipo A

¿El login es un <form> HTML con POST?
  → Tipo B
```
