# Esquema de la base de datos (Google Sheets · Capa 1)

Un Google Sheet estructurado en cuatro hojas. Link: [pendiente]

---

## 1. Hoja `facturas`
Tabla larga de persistencia relacional. Una fila por cada factura procesada.

| Columna | Tipo | Origen | Descripción / Reglas |
|---|---|---|---|
| `id` | texto | `{rut_emisor}-{folio}` | Identificador único para detectar duplicados |
| `fecha_recepcion` | fecha-hora | Correo Gmail | Fecha y hora de recepción del correo |
| `rut_emisor` | texto | LLM / XML | RUT del proveedor emisor |
| `razon_social_emisor` | texto | LLM / XML | Razón social del proveedor emisor |
| `folio` | entero | LLM / XML | Número de folio de la factura |
| `fecha_emision` | fecha | LLM / XML | Fecha de emisión (formato AAAA-MM-DD) |
| `condicion_pago` | texto | LLM / XML | Ej. Contado, Crédito 30 días, etc. |
| `monto_neto` | entero | LLM / XML | Monto neto en pesos chilenos |
| `iva` | entero | LLM / XML | Monto de IVA en pesos chilenos |
| `monto_total` | entero | LLM / XML | Monto total en pesos chilenos |
| `glosa` | texto | LLM / XML | Descripción de ítems (separados por " \| ") |
| `numero_oc` | texto | LLM / XML | Número de Orden de Compra (si aplica; sino vacío/null) |
| `servicio` | texto | Cruce catálogo | Servicio asociado según maestro de proveedores |
| `cuenta_contable` | texto | Cruce catálogo | Código de cuenta contable según maestro |
| `origen` | texto | n8n | Origen de extracción: `XML` o `PDF` |
| `estado_validacion` | texto | Nodo validación | Estado: `OK`, `REVISAR` o `DUPLICADA` |
| `motivo_revision` | texto | Nodo validación | Detalle de la inconsistencia detectada |
| `archivo` | texto | Adjunto | Nombre del archivo procesado |

**Reglas de validación:**
- $\text{IVA} = \text{round}(\text{neto} \times 0.19)$ con tolerancia de $\pm 1$.
- $\text{neto} + \text{IVA} = \text{total}$.
- Campos obligatorios no nulos (`rut_emisor`, `folio`, `monto_neto`, `iva`, `monto_total`).
- `id` no repetido previamente en la hoja.

---

## 2. Hoja `proveedores_mensuales`
Catálogo maestro de proveedores recurrentes mensuales (~50 proveedores).

| Columna | Tipo | Descripción |
|---|---|---|
| `rut_emisor` | texto | RUT del proveedor |
| `razon_social` | texto | Razón social de la empresa |
| `servicio` | texto | Categoría o tipo de servicio prestado |
| `cuenta_contable` | texto | Código contable asignado al gasto |
| `patron_facturacion` | texto | Patrón de cobro: `primer_lunes`, `ultimo_dia_habil`, `primeros_5_dias`, `irregular` |
| `contacto_email` | texto | Correo de contacto del proveedor |
| `activo` | texto | `SI` / `NO` para considerar en el cruce de alertas |

---

## 3. Hoja `resumen_mensual`
Pestaña de visualización y control para el usuario final. Reconstruye el formato de matriz solicitado (agrupado por servicio y proveedor, con subfilas de *Neto*, *IVA*, *Fecha*, *N° OC*, *N° Factura* y columnas mensuales para el año en curso).

> [!IMPORTANT]
> Esta hoja es **generada y calculada exclusivamente mediante fórmulas nativas de Google Sheets** (ej. `QUERY`, `FILTER`, `XLOOKUP`). **n8n no escribe directamente en esta hoja**, preservando la separación entre almacenamiento transaccional y capa de visualización.

---

## 4. Hoja `logs`
Registro de auditoría y ejecución de los workflows.

| Columna | Tipo | Descripción |
|---|---|---|
| `timestamp` | fecha-hora | Fecha y hora del evento |
| `workflow` | texto | Flujo origen (`facturas`, `alerta_faltantes`, `errores`) |
| `estado` | texto | `OK` / `ERROR` |
| `detalle` | texto | Mensaje descriptivo o trace de error |
