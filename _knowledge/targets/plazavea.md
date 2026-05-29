# Plaza Vea Peru — Research

Tipo: B (Cookie de sesión vía browser Playwright)
Recon via: RESEARCH.md + testing en vivo (2026-05-29)
CLI propio: `Cli-propios/plazavea-cli/` — https://github.com/StephCastrof001/cli_plazavea1
Plataforma: **VTEX headless** (e-commerce SaaS)

---

## Portal Map

| Portal | URL Base | Auth Method | Anti-bot | Contenido |
|--------|----------|-------------|----------|-----------|
| Tienda | tienda.plazavea.com.pe | Cookie `VtexIdclientAutCookie_plazavea` | Sin WAF notable | Search, cart, orderForm, simulate |
| OMS | www.plazavea.com.pe | Misma cookie (dominio `.plazavea.com.pe`) | — | Historial de pedidos |

⚠ **Doble host obligatorio.** Los endpoints de órdenes viven en `www`, no en `tienda`. Usar el mismo cookie en ambos funciona porque el dominio es `.plazavea.com.pe` (wildcard).

---

## Auth Flow

**Cookie de login real: `VtexIdclientAutCookie_plazavea`** (también variante con UUID: `VtexIdclientAutCookie_<uuid>`).

⚠ **`vtex_session` NO es auth** — aparece para visitantes anónimos (tracking). Detectar login con `vtex_session` guarda una sesión anónima que da 401 en órdenes.

### Método
1. Playwright abre `tienda.plazavea.com.pe/login` con `headless: false`
2. Usuario inicia sesión manualmente
3. CLI poléa `context.cookies()` hasta encontrar `VtexIdclientAutCookie*`
4. Guarda todas las cookies del dominio en `~/.config/plazavea/session.json`

### TTL
Por confirmar con uso real. Cookie VtexId suele durar horas/días (no minutos como banks). Observar con `plaza whoami`.

### Fallback login
```bash
plaza login --manual "<header Cookie: completo desde DevTools Network tab>"
```

### ⚠ Playwright bajo Bun en Windows
`chromium.launch()` / `connectOverCDP()` cuelgan bajo Bun (el cliente WS/CDP no completa el handshake). No es antivirus — ocurre con Defender apagado. Fix: correr login bajo **Node + tsx**. El dispatcher detecta `command === "login"` y lo rutea a `node node_modules/tsx/dist/cli.mjs`.

---

## Price Schema — Bug conocido de VTEX

Plaza Vea tiene hasta 3 niveles de precio por producto:

| Campo | Nombre en tienda | Cuándo aparece |
|-------|-----------------|----------------|
| `regular` | Precio tachado / sin descuento | Siempre (`commertialOffer.ListPrice`) |
| `led` | Precio descuento sin tarjeta | `commertialOffer.Price < ListPrice` |
| `oh` | Precio Tarjeta OH | Solo en campañas activas |

### El bug: OH price en dos lugares
VTEX codifica el precio OH en DOS lugares distintos según la campaña activa:

**Lugar 1 — Teasers:**
```json
"Teasers": [{ "<Name>k__BackingField": "Tarjeta oh! S/ 12.50 ..." }]
```

**Lugar 2 — Installments:**
```json
"Installments": [{ "Name": "¡Precio Exclusivo Tarjeta oh! S/ 12.50" }]
```

Buscar en ambos, usar el primero que encuentre. Ver `src/schemas/product.ts` → `extractOhPrice()`.

---

## Endpoints confirmados

⚠ Host `tienda` vs `www` — crítico para orders.

| Host | Endpoint | Método | Auth | Descripción |
|------|----------|--------|------|-------------|
| tienda | `/api/catalog_system/pub/products/search/{term}?_from=0&_to=49` | GET | No | Búsqueda |
| tienda | `/api/checkout/pub/orderForm` | GET | Semi | Ver/crear carrito |
| tienda | `/api/checkout/pub/orderForm/{id}/items` | POST | Sí | Agregar ítem |
| tienda | `/api/checkout/pub/orderForm/{id}/items/update` | POST | Sí | Actualizar/eliminar |
| tienda | `/api/checkout/pub/orderForms/simulation` | POST | No | Stock por postal |
| **www** | `/api/oms/user/orders?page=1&per_page=N` | GET | Sí | **Historial pedidos** |
| ~~www~~ | ~~`/api/oms/pvt/orders`~~ | — | — | ❌ Siempre 401 (necesita API key, no cookie) |

---

## Schema de Orders (`/api/oms/user/orders`)

Respuesta: `{ list: Order[], paging, facets, stats }`.

Campos clave por orden:
- `orderId`, `creationDate`, `clientName`, `status`, `statusDescription`
- **`totalValue`** (centavos → ÷100 = soles). ⚠ El campo NO se llama `value`.
- `totalItems`, `currencyCode` ("PEN"), `items` (puede ser `null` en lista)

---

## Stock — Bug conocido (reportado por usuario)

**Problema:** search API devuelve stock GLOBAL (suma de todos los locales del país). Al hacer checkout, VTEX verifica stock del local asignado a la dirección → puede estar agotado aunque global diga disponible.

**Solución v3 (3 capas):**
- A: columna Stock en `search` muestra `⚠ global`
- B: `add` verifica `availability === "withoutStock"` post-add → warning
- C: `simulate --sku X --postal Y` llama `/api/checkout/pub/orderForms/simulation`

**Endpoint simulation:**
```
POST /api/checkout/pub/orderForms/simulation
Body: { items: [{id, quantity, seller}], postalCode: "15001", country: "PER" }
```
No modifica carrito. Auth: no requerida (endpoint público).

---

## Gotchas numerados

1. `commertialOffer.Price` ya viene en soles (ej: 15.90). orderForm usa centavos (1590 = S/ 15.90) — normalizar con ÷100.
2. Seller `"1"` = Plaza Vea directo. Otros = terceros en marketplace — filtrar por seller "1" para precio oficial.
3. orderForm se crea automáticamente al hacer GET — no requiere POST previo.
4. `(x2)`, `(x3)` en nombre del producto = multiplicador VTEX (pack) — limpiar con regex si se necesita el nombre limpio.
5. `commertialOffer.AvailableQuantity > 0 && IsAvailable` = stock GLOBAL, no local.
6. orders `totalValue` en centavos; el campo no se llama `value`.
7. `vtex_session` es anónimo — NO detectar login por esta cookie.
8. Playwright cuelga bajo Bun en Windows → login bajo Node+tsx.
