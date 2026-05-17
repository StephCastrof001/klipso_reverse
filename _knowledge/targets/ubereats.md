# Uber Eats — Research

Tipo: A — SPA con session cookies (macOS UA spoof, draft order UUID)
Recon via: análisis de código (ubereats-cli)

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| Web | https://www.ubereats.com | Browser login → cookies capture | macOS UA spoof, `sid` cookie detection | Restaurants, items, customizations |

## Auth Flow

### Mecanismo
- Tipo de auth: **Session cookies** (no Bearer token)
- Cookie clave: `sid` (length > 10 → login exitoso)
- Cookies filtradas al dominio `.ubereats.com` y `.uber.com`
- Dónde se guarda: `~/.ubereats-config.json` → `{ cookies (string), lat, lng, draftOrderUuid? }`
- Formato cookies: string concatenado `"name1=value1; name2=value2; ..."`

### Gotchas descubiertos en el código
- No hay token portador único — sesión stateful por cookies HTTP-only
- `draftOrderUuid` persiste en config y se reutiliza — evita crear múltiples carritos
- Cookies expiran; re-login periódico necesario
- Polling cada 3s buscando `sid` con length > 10 para confirmar login

## Headers críticos

```
accept: application/json
content-type: application/json
x-csrf-token: x    ← hardcodeado, el API no valida CSRF en /_p/api/*
origin: https://www.ubereats.com
referer: https://www.ubereats.com/
user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 Chrome/131.0.0.0 Safari/537.36
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "macOS"
cookie: {cookies_string}
```

## Endpoints descubiertos

Base: `https://www.ubereats.com/_p/api/`

### Auth / Usuario
- `POST getProfilesForUserV1` → `{ data: { profiles: [{ uuid, firstName, lastName, email, phoneNumber }] } }`
- `POST getActiveOrdersV1` → órdenes activas (también usado como health check auth)

### Búsqueda y catálogo
- `POST getSearchFeedV1` → body: `{ userQuery, date, startTime, endTime, ... }` → `{ feedItems: [{ store, catalogItems }] }`
- `POST getFeedV1` → home feed restaurantes cercanos
- `POST getStoreV1` → body: `{ storeUuid }` → `{ storeInfo, menu }`
- `POST getMenuItemV1` → body: `{ itemUuid, storeUuid }` → `{ item: { customizationGroups: [{ minPermitted, maxPermitted, options }] } }`

### Carrito (Draft Orders)
- `POST createDraftOrderV2` → body: `{ storeUuid }` → `{ data: { draftOrderUUID } }`
- `POST getDraftOrderByUuidV2` → body: `{ draftOrderUUID }` → `{ data: { shoppingCart: { items, subtotal, deliveryFee, total } } }`
- `POST addItemsToDraftOrderV2` → body: `{ draftOrderUUID, storeUuid, items: [{ menuItemUUID, quantity, selectedCustomizations }] }`
- `POST removeItemsFromDraftOrderV2` → body: `{ draftOrderUUID, shoppingCartItemUUIDs }`
- `POST updateItemInDraftOrderV2` → body: `{ draftOrderUUID, shoppingCartItemUUID, quantity }`

### Checkout y órdenes
- `POST getCheckoutPresentationV1` → body: `{ draftOrderUUID }` → preview con desglose de costos
- `POST checkoutOrdersByDraftOrdersV1` → colocar orden
- `POST getPastOrdersV1` → historial de órdenes

## Anti-bot / Protecciones

- Viewport `420x800` + User-Agent macOS completo
- `x-csrf-token: x` hardcodeado (el endpoint `/_p/api/*` no valida CSRF)
- Sin device fingerprinting (a diferencia de Rappi)
- Sin `deviceId` — solo cookies

## Notas críticas de implementación

- **Precios en centavos**: `1299` = `$12.99` — CRÍTICO, siempre dividir por 100
- **Response estructura inconsistente**: `data.feedItems || feedItems`, `data.shoppingCart || shoppingCart` — el API tiene múltiples versiones coexistiendo
- **Draft order lifecycle**: UUID se borra de config después de checkout exitoso
- **No logout explícito**: se descartan cookies, punto
