# Rappi — Research

Tipo: A — SPA con Bearer token (Android UA spoof, deviceId fingerprint)
Recon via: análisis de código (rappi-cli)

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| Web Colombia | https://www.rappi.com.co | Browser login → Bearer token capture | Android UA spoof, app-version header | Restaurants, markets, pharmacies, products |

API Base: `https://services.grability.rappi.com`

## Auth Flow

### Mecanismo
- Tipo de auth: **Bearer token** (formato: `ft.gAAAAA...`)
- Token capturado via Playwright interceptando `GET /ms/application-user/auth`
- Device token: Header `deviceid` (UUID generado)
- Dónde se guarda: `~/.rappi-config.json` → `{ token, deviceId, lat, lng }`

### Gotchas descubiertos en el código
- User-Agent Android falsificado (`Nexus 5 Build/MRA58N`) — obligatorio
- Header `app-version` es un commit SHA hardcodeado (`e1de6be43aa29091011474615d7ac0810051c36a`) — puede rotar con cada deploy
- `deviceId` UUID debe persistir entre sesiones — ausencia puede detectarse como bot
- Las coordenadas (lat/lng) se actualizan con la dirección activa al login

## Headers críticos

```
authorization: Bearer {token}
deviceid: {uuid}
app-version: e1de6be43aa29091011474615d7ac0810051c36a
x-application-id: rappi-microfront-web/e1de6be43aa29091011474615d7ac0810051c36a
accept-language: es-CO
origin: https://www.rappi.com.co
referer: https://www.rappi.com.co/
user-agent: Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 Chrome/146.0.0.0 Mobile Safari/537.36
sec-ch-ua-mobile: ?1
sec-ch-ua-platform: "Android"
```

## Endpoints descubiertos

### Auth / Usuario
- `GET /ms/application-user/auth` → `{ id, first_name, last_name, email, phone, country_code, vip, loyalty }`
- `GET /api/ms/rappi-prime/is-prime` → `{ is_prime: boolean }`

### Direcciones
- `GET /api/ms/address/reverse-geocode?lat=X&lng=Y`
- `GET /api/ms/users-address/addresses` → `{ addresses: [{ id, tag, address, lat, lng, active, order_count }] }`

### Búsqueda y catálogo
- `POST /api/pns-global-search-api/v1/unified-search?is_prime=false` → body: `{ query, lat, lng }` → `{ stores: [{ store_id, store_name, store_type, products: [...] }] }`
- `POST /api/restaurant-bus/stores/catalog-paged/home` → catálogo paginado por ubicación
- `GET /api/web-gateway/web/stores-router/id/{id}/` → detalles tienda (trailing slash OBLIGATORIO)
- `GET /api/web-gateway/web/restaurants-bus/products/toppings/{store}/{product}/` → `{ categories: [{ min_toppings, max_toppings, toppings: [{ id, description, price }] }] }`

### Carrito
- `PUT /api/ms/shopping-cart/v2/{storeType}/store` — agregar al carrito
- `POST /api/ms/shopping-cart/v1/all/get` — ver carrito
- `POST /api/ms/shopping-cart/v1/{storeType}/recalculate` — recalcular totales
- `GET /api/ms/shopping-cart/v1/{storeType}/checkout/detail` — preview checkout
- `POST /api/ms/shopping-cart-proxy/{storeType}/checkout` — colocar orden
- `PUT /api/ms/shopping-cart/v1/{storeType}/payment-method` — método de pago

### Órdenes
- `GET /api/user-order-home/orders` → `{ active_orders: [{ store: { name, address }, state, eta, place_at }] }`
  - Nota: campo `state`, NO `status`. Valores: `created`, `in_store`, `on_the_way`, `delivered`
  - Productos NO incluidos en response — solo info de tienda

## Anti-bot / Protecciones

- No bloquea headless explícitamente pero requiere simulación completa de Android browser
- `app-version` commit SHA puede cambiar con cada deploy del frontend — punto de fragilidad
- `deviceId` ausente puede detectarse

## Notas críticas de implementación

- **Compound product IDs**: carrito requiere `"storeId_productId"`, no IDs simples
- **Toppings como objetos**: `{ id, description, units, price }` — no IDs planos
- **Triple precio**: el API requiere `price`, `real_price`, `markup_price` — sin ellos pone $0
- **Precios en COP**: directo, sin conversión
