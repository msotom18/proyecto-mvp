# Evals · extracción de facturas

`facturas/` contiene 8 facturas ficticias con formato chileno y tres diseños distintos. `resultados_esperados.csv` tiene el valor correcto de cada campo para compararlo con lo que devuelve el LLM.

Casos especiales: F07 trae un IVA impreso que no es el 19% del neto (debe quedar REVISAR). F08 es un PDF escaneado sin capa de texto (prueba lectura con visión/OCR). En `proveedores_mensuales.csv`, Seguridad Guardian SpA no tiene factura en el set, por lo que debe gatillar la alerta de faltantes.

Resultados de cada corrida: [pendiente — agregar tabla por versión del prompt con % de campos correctos]
