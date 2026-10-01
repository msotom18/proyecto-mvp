# AGENTS.md

Guía de contexto, arquitectura, reglas de trabajo y decisiones técnicas para agentes de IA que colaboren en este repositorio.

---

## 1. Contexto del Proyecto

- **Proyecto:** Proyecto final Track A para la clínica **SGFertility**.
- **Cliente y Usuario inicial:** Fabián (Jefatura en SGFertility). Diseñado para ser extensible posteriormente a las otras 4 personas/jefaturas que procesan facturas en la clínica (reparto exacto pendiente de confirmación).
- **Objetivo:** Automatizar la recepción, extracción de datos, imputación contable, validación y registro de facturas emitidas por proveedores, reduciendo la carga manual y previniendo omisiones de cobros recurrentes.
- **Volumen e impacto:** ~200 facturas al mes, ~5 minutos por factura (~16,7 horas mensuales por persona), valor HH estimado en $25.000 brutos.
- **Flujo de la solución:**
  1. **Recepción:** n8n monitorea Gmail mediante `has:attachment (filename:pdf OR filename:xml)` (sin exigir que el correo esté no leído) y procesa correos con facturas adjuntas en formato PDF o XML (DTE). Tipos de documento soportados: facturas afectas, facturas exentas y notas de crédito (con montos negativos).
  2. **Extracción:**
     - Si el correo incluye archivo **XML**: se leen los datos directamente desde el DTE del SII sin invocar modelos de lenguaje (ahorro de costos/tokens y 100% precisión).
     - Si el correo incluye **únicamente PDF**: se envía el documento binario completo a **Claude Haiku 4.5** (nodo Anthropic de n8n con créditos de n8n Cloud) mediante el prompt estructurado (`src/prompts/extraccion_factura_v3.md`), soportando tanto facturas digitales como documentos escaneados.
  3. **Imputación Contable:**
     - Consulta a `catalogo_proveedores`: si el emisor tiene `clasificacion_unica = SI`, se asignan directamente `cuenta`, `nombre_cuenta`, `concepto` y `ceco` (`clasificacion_origen = CATALOGO`).
     - Si no existe en el catálogo o `clasificacion_unica = NO` (múltiples destinos históricos), se invoca a Claude Haiku 4.5 mediante `src/prompts/clasificacion_v1.md` utilizando el historial del proveedor y opciones válidas de SGFertility como contexto (`clasificacion_origen = LLM`).
  4. **Validación de negocio:**
     - Campos obligatorios no nulos (`rut_emisor`, `folio`, `fecha_emision`, `monto_total`).
     - Validación algorítmica del dígito verificador del RUT del emisor (módulo 11).
     - Factura afecta: $\text{IVA} = \text{round}(\text{neto} \times 0.19) \pm 1$.
     - Consistencia total: $\text{neto} + \text{exento} + \text{IVA} = \text{total} \pm 1$.
     - Consistencia de detalle: $\sum(\text{monto ítems}) = \text{neto} + \text{exento} \pm 1$.
     - Control de duplicados por ID (`{rut_emisor}_{folio}`).
     - Si alguna validación falla, el registro se marca con estado **`REVISAR`** (o `DUPLICADA`) indicando los motivos en `motivo_revision`.
  5. **Persistencia en Google Sheets (Libro Mayor & Control):**
     - `facturas`: Cabecera del documento (una fila por factura/nota de crédito) con metadatos completos, servicio, cuenta contable, ceco, origen (XML/PDF) y estado de validación (`OK` / `REVISAR` / `DUPLICADA`).
     - `facturas_detalle`: Detalle línea a línea (una fila por ítem) adaptado al modelo de **Libro Mayor** de SGFertility (`folio`, `rut_emisor`, `proveedor`, `fecha`, `descripcion`, `monto`, `cuenta`, `nombre_cuenta`, `concepto`, `ceco`).
     - `catalogo_proveedores`: Maestro de consulta con reglas y porcentajes de clasificación por proveedor.
     - `proveedores_mensuales`: Catálogo de ~50 proveedores habituales con su patrón de facturación (`primer_lunes`, `ultimo_dia_habil`, `primeros_5_dias`, `irregular`), servicio y cuenta contable.
     - `resumen_mensual`: Vista histórica por proveedor/servicio con subfilas (Neto, IVA, Fecha, N° OC, N° Factura) calculada por **fórmulas de Google Sheets** (n8n no escribe aquí).
     - `logs`: Registro de auditoría y ejecución de workflows.
  6. **Flujo de alerta mensual:** Ejecución programada diaria a las 18:00 que verifica si es fin de mes, cruza `proveedores_mensuales` activos con `facturas` recibidas en el mes y emite un correo con la tabla HTML: *"Las siguientes empresas no han facturado este mes"*.

---

## 2. Estructura del Repositorio

```text
.
├── AGENTS.md                             # Este archivo: contexto, reglas y directrices para agentes
├── README.md                             # Documentación general, identificación y resumen ejecutivo
├── evidencia/                            # Capturas, diagramas y evidencias de funcionamiento
│   └── .gitkeep
└── src/
    ├── bbdd/                             # Definición de datos y almacenamiento
    │   ├── catalogo_proveedores.csv      # Catálogo maestro de imputación contable
    │   ├── esquema.md                    # Esquema y reglas de las hojas de Google Sheets
    │   ├── proveedores_mensuales.csv     # Catálogo base de proveedores mensuales
    │   └── sheet/                        # CSVs base para importar en Google Sheets
    │       ├── catalogo_proveedores.csv
    │       ├── facturas.csv
    │       ├── facturas_detalle.csv
    │       ├── logs.csv
    │       └── proveedores_mensuales.csv
    ├── evals/                            # Set de evaluación y pruebas de extracción
    │   ├── README.md                     # Descripción del set de evaluación y casos especiales
    │   ├── resultados_esperados.csv      # Ground truth / valores esperados para benchmark
    │   └── facturas/                     # 8 facturas sintéticas de prueba en PDF (F01 a F08)
    ├── flujo/                            # Workflows exportados en JSON para n8n (v1.1 vigentes)
    │   ├── sgf_alerta_proveedores_v1.1.json
    │   └── sgf_ingreso_facturas_v1.1.json
    └── prompts/                          # Prompts para LLM
        ├── clasificacion_v1.md           # Prompt v1 para clasificación contable (cuenta, concepto, CECO)
        ├── extraccion_factura_v1.md      # Prompt v1 base para extracción en JSON
        ├── extraccion_factura_v2.md      # Prompt v2 con soporte para numero_oc
        └── extraccion_factura_v3.md      # Prompt v3 con tipo_documento, exento, vencimiento e items
```

---

## 3. Reglas de Trabajo

1. **Cambios pequeños y uno a la vez:** Realizar modificaciones atómicas e incrementales para mantener el control y facilitar la revisión.
2. **Mostrar el cambio antes de guardarlo:** Presentar siempre la propuesta de cambio o diff antes de escribir o modificar archivos.
3. **Nunca escribir credenciales en el repo:** Prohibido subir API keys, tokens de acceso (Gmail, Google Sheets, LLMs) o contraseñas en el código fuente o archivos versionados.
4. **.env fuera de git:** Toda variable de entorno o secreto debe gestionarse en archivos `.env` ignorados por Git o configurarse directamente en el gestor de credenciales de n8n.
5. **Preservación de workflows exportados:** Los archivos `.json` en `src/flujo/` representan los workflows reales exportados desde n8n y no deben alterarse manualmente a menos que se solicite explícitamente.

---

## 4. Decisiones Tomadas

- **Orquestador (n8n vs. Power Automate):** Se utiliza **n8n** en lugar de Power Automate por ser la tecnología y el stack oficial del curso, permitiendo mayor control, integración mediante JSON y portabilidad de los flujos.
- **Modelo de IA y Créditos de n8n Cloud:** La extracción de PDF y la clasificación contable asistida se ejecutan mediante **Claude Haiku 4.5** (`claude-haiku-4-5-20251001`) usando el nodo nativo de Anthropic en n8n impulsado por los créditos incluidos de n8n Cloud, evitando costos directos de API de OpenAI durante la etapa de desarrollo.
- **Procesamiento de PDF Completo (Digital y Escaneado):** El archivo PDF se transmite de forma binaria completa al modelo de lenguaje multimodal, permitiendo extraer información tanto de facturas con capa de texto digital como de documentos escaneados o imágenes.
- **Trigger de Gmail Amplio:** El gatillo de Gmail busca con la query `has:attachment (filename:pdf OR filename:xml)` procesando cualquier correo con adjuntos válidos, sin restringirse al estado de no leído.
- **Extracción Híbrida (XML directo vs. LLM):** Si el correo incluye XML de factura electrónica (DTE del SII), se parsean los datos directamente mediante código sin llamar al LLM. El LLM se invoca exclusivamente cuando la factura llega solo como PDF.
- **Clasificación Contable en Dos Capas (Catálogo $\rightarrow$ LLM):** La imputación contable se consulta primero en `catalogo_proveedores`. Si `clasificacion_unica = SI`, se asigna de forma determinista inmediata. Solo si no existe o tiene `clasificacion_unica = NO` (ambigüedad histórica), se invoca al LLM mediante `src/prompts/clasificacion_v1.md` usando las alternativas del catálogo como contexto.
- **Persistencia Dual (Cabecera `facturas` + Detalle `facturas_detalle`):** Se separa el almacenamiento entre cabecera de documentos (`facturas`) y detalle por ítem (`facturas_detalle`) para integrarse de forma exacta y transparente con el modelo de **Libro Mayor** de contabilidad de la clínica.
- **Alerta Mensual a Fin de Mes:** La alerta de control de proveedores se ejecuta mensualmente al cierre del mes comprobando si la fecha corresponde al último día del mes, emitiendo el correo: *"Las siguientes empresas no han facturado este mes"*.
- **Persistencia en Tabla Larga y Vista Resumen por Fórmulas:** Los registros se guardan en la hoja `facturas` como tabla relacional limpia (una fila por factura). La visualización matricial solicitada por el usuario se genera en la pestaña `resumen_mensual` utilizando fórmulas nativas de Google Sheets, evitando sobrecargar la lógica de n8n.
- **Gestión de MCPs (Model Context Protocol):** Los servidores MCP se instalan directamente desde la tienda de Antigravity y residen en su configuración global de usuario, no como dependencias locales dentro del repositorio.
