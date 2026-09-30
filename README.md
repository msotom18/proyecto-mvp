# [Nombre del grupo] · Recepción automática de facturas para SGFertility

## 1 · Identificación

- **Grupo:** [pendiente]
- **Integrantes:** Paula Coustasse, Camila Larraín, Martín Mendicute, Fernanda Sánchez, Matías Soto
- **Track:** A — negocio real (clínica SGFertility)
- **Tipo declarado:** Automatización

## 2 · Resumen ejecutivo

[Completar al cierre] Construimos una automatización en n8n que recibe las facturas de proveedores que llegan por correo en PDF o XML, procesa los datos (vía XML directo o extracción con LLM), los valida y los guarda en una base estructurada en Google Sheets, alertando mensualmente a fin de mes si algún proveedor habitual no envió su factura. Está enfocada inicialmente en la jefatura de Fabián en SGFertility (extensible a las otras 4 personas que procesan facturas) y reduce el tiempo mensual de transcripción de ~16,7 horas a prácticamente cero (solo revisión de excepciones).

## 3 · Problema, filtro VRR y solución

**El dolor:** hoy el usuario abre cada factura (PDF o XML) que llega por correo y transcribe manualmente a una matriz en Excel ("Fijo") los campos clave (RUT, folio, neto, IVA, total, glosa, N° OC, fecha), además de controlar manualmente qué proveedores habituales no han facturado. Con un volumen de ~200 facturas al mes a 5 minutos por factura, este proceso consume ~16,7 horas mensuales por persona y es propenso a errores de digitación y omisiones de cobros.

- **Valor:** ~16,7 horas al mes recuperables solo para Fabián, equivalentes a ~$416.667 CLP mensuales brutos (valor HH: $25.000). Potencial acumulado mayor al extenderse al equipo total de 5 personas que procesan facturas (reparto exacto pendiente de confirmación).
- **Repetitividad:** proceso continuo y repetitivo a lo largo de cada mes hábil sobre ~50 proveedores habituales.
- **Reglas claras:** parseo directo de XML o extracción de campos fijos mediante LLM con validaciones aritméticas estrictas ($\text{IVA} = 19\%$ y $\text{neto} + \text{IVA} = \text{total}$).

**Solución:** Sistema automatizado en n8n que procesa correos con facturas adjuntas en PDF/XML, extrae la información requerida, valida consistencia contable, alimenta una tabla relacional en Google Sheets (`facturas`) vinculada a una vista matricial (`resumen_mensual`) calculada por fórmulas, y emite una alerta consolidada a fin de mes por proveedores faltantes.

## 4 · Arquitectura

![Diagrama](evidencia/arquitectura.png) [pendiente]

Correo con PDF/XML → gatillo Gmail en n8n:
- **Si adjunta XML:** lectura y parseo directo de campos DTE (sin costo de LLM).
- **Si adjunta solo PDF:** extracción de texto/imagen → LLM (prompt `src/prompts/extraccion_factura_v2.md`) devuelve JSON estructurado con soporte para `numero_oc`.
- **Nodo de validación:** comprueba IVA 19%, $\text{neto} + \text{IVA} = \text{total}$, campos obligatorios y no duplicidad de ID (`{rut_emisor}-{folio}`).
- **Persistencia en Google Sheets:** registro en tabla larga `facturas` con estado `OK`, `REVISAR` o `DUPLICADA`.
- **Vista resumen:** la pestaña `resumen_mensual` reconstruye automáticamente la vista histórica por proveedor/servicio mediante fórmulas nativas de Google Sheets.
- **Alerta fin de mes:** schedule mensual cruza `proveedores_mensuales` con `facturas` del mes y envía correo consolidado con empresas faltantes.
- **Manejo de errores:** workflow con Error Trigger registra eventos y fallas en `logs`.

## 5 · Las 4 verticales

| Vertical | Capa cumplida | Dónde está la evidencia |
|---|---|---|
| Automatización | [Capa 1] | `src/flujo/` + `evidencia/` |
| IA | [Capa 1] | `src/prompts/extraccion_factura_v2.md` |
| BBDD | [Capa 1] | `src/bbdd/esquema.md` + `src/bbdd/sheet/` |
| Front | [Capa 1] | Sección 6 (Google Sheets: `facturas` + `resumen_mensual`) |

## 6 · Touchpoint del usuario

El usuario mantiene su flujo habitual: los proveedores continúan enviando las facturas por correo. La automatización se activa de forma desatendida, poblando la hoja `facturas` en Google Sheets (el usuario solo debe revisar aquellas marcadas como `REVISAR`) y actualizando dinámicamente la pestaña `resumen_mensual`. A fin de mes, recibe un correo resumen con las empresas que no han emitido su factura en el período.

## 7 · Cómo correrlo

[pendiente] Importar `src/flujo/*.json` en n8n, configurar credenciales de Gmail, Google Sheets y LLM, inicializar las hojas según `src/bbdd/esquema.md` y `src/bbdd/sheet/`, y enviar una factura de prueba al buzón monitoreado.

## 8 · Track A: cliente, métricas y handoff

- **Cliente y contacto:** SGFertility · Fabián · acceso por contacto directo · evidencia en `evidencia/`
- **Antes:**
  - Volumen: ~200 facturas/mes procesadas por Fabián.
  - Tiempo unitario: ~5 min por factura.
  - Dedicación mensual: ~16,7 horas al mes por persona.
  - Equipo total: 5 personas procesan facturas en la clínica (reparto exacto del volumen entre las 5 personas: **pendiente de confirmar**).
  - Valor HH: $25.000 CLP brutos.
  - Costo HH mensual base (Fabián): **$416.667 CLP/mes**.
- **Después:**
  - Tiempo por factura: 0 min en facturas automáticas; ~1-2 min solo en casos excepcionales marcados como `REVISAR`.
- **Cuantificación y ROI:**
  - Ahorro bruto mensual estimado: **~$416.667 CLP/mes** (solo para el alcance inicial de Fabián; escalable a más de $1.5M - $2M CLP/mes al incorporar a las 4 personas restantes).
  - Costo de operación: Tokens LLM (reducido al usar XML directo) + n8n + Google Sheets $\approx$ bajo/marginal.
  - ROI neto altamente positivo desde el primer mes de operación.
- **Handoff:** [quién usa y mantiene la solución, qué cuentas se usan, qué se entrega]

## 9 · Costos de operación

[pendiente] Tokens por factura PDF (LLM) + n8n hosting + Google Sheets API = costo marginal estimado en < $10 USD/mes para el volumen de 200 facturas.

## 10 · Limitaciones y próximos pasos

- Integración directa con casillas Outlook / ERP contable si la clínica migra a Microsoft 365.
- Confirmar y balancear el flujo para las otras 4 personas que procesan facturas.
- OCR avanzado para PDFs escaneados de baja resolución.

## 11 · Roles del equipo

| Integrante | Rol |
|---|---|
| Martín Mendicute | Cuantificación, línea base y ROI |
| [ ] | Flujo n8n + IA |
| [ ] | BBDD |
| [ ] | Repo, README y evidencia |

**Agente de código:** usamos Antigravity configurado con directrices en `AGENTS.md`.
