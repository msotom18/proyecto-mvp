# Prompt de Clasificación Contable · v1

> [!NOTE]
> **Prompt vigente en n8n:** El texto oficial y operativo de este prompt se construye dinámicamente en el nodo **"Preparar clasificación"** y se envía al nodo **"Clasificar con IA"** del workflow [`src/flujo/sgf_ingreso_facturas_v1.1.json`](file:///c:/Users/Usuario/OneDrive/Desktop/proyecto%20mvp/src/flujo/sgf_ingreso_facturas_v1.1.json), ejecutado sobre el modelo **Claude Haiku 4.5** (`claude-haiku-4-5-20251001`) mediante el nodo nativo de Anthropic en n8n.

---

## Estructura del Prompt (Configurado en nodo "Preparar clasificación")

```text
Clasifica contablemente una factura recibida por la clínica SGFertility.
Proveedor: {{f.razon_social_emisor || 'desconocido'}}
Ítems: {{JSON.stringify((f.items || []).map(x => x.descripcion))}}
Glosa: {{f.glosa || ''}}
Clasificaciones históricas de este proveedor (pueden estar vacías): {{JSON.stringify(candidatos.map(c => ({ cuenta: String(c.cuenta), nombre_cuenta: c.nombre_cuenta, concepto: c.concepto, ceco: c.ceco, pct: c.pct_del_proveedor })))}}
Opciones válidas de cuenta y concepto: {{JSON.stringify(opciones)}}
Elige UNA combinación que exista en las opciones válidas; si los ítems calzan con una clasificación histórica, prefiérela.
Si no hay información de CECO usa "ADMINISTRACION".
Responde SOLO un objeto JSON, sin texto adicional ni bloques de código, con: cuenta, nombre_cuenta, concepto, ceco, confianza (número entre 0 y 1) y justificacion (una frase).
```

---

## Formato JSON de Salida

```json
{
  "cuenta": "4510301005",
  "nombre_cuenta": "613070 MANTENCION Y REPARACION MAINTENANCE",
  "concepto": "Mantención plantas",
  "ceco": "ADMINISTRACION",
  "confianza": 0.95,
  "justificacion": "El proveedor registra mantenciones periódicas para la sede central en administración."
}
```

---

## Reglas de Procesamiento en n8n
- Si `cuenta` viene vacía o nula $\rightarrow$ se agrega motivo de revisión: `"La IA no pudo clasificar la factura"`.
- Si `confianza < 0.6` $\rightarrow$ se agrega motivo de revisión: `"Clasificación IA con baja confianza"`.
- Si el CECO no es identificado con certeza $\rightarrow$ fallback automático a `"ADMINISTRACION"`.
