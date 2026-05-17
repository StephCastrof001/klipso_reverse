# Banco Cuenta Propia - Automatizacion Personal

## Caso de uso: conciliacion de tu propia cuenta

Legal: 100% verde. Es tu cuenta, tus datos.
Tecnico: el mas dificil del framework por anti-bot bancario.

## Que se puede automatizar

- Descargar movimientos del mes automaticamente
- Exportar a CSV/JSON para contabilidad personal
- Detectar cargos no reconocidos
- Conciliar con facturas emitidas en SUNAT (combo sunat-cli + bank-cli)
- Alertas de saldo bajo
- Resumen mensual de gastos por categoria

## Dificultades tecnicas en bancos peruanos

BCP, Interbank, BBVA, Scotiabank usan:
- Device fingerprinting (canvas, WebGL, fonts)
- Behavioral analysis (velocidad de escritura, movimiento de mouse)
- Session tokens rotativos
- OTP por SMS/email en cada login
- Algunos: CAPTCHA invisible (reCAPTCHA v3)

## Estrategia tecnica viable

Opcion A - headed Chrome con perfil real:
  Usar el mismo perfil de Chrome donde ya tienes sesion activa
  agent-browser --profile ~/.config/google-chrome/Default
  Evita el login completamente si la sesion no expiro

Opcion B - interceptar la app movil:
  Las apps moviles de bancos tienen APIs menos protegidas que la web
  Requiere proxy MITM (mitmproxy) para capturar requests
  Mas trabajo de recon pero endpoints mas limpios

Opcion C - Open Banking (recomendada a futuro):
  Peru esta implementando Open Banking (SBS lo regula)
  BCP y BBVA ya tienen APIs para esto
  Sin reverse engineering, con consentimiento formal

## Combo de mayor valor personal

sunat-cli (emite RHE) + bank-cli (verifica deposito recibido)

Flujo:
1. Cliente te paga
2. bank-cli detecta deposito en cuenta
3. Automaticamente emite RHE con sunat-cli por ese monto
4. Registra en log personal

Esto resuelve el flujo completo del freelancer peruano de 4ta categoria.