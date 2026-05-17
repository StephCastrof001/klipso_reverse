# Bancolombia — Research

Tipo: C — Hybrid Banking (OAuth2 + device fingerprinting + Imperva WAF)
Recon via: análisis de código (bancolombia-cli)

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| Login Web | https://svpersonas.apps.bancolombia.com | OAuth2 browser-based | Headless detection: SÍ | Auth visual con 2FA |
| API Gateway | https://canalpersonas-ext.apps.bancolombia.com/super-svp/api/v1 | Bearer Token + device headers | Imperva WAF | Cuentas, transacciones, saldo |
| Proxy local (opcional) | http://localhost:3001 | x-bank-token custom | Ninguno | Wrapper headless |

## Auth Flow

### Mecanismo
- Tipo de auth: OAuth2 browser-based + captura de `sessionTracker` cookie
- Endpoint login: GET `https://svpersonas.apps.bancolombia.com/autenticacion`
- Intercepción: response en `oauth2/token` (POST) → extrae `data.accessToken`
- Dónde se guarda: `~/.bancolombia-config.json`
  ```json
  {
    "mode": "direct",
    "accessToken": "JWT...",
    "cookies": "cookie1=value1; ...",
    "ip": "192.168.1.1",
    "deviceId": "UUID",
    "sessionTracker": "tracking_value",
    "expiresAt": "2026-05-16T..."
  }
  ```
- **Session TTL: 6 minutos** — extremadamente corta, re-auth frecuente

### Gotchas descubiertos en el código
- `device-id` capturado durante la navegación — único por navegador, debe persistir
- `session-tracker` cambia en CADA request — viene en las responses, debe reenviarse en el siguiente
- `request-timestamp` formato exacto: `YYYY-MM-DD HH:MM:SS:mmm` (milisegundos, precisión crítica)
- Solo cookies de dominio `.bancolombia.com` son válidas — las demás se descartan
- **Dos modos**: direct (browser + OAuth2) y api (proxy local headless)
- `expiresAt` calculado como "ahora + 6 min" — Bancolombia puede expirar antes; recomendado re-auth cada 3 min en automatización

## Headers críticos

```
Authorization: Bearer {accessToken}
Content-Type: application/json
message-id: {random-uuid}    ← UUID único por request, NO reutilizable
platform-type: web
request-timestamp: 2026-05-16 14:30:45:123
app-version: 4.2.5
device-id: {captured-device-id}
channel: SVP
ip: {client-ip}
session-tracker: {value-from-previous-response}
device-info: {"device":"Apple","os":"Mac OS","browser":"Chrome"}
Cookie: {captured-cookies}
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...
```

## Endpoints descubiertos

Base: `https://canalpersonas-ext.apps.bancolombia.com/super-svp/api/v1`

| Method | Endpoint | Datos |
|--------|----------|-------|
| GET | `/ch-ms-deposits/hybrid/accounts/customization/consolidated-balance` | Array de cuentas con saldo (`CUENTA_DE_AHORRO`, `CUENTA_CORRIENTE`, `TARJETA_CREDITO`) |
| POST | `/ch-ms-deposits/account/transactions` | Transacciones por rango → body: `{ account: { number, type }, pagination: { key: 1 }, filter: { dateFrom, dateTo } }` — fechas en `DD/MM/YYYY` |

Response accounts:
```json
{
  "data": {
    "accounts": [{ "type": "CUENTA_DE_AHORRO", "number": "123456789", "currency": "COP", "balances": { "available": 500000 } }]
  }
}
```

## Anti-bot / Protecciones

- **Bloquea headless**: SÍ — requiere `--disable-blink-features=AutomationControlled` + `--no-sandbox`
- **WAF**: Imperva — valida `ip` header + User-Agent + cookies de dominio
- **Device binding**: SÍ — `device-id` debe ser estable entre requests (cambio → 401)
- **Timestamp validation**: `request-timestamp` debe estar dentro de ~±5s del servidor
- **Rate limiting**: Session TTL de 6 min sugiere throttling agresivo
- **Locale/timezone**: Contexto del browser fijado a `es-CO` y `America/Bogota`

## Notas críticas de implementación

- `message-id` UUID único por request — no reutilizar
- `session-tracker` viene en responses de Bancolombia — capturar y reenviar en siguiente request
- Date formats: requests usan `DD/MM/YYYY`, almacenamiento interno `YYYY-MM-DD`
- Account type mapping: `CUENTA_CORRIENTE` → `checking`, `CUENTA_DE_AHORRO` → `savings`, `TARJETA_CREDITO` → `credit_card`
- **Playwright flags CRÍTICOS**: `--disable-blink-features=AutomationControlled` sin esto → bloqueado
