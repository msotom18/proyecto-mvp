# Esquema de la base de datos (Google Sheets · Capa 1)

Estructura de almacenamiento, consulta y control contable en Google Sheets para el procesamiento de facturas en **SGFertility**.

---

## 1. Hoja `facturas` (Cabecera por Documento)
Tabla relacional principal. Registra una fila por cada documento procesado (factura afecta, factura exenta o nota de crédito).

| Columna | Tipo | Origen | Descripción / Reglas |
|---|---|---|---|
| `id` | texto | `{rut_emisor}-{folio}` | Identificador único para control de duplicados |
| `fecha_recepcion` | fecha-hora | Correo Gmail | Fecha y hora de recepción del correo |
| `tipo_documento` | texto | LLM / XML | `factura_afecta`, `factura_exenta`, `nota_credito` |
| `rut_emisor` | texto | LLM / XML | RUT del proveedor emisor |
| `razon_social_emisor` | texto | LLM / XML | Razón social del emisor |
| `folio` | entero | LLM / XML | Número de folio del documento |
| `fecha_emision` | fecha | LLM / XML | Fecha de emisión (formato AAAA-MM-DD) |
| `fecha_vencimiento` | fecha | LLM / XML | Fecha de vencimiento (AAAA-MM-DD o vacío si no aplica) |
| `condicion_pago` | texto | LLM / XML | Ej. Contado, Crédito 30 días, etc. |
| `monto_neto` | entero | LLM / XML | Monto neto en pesos chilenos |
| `monto_exento` | entero | LLM / XML | Monto exento en pesos chilenos (o 0 si no aplica) |
| `iva` | entero | LLM / XML | Monto de IVA en pesos chilenos |
| `monto_total` | entero | LLM / XML | Monto total en pesos chilenos (positivo en facturas, negativo en NC) |
| `glosa` | texto | LLM / XML | Resumen o concatenación de ítems (" \| ") |
| `numero_oc` | texto | LLM / XML | Número de Orden de Compra (si aplica; sino vacío/null) |
| `servicio` | texto | Catálogo | Servicio asociado según maestro de proveedores |
| `cuenta` | texto | Catálogo / LLM | Código contable asignado (ej. `4510301005`) |
| `concepto` | texto | Catálogo / LLM | Concepto contable (ej. `Mantención`, `Utiles de oficina`) |
| `ceco` | texto | Catálogo / LLM | Centro de Costo (ej. `ADMINISTRACION`, `LABORATORIO FIV`) |
| `clasificacion_origen` | texto | Sistema | Origen de la imputación: `CATALOGO` o `LLM` |
| `origen` | texto | n8n | Origen de lectura: `XML` o `PDF` |
| `estado_validacion` | texto | Nodo validación | `OK`, `REVISAR` o `DUPLICADA` |
| `motivo_revision` | texto | Nodo validación | Detalle de inconsistencia detectada |
| `archivo` | texto | Adjunto | Nombre del archivo adjunto procesado |

**Reglas de validación:**
- Para `factura_afecta`: $\text{IVA} = \text{round}(\text{neto} \times 0.19) \pm 1$ y $\text{neto} + \text{exento} + \text{IVA} = \text{total}$.
- Para `factura_exenta`: $\text{IVA} = 0$ y $\text{monto\_exento} = \text{total}$.
- Para `nota_credito`: Monto registrado con signo negativo para el libro mayor.
- Campos obligatorios no nulos (`rut_emisor`, `folio`, `monto_total`).
- `id` no repetido previamente en la hoja.

---

## 2. Hoja `facturas_detalle` (Línea por Ítem · Libro Mayor)
Refleja la estructura del **Libro Mayor** de contabilidad de SGFertility (una fila por cada ítem/línea de factura).

| Columna | Tipo | Origen | Descripción |
|---|---|---|---|
| `folio` | entero | `facturas` | Número de folio de la factura |
| `rut_emisor` | texto | `facturas` | RUT del proveedor |
| `proveedor` | texto | `facturas` | Razón social del proveedor |
| `fecha` | fecha | `facturas` | Fecha de emisión de la factura |
| `descripcion` | texto | LLM / XML | Descripción individual del ítem/servicio |
| `monto` | entero | LLM / XML | Monto del ítem (negativo si es nota de crédito) |
| `cuenta` | texto | Catálogo / LLM | Código de la cuenta contable |
| `nombre_cuenta` | texto | Catálogo / LLM | Nombre oficial de la cuenta contable |
| `concepto` | texto | Catálogo / LLM | Concepto contable |
| `ceco` | texto | Catálogo / LLM | Centro de Costo (CECO) |

---

## 3. Hoja `catalogo_proveedores` (Maestro de Imputación Contable)
Pestaña de consulta generada a partir del historial contable de SGFertility. Permite clasificar automáticamente los documentos de proveedores frecuentes.

| Columna | Tipo | Descripción |
|---|---|---|
| `proveedor` | texto | Razón social del proveedor |
| `cuenta` | texto | Código contable (ej. `4510301005`) |
| `nombre_cuenta` | texto | Descripción completa de la cuenta contable |
| `concepto` | texto | Concepto asignado al gasto |
| `ceco` | texto | Centro de Costo imputado |
| `n_facturas` | entero | Cantidad histórica de facturas |
| `n_lineas` | entero | Cantidad histórica de líneas/ítems |
| `ultima_fecha` | fecha | Fecha del último registro histórico |
| `pct_del_proveedor` | decimal | Proporción de apariciones bajo esta combinación (0.0 a 1.0) |
| `clasificacion_unica` | texto | `SI` (100% de certeza histórica) / `NO` (múltiples destinos posibles) |

**Lógica de clasificación:**
1. Se busca el proveedor en `catalogo_proveedores`.
2. Si existe y `clasificacion_unica = SI`: se asigna directamente `cuenta`, `nombre_cuenta`, `concepto` y `ceco`, marcando `clasificacion_origen = CATALOGO`.
3. Si `clasificacion_unica = NO` o el proveedor no existe: se invoca el LLM con el prompt `src/prompts/clasificacion_v1.md`, marcando `clasificacion_origen = LLM`.

---

## 4. Hoja `proveedores_mensuales` (Control Recurrente)
Catálogo maestro de proveedores con facturación mensual periódica (~50 proveedores).

| Columna | Tipo | Descripción |
|---|---|---|
| `rut_emisor` | texto | RUT del proveedor |
| `razon_social` | texto | Razón social |
| `servicio` | texto | Categoría o tipo de servicio prestado |
| `cuenta_contable` | texto | Código contable asignado |
| `patron_facturacion` | texto | `primer_lunes`, `ultimo_dia_habil`, `primeros_5_dias`, `irregular` |
| `contacto_email` | texto | Correo de contacto del proveedor |
| `activo` | texto | `SI` / `NO` para cruce de alertas |

---

## 5. Hoja `resumen_mensual` (Vista Usuario Final)
Pestaña de visualización que reconstruye la vista histórica matricial (un bloque por servicio/proveedor con subfilas *Neto*, *IVA*, *Fecha*, *N° OC*, *N° Factura* y columnas mensuales para el año en curso).

> [!IMPORTANT]
> Esta hoja es **generada y calculada exclusivamente mediante fórmulas nativas de Google Sheets** (ej. `QUERY`, `FILTER`, `XLOOKUP`). **n8n no escribe en esta hoja**.

---

## 6. Hoja `logs` (Auditoría)
Registro de auditoría y ejecución de los workflows.

| Columna | Tipo | Descripción |
|---|---|---|
| `timestamp` | fecha-hora | Fecha y hora del evento |
| `workflow` | texto | Flujo origen (`facturas`, `alerta_faltantes`, `errores`) |
| `estado` | texto | `OK` / `ERROR` |
| `detalle` | texto | Mensaje descriptivo o trace de error |
