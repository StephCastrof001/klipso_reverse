# Trii — Research

Tipo: E — Next.js RSC con Server Actions (hash volátil + Cloudflare WAF)
Recon via: análisis de código (trii-cli)

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| Trii Web (SPA) | https://web.trii.co | Bearer token + sk2 cookie (2FA) | Cloudflare challenge | Portfolio, órdenes, dividendos, movimientos |
| Market Data | https://api-multi.trii.ws | Ninguno (público) | Rate limiting | Velas (candles), datos bursátiles |

## Auth Flow

### Mecanismo
- Tipo de auth: Multi-step 2FA con TOTP/SMS
  - Paso 1: email + password → `twoFactorToken` + flag `requires_2fa`
  - Paso 2: TOTP/SMS code + `twoFactorToken` → cookie `sk2` (session key)
- Dónde se guarda: `~/.trii-config.json`
  - Modo direct: `{ accessToken, cookies, expiresAt }`
  - Modo api (proxy): `{ token (JWT), apiUrl, expiresAt }`
- **Session TTL**: ~1 hora (asumido — Trii no lo expone explícitamente)

### Server Actions de auth
```
LOGIN_STEP_1:
  POST https://web.trii.co/ (hash: 707c99395b31ddc8a319dd470ba2c1d74540263633)
  Body: [email, password]
  Response (RSC line "1:"): { data: { requires_2fa: bool, twoFactorToken: string }, error: null }

LOGIN_STEP_2:
  POST https://web.trii.co/verify (hash: 40dc16798443735becc69baa2f22a315c23e2ecd91)
  Body: [twoFactorToken, code]
  Response: sets sk2 cookie
```

### Gotchas CRÍTICOS
- **Los hashes cambian con cada deploy de Trii** — están hardcodeados en `src/constants.ts` y se rompen al actualizar el frontend
- **2FA es obligatorio** — siempre. No hay bypass
- `twoFactorToken` es transitorio — solo válido para el paso 2
- Si Cloudflare bloquea → response empieza con `"<"` (HTML) en vez de RSC

## Headers críticos

```
Accept: text/x-component
Content-Type: text/plain;charset=UTF-8
Next-Action: <hash>    ← específico por Server Action, VOLÁTIL
Next-Router-State-Tree: <URL-encoded JSON router tree>
Origin: https://web.trii.co
Referer: https://web.trii.co<path>
Cookie: sk2=...; user=...; cf_clearance=...
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...
sec-ch-ua: "Chromium";v="146", "Not-A.Brand";v="24", "Google Chrome";v="146"
sec-fetch-dest: empty
sec-fetch-mode: cors
sec-fetch-site: same-origin
```

Para market data (público):
```
Accept: application/json
Referer: https://web.trii.co/
```

## Server Actions descubiertos

Todos hacen POST a `https://web.trii.co/<path>` con `Next-Action: <hash>`:

| Action | URL path | Hash | Body | Response (line "1:") |
|--------|----------|------|------|----------------------|
| Posiciones | /movements/account | 001a6518707c64a352aaa6cabd85d352f591237097 | [] | `{ data: [positions...] }` |
| Transacciones | /movements/account | 409276d806f637954118518e7b51308f32f51bb21f | [] | `{ data: [transactions...] }` |
| Balance | /cashin | 0007ee7370fdd8fa7cafefe7d2a06490de49944dee | [] | `{ data: { cash, titles, funds, us_cash } }` |
| Órdenes | /dashboard | 40793437e04963e0bf48a6fc2d22ca51f1b6e1a482 | [{ filter: "all" }] | `{ data: [orders...] }` |
| Dividendos | /dashboard | 607f3acfcfa0a78fe4a505794b1300ebc60d84fe9c | [{ start_date, end_date, country }] | `{ data: [dividends...] }` |
| Trade | /dashboard | 409d5ecdd4338c0ae1d82dbe2ecb2abab4c95c0f50 | [{ type, stock, quantity, type_order, limit_price, fee, platform: "web" }] | `{ data: { response: "ok" } }` |

## RSC Stream Parsing

```typescript
// Response es plain text, no JSON
const lines = (await res.text()).split('\n');
const match = lines.find(l => /^1:(.*)$/.test(l));
const data = JSON.parse(match!.slice(2)); // Quitar "1:"

// Si Cloudflare bloqueó:
if (text.startsWith('<')) throw new CloudflareBlockError();
```

## Next-Router-State-Tree

Cada Server Action necesita este header — JSON que describe la ruta actual en el árbol Next.js:
```
["(main)", { "children": [...nested segments...] }, null, null, true]
```
El CLI construye esto dinámicamente según el `path` del Server Action.

## Anti-bot / Protecciones

- **Cloudflare WAF**: retorna HTML challenge si detecta bot — respuesta empieza con `"<"`
- `cf_clearance` cookie: Cloudflare verification token, persistente en sesión
- `sec-*` headers: Trii verifica presencia y valores
- Session expiry: `sk2` expira en ~1 hora → 401/403

## Notas críticas de implementación

- **Hashes rotan**: monitorear deploys de Trii con Playwright → interceptar headers `Next-Action` en network para actualizar `src/constants.ts`
- **Modo capture** (`trii login`): abre browser con HAR recording, busca headers auth + JWT-shaped values en localStorage
- **Dual mode**: direct (Playwright + Server Actions directos) vs api (proxy local en localhost:3001)
- Market data (`api-multi.trii.ws`): no requiere auth — headers simples
