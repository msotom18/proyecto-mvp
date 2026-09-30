# AGENTS.md

Guía de contexto, arquitectura, reglas de trabajo y decisiones técnicas para agentes de IA que colaboren en este repositorio.

---

## 1. Contexto del Proyecto

- **Proyecto:** Proyecto final Track A para la clínica **SGFertility**.
- **Cliente y Usuario inicial:** Fabián (Jefatura en SGFertility). Diseñado para ser extensible posteriormente a las otras 4 personas/jefaturas que procesan facturas en la clínica (reparto exacto pendiente de confirmación).
- **Objetivo:** Automatizar la recepción, extracción de datos, validación y registro de facturas emitidas por proveedores, reduciendo la carga manual y previniendo omisiones de cobros recurrentes.
- **Volumen e impacto:** ~200 facturas al mes, ~5 minutos por factura (~16,7 horas mensuales por persona), valor HH estimado en $25.000 brutos.
- **Flujo de la solución:**
  1. **Recepción:** n8n monitorea una casilla de Gmail y procesa correos entrantes con facturas adjuntas en formato PDF o XML (DTE).
  2. **Extracción:**
     - Si el correo incluye archivo **XML**: se leen los datos directamente desde el DTE sin invocar modelos de lenguaje (ahorro de costos/tokens y 100% precisión).
     - Si el correo incluye **únicamente PDF**: se envía el texto/imagen al LLM mediante el prompt estructurado (`src/prompts/extraccion_factura_v2.md`) para extraer en JSON: `rut_emisor`, `razon_social_emisor`, `folio`, `fecha_emision`, `condicion_pago`, `monto_neto`, `iva`, `monto_total`, `glosa`, `numero_oc` y `observaciones`.
  3. **Validación de negocio:**
     - $\text{IVA} = \text{round}(\text{neto} \times 0.19)$ (con tolerancia de $\pm 1$)
     - $\text{neto} + \text{IVA} = \text{total}$
     - Campos obligatorios no nulos y no duplicidad de ID (`{rut_emisor}-{folio}`).
     - Si alguna validación falla, el registro se marca con estado **`REVISAR`** (o `DUPLICADA`) indicando el motivo en `motivo_revision`.
  4. **Persistencia en Google Sheets:**
     - `facturas`: Tabla larga (una fila por factura procesada) con metadatos completos, servicio, cuenta contable, origen (XML/PDF) y estado de validación (`OK` / `REVISAR` / `DUPLICADA`).
     - `proveedores_mensuales`: Catálogo de ~50 proveedores habituales con su patrón de facturación (`primer_lunes`, `ultimo_dia_habil`, `primeros_5_dias`, `irregular`), servicio y cuenta contable.
     - `resumen_mensual`: Pestaña con formato amigable que reconstruye la vista histórica por proveedor/servicio con subfilas (Neto, IVA, Fecha, N° OC, N° Factura) y columnas mensuales mediante **fórmulas de Google Sheets** (n8n no escribe en esta hoja).
     - `logs`: Registro de auditoría y ejecución de workflows.
  5. **Flujo de alerta mensual:** Ejecución programada al cierre de cada mes que cruza `proveedores_mensuales` con `facturas` recibidas en el mes y emite un correo con el listado consolidado: *"Las siguientes empresas no han facturado este mes"*.

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
    │   ├── esquema.md                    # Esquema y reglas de las hojas de Google Sheets
    │   ├── proveedores_mensuales.csv     # Catálogo base de proveedores mensuales
    │   └── sheet/                        # CSVs base para importar en Google Sheets
    │       ├── facturas.csv
    │       ├── logs.csv
    │       └── proveedores_mensuales.csv
    ├── evals/                            # Set de evaluación y pruebas de extracción
    │   ├── README.md                     # Descripción del set de evaluación y casos especiales
    │   ├── resultados_esperados.csv      # Ground truth / valores esperados para benchmark
    │   └── facturas/                     # 8 facturas sintéticas de prueba en PDF (F01 a F08)
    ├── flujo/                            # Workflows exportados en JSON para n8n
    │   └── .gitkeep
    └── prompts/                          # Prompts para LLM
        ├── extraccion_factura_v1.md      # Prompt v1 base para extracción en JSON
        └── extraccion_factura_v2.md      # Prompt v2 con soporte para numero_oc
```

---

## 3. Reglas de Trabajo

1. **Cambios pequeños y uno a la vez:** Realizar modificaciones atómicas e incrementales para mantener el control y facilitar la revisión.
2. **Mostrar el cambio antes de guardarlo:** Presentar siempre la propuesta de cambio o diff antes de escribir o modificar archivos.
3. **Nunca escribir credenciales en el repo:** Prohibido subir API keys, tokens de acceso (Gmail, Google Sheets, LLMs) o contraseñas en el código fuente o archivos versionados.
4. **.env fuera de git:** Toda variable de entorno o secreto debe gestionarse en archivos `.env` ignorados por Git o configurarse directamente en el gestor de credenciales de n8n.

---

## 4. Decisiones Tomadas

- **Orquestador (n8n vs. Power Automate):** Se utiliza **n8n** en lugar de Power Automate por ser la tecnología y el stack oficial del curso, permitiendo mayor control, integración mediante JSON y portabilidad de los flujos.
- **Gestión de MCPs (Model Context Protocol):** Los servidores MCP se instalan directamente desde la tienda de Antigravity y residen en su configuración global de usuario, no como dependencias locales dentro del repositorio.
- **Extracción Híbrida (XML directo vs. LLM):** Si el correo incluye XML de factura electrónica (DTE), se extrae la información directamente mediante parsing XML sin llamar al LLM. El LLM se invoca exclusivamente cuando la factura llega solo como PDF.
- **Alerta Mensual a Fin de Mes:** La alerta de control de proveedores se ejecuta mensualmente al cierre del mes (reemplazando la lógica de alertas diarias) emitiendo el mensaje *"Las siguientes empresas no han facturado este mes"*.
- **Persistencia en Tabla Larga y Vista Resumen por Fórmulas:** Los registros se guardan en la hoja `facturas` como tabla relacional limpia (una fila por factura). La visualización matricial solicitada por el usuario se genera en la pestaña `resumen_mensual` utilizando fórmulas nativas de Google Sheets, evitando sobrecargar la lógica de n8n.
