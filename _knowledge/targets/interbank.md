# Interbank Peru — Research

Tipo: Híbrido C (ECIES P-256 + OTP email 2FA) → arquitectura cache-first
Recon via: análisis de código + RESEARCH.md (2026-04-01/02)
CLI propio: `Cli-propios/interbank-cli/`

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| Banca por Internet | bancaporinternet.interbank.pe | Cookie auth-token JWT HS256 + XSRF-TOKEN | Cloudflare WAF (bloquea Node.js fetch por JA3 mismatch) | Cuentas, movimientos, transferencias, pagos, seguros |

## Auth Flow

**TTL: 5 minutos exactos.** Solución: cache-first (login-sync vuelca todo, queries leen caché).

### Pre-login (sin auth)
```
GET  /bpi/api/excluded/monitor/information     → {fp3LoginEnabled:false}
GET  /bpi/api/excluded/ecies/key/public        → {publicKey:"047420..."} ← P-256 para encriptar
POST /bpi/api/excluded/keyBoardRestService/keys {type:"allNew"}
     → {keyboard:"<base64 73KB>"} ← teclado virtual scrambled, cambia por sesión
```

### Paso 1: Factor 1 (ECIES P-256)
```
POST /bpi/external-api/security/authentication
     req: {iv, publicKey, cipherMessage, mac}  ← credentials encriptados ECIES P-256
     res: {uniqueCode:"0011564321", success:true, data:"<opaque>"}
```

### Paso 2: OTP email
```
POST /bpi/external-api/otp-email/generation  {}
     res: {email:"STE**********@GMAIL.COM"}
```

### Paso 3: Factor 2 (OTP encriptado)
```
POST /bpi/api/auth/login
     req: {iv, publicKey, cipherMessage, mac}  ← OTP encriptado ECIES P-256
     res: {expiration:5, issuedAt:<ts>, uniqueCode, identityNumber}
     cookies: auth-token (JWT HS256), refresh-token (JWT), XSRF-TOKEN (UUID)
```

### Refresh (antes de 5 min)
```
GET /bpi/api/excluded/refresh/token
    res: {expiration:5, issuedAt:<ts>, previousSessionClosed:false}
```

### Headers en llamadas autenticadas
```
x-xsrf-token: <XSRF-TOKEN cookie value>
x-requested-with: XMLHttpRequest
cookie: auth-token=...; XSRF-TOKEN=...; refresh-token=...; cf_clearance=...
```

## Endpoints confirmados

### Cuentas y balances
```
GET  /bpi/api/product/listProductOperations          → id, name, currency, accountNumber, balance, CCI
POST /bpi/api/productRestService/listProduct {}      → lista básica (sin número ni saldo)
GET  /bpi/api/productRestService/products/{id}       → detalle completo
POST /bpi/api/productBalanceInquiryProduct/balanceInquiry {id}
     cuentas:  {accountingBalance, availableBalance}
     tarjetas: {availableCreditLine}
POST /bpi/api/creditCardRestService/additionalInformation {id}
     → {availableCreditLine, creditLine, usedCreditLine}
```

### Movimientos (paginados)
```
POST /bpi/api/productMovementsRestService/movements
     req: {id:"<productId>", pageIndex:""}
     res: {hasMoreMovements:bool, nextKeyPageIndex:"<opaque>", movements:[{
       datetime, description, amount, amountString, currency, referenceNumber
     }]}
     PAGINACIÓN: pasar nextKeyPageIndex en siguiente llamada (string opaco, no número)
```

### Transferencias
```
GET  /bpi/api/transferRestService/outside/ips/verification  → {enabled:true}
POST /bpi/api/commissionRestService/transfer/outside
     req: {amount, cci, currencyCode:"604", idChargeProduct}
     res: comisión de la transferencia
```

### Pagos de servicios
```
POST /bpi/api/companiesRestService/getCompanies {}           → catálogo empresas
POST /bpi/api/companiesRestService/getServicesByCompany      → servicios por empresa
POST /bpi/api/debtRestService/bill                           → monto de deuda a pagar
```

### Seguros
```
GET  /bpi/api/insurance/validation                           → {hasInsurances:true}
GET  /bpi/api/insurance/information
     → {insurances:[{insuranceProductName, insuranceCarrierName, statusName}]}
```

### Perfil
```
POST /bpi/api/userInfoRestService/userInfo {}                → {digitalId, completeName}
POST /bpi/api/profileRestService/all {}                      → {addressDataList, phoneDataList}
```

### Tipo de cambio (sin auth)
```
POST /bpi/api/exchangeRateRestService/exchangeRate {}
     → {exchangeRateSale:3.475, exchangeRatePurchase:3.435, currentDate}
```

## Estado del CLI propio (interbank-cli)

### Implementado ✅
| Comando | Endpoint | Notas |
|---------|----------|-------|
| `login-sync` | Playwright + todos los endpoints | Cache-first, vuelca datos en sesión |
| `accounts` | `listProductOperations` | Lee caché |
| `balance` | `balanceInquiry` | Lee caché |
| `transactions` | `movements` | Lee caché |
| `exchange` | `exchangeRateRestService` | Live, sin auth |
| `profile` | `userInfoRestService` | Lee caché |
| `bills` | `debtRestService` | Implementado |
| `sync` | Re-corre login-sync | Refresh manual |
| `logout` | Borra config local | OK |
| MCP server | `get_accounts`, `get_transactions`, `get_exchange`, `login_status` | Funcional |

### No implementado (endpoints disponibles) ❌
| Qué | Endpoint | Dificultad |
|-----|----------|------------|
| Seguros | `GET /bpi/api/insurance/information` | Baja — solo GET con auth |
| Detalle tarjeta crédito | `POST /bpi/api/creditCardRestService/additionalInformation` | Baja |
| Comisión transferencia | `POST /bpi/api/commissionRestService/transfer/outside` | Baja |
| Paginación completa movimientos | `nextKeyPageIndex` loop | Media — loop hasta `hasMoreMovements:false` |
| Catálogo empresas para pagos | `getCompanies` + `getServicesByCompany` | Baja |

## Problemas del CLI actual

### 1. Stack: Node.js + tsx (debería ser Bun)
```json
// package.json actual — MALO
"scripts": { "start": "npx tsx index.ts" },
"devDependencies": { "tsx": "^4.21.0" }

// Lo que debería ser
"scripts": { "dev": "bun run index.ts" },
// Sin tsx, sin @types/node
```

### 2. Paginación incompleta en login-sync
Solo captura primera página (~20 movimientos). Pendiente: loop con `nextKeyPageIndex` para capturar 3 meses.

### 3. Sin cligentic blocks
Usa chalk manual. Debería usar json-mode, error-map, global-flags, audit-log de cligentic.

### 4. Sin `whoami` command
Sin forma de saber si caché está fresco o cuándo fue el último sync.

## Constraint fundamental

Cloudflare WAF bloquea Node.js fetch (JA3 mismatch). **Browser requerido para TODA llamada** — no solo login. La arquitectura cache-first no es una optimización de velocidad, es la única forma de operar con TTL 5 min + Cloudflare.

## Gotchas críticos

1. TTL = exactamente 5 min. `GET /excluded/refresh/token` cada 4 min si sesión activa.
2. `productId` = SHA-256 64 chars ≠ número de cuenta visible.
3. `moneyBoxId` ≠ `productId` — IDs distintos para la misma cuenta.
4. `listProductOperations` > `listProduct` — incluye número real + saldo en 1 call.
5. `nextKeyPageIndex` es string opaco, no número de página.
6. `X-XSRF-TOKEN` header debe ser idéntico al cookie `XSRF-TOKEN`.
7. Login headless imposible — OTP email requiere Playwright headed.

## Migración a Bun — cambios necesarios

```bash
# 1. Cambiar runtime
bun init  # o editar package.json

# 2. Quitar deps innecesarios
bun remove tsx @types/node

# 3. Instalar cligentic
bunx shadcn@latest add https://cligentic.railly.dev/r/json-mode.json
bunx shadcn@latest add https://cligentic.railly.dev/r/error-map.json
bunx shadcn@latest add https://cligentic.railly.dev/r/global-flags.json
bunx shadcn@latest add https://cligentic.railly.dev/r/audit-log.json
bunx shadcn@latest add https://cligentic.railly.dev/r/xdg-paths.json

# 4. package.json
{
  "scripts": { "dev": "bun run index.ts" },
  "bin": { "interbank": "./index.ts" }
}
```

Archivos a cambiar: `package.json`, `tsconfig.json` (target ES2022+), imports `fs` → `Bun.file()`.
