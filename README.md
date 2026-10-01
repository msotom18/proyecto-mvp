# [Nombre del grupo] · Recepción automática de facturas para SGFertility

## 1 · Identificación

- **Grupo:** [pendiente]
- **Integrantes:** Paula Coustasse, Camila Larraín, Martín Mendicute, Fernanda Sánchez, Matías Soto
- **Track:** A — negocio real (clínica SGFertility)
- **Tipo declarado:** Automatización

---

## 2 · Resumen ejecutivo

Construimos una automatización en n8n que recibe las facturas de proveedores que llegan por correo en PDF o XML, procesa los datos (vía parsing directo de XML o extracción con Claude Haiku 4.5), los valida matemáticamente y los registra en Google Sheets (cabecera en `facturas` y desglose contable en `facturas_detalle`), alertando a fin de mes si algún proveedor recurrente no emitió su cobro. Está orientada inicialmente a la jefatura de Fabián en SGFertility (extensible a las otras 4 jefaturas de la clínica) y reduce el tiempo mensual de transcripción de ~16,7 horas a prácticamente cero (solo revisión de casos excepcionales marcados como `REVISAR`).

---

## 3 · Problema, filtro VRR y solución

**El dolor:** hoy el usuario abre cada factura (PDF o XML) que llega por correo y transcribe manualmente a una matriz en Excel ("Fijo") los campos clave (RUT, folio, neto, IVA, total, glosa, N° OC, fecha), además de controlar manualmente qué proveedores habituales no han facturado. Con un volumen de ~200 facturas al mes a 5 minutos por factura, este proceso consume ~16,7 horas mensuales por persona y es propenso a errores de digitación y omisiones de cobros.

- **Valor:** ~16,7 horas al mes recuperables solo para Fabián, equivalentes a ~$416.667 CLP mensuales brutos (valor HH: $25.000). Potencial acumulado significativamente mayor al extenderse al equipo total de 5 personas que procesan facturas en la clínica (reparto exacto pendiente de confirmación).
- **Repetitividad:** proceso continuo y repetitivo a lo largo de cada mes hábil sobre ~50 proveedores habituales.
- **Reglas claras:** parseo directo de XML o extracción con LLM multimodal sobre campos normalizados, validaciones aritméticas estrictas ($\text{IVA} = 19\%$ y $\text{neto} + \text{IVA} = \text{total}$) y reglas de imputación contable derivadas del historial de la clínica.

**Solución:** Sistema automatizado en n8n compuesto por dos workflows principales que gestionan el ciclo completo de ingreso, validación, imputación al Libro Mayor y control mensual de proveedores.

---

## 4 · Cómo funciona el flujo

La solución opera mediante dos workflows modulares exportados en [`src/flujo/`](file:///c:/Users/Usuario/OneDrive/Desktop/proyecto%20mvp/src/flujo/):

### 1. Workflow de Ingreso de Facturas (`sgf_ingreso_facturas_v1.1.json`)
1. **Gatillo y Recepción:** Un trigger de Gmail monitorea el buzón cada minuto buscando correos con adjuntos PDF o XML (`has:attachment (filename:pdf OR filename:xml)`), sin importar si ya fueron leídos.
2. **Separación y Extracción Híbrida:**
   - Si el correo incluye un archivo **XML** de factura electrónica (DTE del SII), se parsean directamente los tags tributarios mediante código JavaScript sin invocar modelos de lenguaje.
   - Si el correo trae **únicamente PDF**, se envía el archivo binario completo a **Claude Haiku 4.5** (nodo Anthropic de n8n con créditos de n8n Cloud), permitiendo extraer los datos tanto de facturas digitales como de documentos escaneados o imágenes.
3. **Validaciones de Negocio:** En el nodo "Validar y clasificar" se verifican:
   - Presencia de campos obligatorios (`rut_emisor`, `folio`, `fecha_emision`, `monto_total`).
   - Algoritmo de dígito verificador del RUT (módulo 11).
   - Coherencia del IVA del 19% para facturas afectas ($|\text{IVA} - \text{round}(\text{neto} \times 0.19)| \le 1$).
   - Cuadratura aritmética total ($|\text{neto} + \text{exento} + \text{IVA} - \text{total}| \le 1$).
   - Cuadratura de líneas de detalle ($|\sum(\text{monto de ítems}) - (\text{neto} + \text{exento})| \le 1$).
   - Si alguna validación falla, se marca el documento con estado `REVISAR` registrando los motivos específicos.
4. **Imputación Contable en Dos Capas:**
   - Se consulta la hoja `catalogo_proveedores` (maestro derivado del Libro Mayor).
   - Si el proveedor tiene una clasificación única registrada (`clasificacion_unica = SI`), se asignan automáticamente cuenta, concepto y CECO (`CATALOGO`).
   - Si el proveedor tiene múltiples destinos históricos o es nuevo, se deriva a Claude Haiku 4.5 (`LLM`) mediante el prompt de clasificación para determinar la cuenta contable y centro de costo más adecuados.
5. **Persistencia y Trazabilidad:** Se registra la cabecera en la hoja `facturas`, cada ítem o línea en `facturas_detalle` (para el Libro Mayor contable) y el resultado en `logs`.

### 2. Workflow de Alerta Mensual (`sgf_alerta_proveedores_v1.1.json`)
1. **Schedule Diario:** Se ejecuta todos los días a las 18:00 hrs.
2. **Filtro de Fin de Mes:** Comprueba por código si el día actual es el último día del mes corriente (o se ejecuta directamente si es lanzado de forma manual).
3. **Cruce de Datos:** Lee los proveedores habituales activos en `proveedores_mensuales` y las facturas registradas durante el mes en la hoja `facturas`.
4. **Emisión de Alerta:** Si detecta proveedores activos sin factura recibida en el período, genera una tabla HTML y envía un correo consolidado a `gruposfg1@gmail.com` con el asunto *"Proveedores sin factura – AAAA-MM"* y registra el evento en `logs`.

---

## 5 · Estado actual

### Pruebas Realizadas
- **Factura sintética PLANTME 4587 (Prueba Exitosa):** Se procesó de punta a punta la factura sintética de prueba `PLANTME 4587` en formato PDF, logrando:
  - Extracción precisa de todos los campos mediante Claude Haiku 4.5.
  - Validación de RUT, IVA 19%, total e ítems con resultado `OK`.
  - Clasificación unívoca automática vía catálogo (`4510301005` - *Mantención plantas* / *ADMINISTRACION*).
  - Persistencia correcta de cabecera en `facturas` y detalle en `facturas_detalle`.

### Pruebas Pendientes del Set de Evaluación
1. **Factura `F07`:** Documento sintético con error intencional de IVA (no es el 19% del neto) para verificar que el nodo de validación asigne correctamente el estado `REVISAR` con su detalle.
2. **Factura `F08`:** Documento escaneado como imagen (sin capa de texto) para validar la lectura visual multimodal con Claude Haiku 4.5.
3. **Proveedor Fuera de Catálogo:** Prueba de factura de una empresa no registrada en `catalogo_proveedores` para comprobar la rama de clasificación automática asistida por IA (`LLM`).
4. **Control de Duplicados:** Validación de bloqueo/alerta ante el reenvío de un mismo documento (`{rut_emisor}_{folio}`).

---

## 6 · Las 4 verticales

| Vertical | Capa cumplida | Dónde está la evidencia |
|---|---|---|
| Automatización | Capa 1 | `src/flujo/sgf_ingreso_facturas_v1.1.json` y `src/flujo/sgf_alerta_proveedores_v1.1.json` |
| IA | Capa 1 | Claude Haiku 4.5 en `src/prompts/` y nodos Anthropic en workflows |
| BBDD | Capa 1 | `src/bbdd/esquema.md`, `src/bbdd/sheet/` y Google Sheets |
| Front | Capa 1 | Google Sheets (`facturas`, `facturas_detalle` y `resumen_mensual`) |

---

## 7 · Touchpoint del usuario

El usuario mantiene su flujo natural: los proveedores envían las facturas por correo. La automatización procesa los adjuntos de forma desatendida, alimentando el Google Sheet (donde el usuario únicamente debe poner atención en las facturas marcadas como `REVISAR`) y actualizando dinámicamente la vista matricial en `resumen_mensual`. A fin de mes, recibe un correo consolidado con las alertas de proveedores faltantes.

---

## 8 · Cómo correrlo

1. Importar los flujos `src/flujo/*.json` en la instancia de n8n.
2. Configurar credenciales OAuth2 para Gmail y Google Sheets en n8n.
3. Crear el Google Sheet según [src/bbdd/esquema.md](file:///c:/Users/Usuario/OneDrive/Desktop/proyecto%20mvp/src/bbdd/esquema.md) importando las plantillas de [src/bbdd/sheet/](file:///c:/Users/Usuario/OneDrive/Desktop/proyecto%20mvp/src/bbdd/sheet/).
4. Enviar un correo con una factura PDF o XML al buzón monitoreado para verificar el ingreso automático.

---

## 9 · Track A: cliente, métricas y handoff

- **Cliente y contacto:** SGFertility · Fabián · acceso directo.
- **Línea Base (Fabián):** ~200 facturas/mes × 5 min = 16,7 horas al mes ($416.667 CLP/mes brutos base).
- **Alcance ampliado:** 5 personas procesan facturas en la clínica (reparto exacto **pendiente de confirmar**).
- **Ahorro esperado:** Reducción del tiempo de transcripción a solo minutos mensuales dedicados a resolver excepciones (`REVISAR`).
- **Costos de operación:** Prácticamente nulos en desarrollo gracias a créditos de n8n Cloud y parseo directo de XML; costo marginal (< $10 USD/mes) en producción.

---

## 10 · Roles del equipo

| Integrante | Rol |
|---|---|
| Martín Mendicute | Cuantificación, línea base y ROI |
| Matías Soto | Desarrollo flujos n8n, integración IA y BBDD |
| Paula Coustasse | Validación funcional y reglas contables |
| Camila Larraín | Pruebas de evaluación y benchmarks |
| Fernanda Sánchez | Repo, documentación y evidencia |
