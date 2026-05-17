# Sectores donde aplica el framework Crafter

## Mapa completo

| Sector | Tipo | Dificultad | Anti-bot | Valor | Legal |
|---|---|---|---|---|---|
| Tributario (SUNAT, SAT, AFIP) | Form-based | Media | Bajo | Alto | Verde |
| AFP / Pensiones | Form-based | Baja | Ninguno | Alto | Verde |
| Registros civiles (RENIEC, SUNARP) | Form-based | Baja | Ninguno | Medio | Verde |
| E-commerce (Rappi, Falabella) | API privada | Baja | Medio | Medio | Gris |
| Salud (EsSalud, clinicas) | Form-based | Baja | Ninguno | Alto | Verde |
| Utilities (luz, agua, telefono) | Form-based | Baja | Bajo | Medio | Verde |
| Logistica (couriers, aduanas) | Mixto | Media | Bajo | Alto | Verde |
| Bancos (cuenta propia) | Form-based | Muy alta | Muy alto | Muy alto | Verde* |
| Bancos (cuentas de terceros) | Form-based | Muy alta | Muy alto | Muy alto | ROJO |

*Verde solo para tu propia cuenta o con consentimiento explicito (modelo Plaid/Belvo)

## Bancos: caso especial

LO QUE SI PUEDES:
- Tu propia cuenta: automatizar conciliacion personal
- Con Open Banking / API oficial (Peru lo esta implementando)
- Empresas que dan consentimiento explicito

LO QUE NO PUEDES:
- Automatizar cuentas de clientes sin licencia de PSP
- Revender acceso a datos bancarios sin regulacion
- Capturar credenciales bancarias de terceros

Tecnicamente: BCP, Interbank, BBVA usan device fingerprinting + behavioral analysis.
El framework necesita trabajo adicional serio para bancos.

## Orden de ataque recomendado

Fase 1 (base existente):
  sunat-cli -> productizar multi-tenant -> cobrar a contadores/freelancers Peru

Fase 2 (mismo framework, nuevo recon):
  SAT Mexico -> mismo producto, mercado 5x mas grande

Fase 3 (expansion natural):
  AFP Peru -> complemento al producto tributario (mismo usuario)
  SUNAT Aduanas -> segmento empresarial, ticket mas alto

Fase 4 (si hay demanda validada):
  Bancos -> solo via Open Banking oficial, NO reverse engineering de terceros
  Excepcion: cuenta propia para conciliacion personal = completamente viable