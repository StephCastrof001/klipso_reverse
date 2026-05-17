# MercadoLibre CLI — Análisis y Plan

> Basado en reverse engineering de rappi-cli (camilocbarrera) y sunat-cli (crafter-research)
> Fecha: 2026-03-30

---

## Contexto: Por qué MercadoLibre es diferente a Rappi

| Aspecto | rappi-cli | meli-cli |
|---|---|---|
| API | Reverse engineered (privada) | Oficial + documentada |
| Auth | Token robado del browser | OAuth2 real con refresh |
| Endpoints | Sin documentar, frágiles | Versionados, estables |
| app-version header | Commit hash hardcodeado, rompe con deploy | No aplica |
| Token refresh | Manual, rompe | refresh_token automático (6 meses) |
| Fragilidad general | Alta | Baja |

---

## Qué reutilizar de rappi-cli (core agnóstico)

- http.ts — COPIAR DIRECTO (solo cambiar BASE_URL y headers)
- config.ts — COPIAR DIRECTO (cambiar schema Zod)
- ui/spinner, table, colors — COPIAR DIRECTO
- dispatcher pattern de index.ts — ADAPTAR

---

## Endpoints MELI Peru (MPE)

Base URL: https://api.mercadolibre.com

| Operación | Método | Endpoint |
|---|---|---|
| Buscar | GET | /sites/MPE/search?q={query} |
| Detalle item | GET | /items/{item_id} |
| Categorías | GET | /sites/MPE/categories |
| Usuario | GET | /users/me |
| Mis publicaciones | GET | /users/{user_id}/items/search |

---

## Caso de uso real: Monitor de precios Peru

```bash
meli setup                          # OAuth login (una vez)
meli search "iPhone 15 Pro"         # Buscar producto
meli item MPE1234                   # Ver detalle, precio, vendedor
meli watch MPE1234 --below 4000     # Registrar alerta de precio
meli check                          # Ver items watcheados y cambios
```

---

## Decisión pendiente

Objetivo final:
- Automatización personal → CLI primero
- Producto para usuarios → Landing/App Next.js (mismo services/, diferente UI)
- Ambos → CLI primero, luego app reutiliza services

