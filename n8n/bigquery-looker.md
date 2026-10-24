# Análisis de datos: BigQuery y Looker Studio

Estado: **diseño listo (2026-10-24); ejecución por fases**. Campos del CRM y del sitio que lo hacen posible: [../crm/campos-analitica.md](../crm/campos-analitica.md).

## Principio
BigQuery es una **base de análisis sin identidad**: nunca recibe nombre, teléfono, correo, mensajes, notas libres, calle/número ni IP. El CRM conserva la identidad y la llave para volver a ella. Todo se une por `ref_code` y por un `cliente_hash` interno.

## 1. Qué conectar y cómo (todo al mismo dataset/región)
| Fuente | Cómo | Cuándo |
|---|---|---|
| **GA4** | Admin › Vinculación con BigQuery, exportación **diaria** (no streaming: tiene costo). Trae `lead_generado` con `lead_id` = `ref_code` y `user_pseudo_id`. | **Ya.** No tiene histórico: cada día sin activar se pierde. |
| **Search Console** | Configuración › Exportación masiva de datos a BigQuery (tablas `searchdata_url_impression`, `searchdata_site_impression`). Requiere dar permisos a la cuenta de servicio que indica Google. | **Ya.** Tampoco rellena histórico. |
| **Google Ads** | BigQuery Data Transfer Service (conector Google Ads, diario; permite rellenar histórico reciente). Une por `gclid` (vista de clics) con el lead. | Cuando haya campañas activas. |
| **Meta Ads** | Sin conexión nativa a BigQuery: n8n con la API de Marketing, o conector de pago en Looker. | Después. |
| **CRM** | n8n → BigQuery (sección 2). | Cuando el CRM tenga datos reales. |
| **Catálogo del sitio** | n8n lee `GET /propiedades` (ya sin metas privadas) y guarda foto diaria de precio/estado por `eb_id` → historial de precios por zona. | Después. |

Región: elegir **una** y usarla en todos los datasets (no se puede cambiar; las uniones exigen misma región). Verificar al crear que GA4, Search Console y el conector de Ads la aceptan; multi-región US es la opción más compatible.

## 2. CRM → BigQuery con n8n (seudonimizado)
Tablas del dataset `besthome_crm` (n8n inserta/actualiza; el CRM es la fuente):
- `leads`: `ref_code`, `cliente_hash`, fecha, canal de entrada (formulario/whatsapp/manual/referido), utm y click ids first/last (`gclid`, `gbraid`, `wbraid`, `fbclid` son identificadores de clic de anuncio, no de contacto), `ga_client_id`, `eb_id` de la propiedad de origen.
- `busquedas`: operación, tipo de inmueble, zonas, presupuesto, condición, características deseadas, etapa actual, motivo de caída (catálogo).
- `etapas_log`: **una fila por cambio de etapa** (`busqueda_id`, etapa anterior/nueva, fecha). Se alimenta con el webhook de cambio de etapa del CRM; sin esto se pierde la historia.
- `mostradas`: propiedades mostradas por búsqueda (`eb_id`, fecha, medio, reacción, motivo de descarte, zona y precio al momento).
- `cierres`: ver campos en el CRM. Solo valores de catálogo y números.

Reglas del flujo:
- **Lista blanca**: n8n arma cada fila campo por campo; nunca copia `payload_json` ni el registro completo.
- `cliente_hash` = HMAC-SHA256 del ID interno del contacto con un secreto que **solo vive en n8n** (credencial); permite contar clientes únicos y recompras sin poder reconstruir la identidad.
- Texto libre (notas, motivo de caída «otro», conversaciones) **no sale del CRM**.
- Fecha exacta de cierre, zona, fraccionamiento, precio y comisión sí; calle, número, nombre del dueño o constructor no.
- Con pocos cierres, zona + fraccionamiento + precio + fecha puede reidentificar una operación: las vistas de Looker deben agrupar (por mes, rango de precio) y el dataset se comparte solo con cuentas propias.
- Se carga por **upsert por llave** (`ref_code`, `busqueda_id`, `cierre_id`) para poder corregir y para borrar a petición ARCO (`DELETE … WHERE cliente_hash = …`).
- Los leads de WhatsApp usan el `GET /lead/{ref}` que n8n ya consulta (sin datos personales). Los clics sin mensaje (`local`, se borran a los 30 días) ya están en GA4 como `lead_generado`, no hace falta copiarlos.

## 3. Unir origen con cierre
`ref_code` viaja: clic/formulario → `lead_generado.lead_id` en GA4 → chat de WhatsApp (`Ref:`) → `Lead.ref_code` en el CRM → Búsqueda → Cierre. En BigQuery: `GA4 (lead_id) ⇄ leads (ref_code) ⇄ busquedas ⇄ cierres`, y `leads.gclid` ⇄ tabla de clics de Google Ads para campaña/palabra clave. La atribución first/last del propio payload del lead es la fuente principal; GA4 completa sesiones que no generaron lead.
Límites: contactos sin `Ref:` (llamada, recomendación) entran como origen `manual`/`referido`; un cliente con varios `ref_code` se une por `cliente_hash` y el origen es el del primero; consentimiento de cookies y bloqueadores dejan huecos.

## 4. Looker Studio y costos
- Looker Studio es gratis. Conectar a **vistas** de BigQuery (no a tablas crudas), con filtro de fecha obligatorio; el caché de Looker evita consultas repetidas.
- BigQuery gratis cada mes: 10 GB de almacenamiento y 1 TiB de consultas; la exportación diaria de GA4 y de Search Console no cobra carga. Un sitio de este tamaño queda muy por debajo. Verificar cifras vigentes en la página de precios de Google Cloud.
- Crear el proyecto **con facturación** (el sandbox borra tablas a los 60 días), poner **alerta de presupuesto** (p. ej. USD 5) y **cuota diaria de bytes consultados** en el proyecto. El riesgo de costo es una consulta mal hecha, no el almacenamiento.
- Exportación diaria de GA4: límite de 1 millón de eventos por día en propiedades estándar.

## 5. Privacidad
- Aviso de privacidad (pendiente de redactar, ver [../PENDIENTES.md](../PENDIENTES.md)): añadir finalidad de **análisis estadístico y de mercado con información seudonimizada**, proveedores (Google Analytics, Google Cloud/BigQuery, Google Ads, Meta), cookies de atribución y plazos. Subir `BH_PRIVACY_VERSION`.
- Seudonimizado no es anónimo: mientras el CRM guarde la correspondencia, siguen siendo datos personales para efectos legales; por eso hay borrado por `cliente_hash` y acceso restringido. Conviene revisión del texto con asesor legal.
- No enviar a GA4 nombre, teléfono ni correo (prohibido por sus términos); `ref_code` es un código aleatorio y es válido.
- Cuentas con acceso al proyecto de Google Cloud: solo las de la empresa, con verificación en dos pasos.

## Orden de pasos
**Ahora (sin depender del CRM ni de n8n):**
1. Proyecto de Google Cloud con facturación, alerta de presupuesto y cuota diaria; elegir región.
2. Activar exportación diaria GA4 → BigQuery.
3. Activar exportación masiva de Search Console.
4. Crear en el CRM los campos y entidades de [../crm/campos-analitica.md](../crm/campos-analitica.md) antes de capturar clientes reales.
5. Verificar en GA4 DebugView que `lead_generado` trae `lead_id`.

**Después:**
6. Conector Google Ads (al lanzar campañas).
7. Flujo n8n CRM → BigQuery (`etapas_log` primero, luego el resto) y foto diaria del catálogo.
8. Vistas y tablero Looker: embudo canal → visita → lead → cita → cierre por campaña y zona; precios y comisión por zona/tipo/condición; qué características se venden; motivos de caída; consultas SEO por página.
9. Meta Ads a BigQuery y conversiones offline ([conversiones-offline.md](conversiones-offline.md)), que reutilizan los mismos campos.
10. Captura automática de «propiedades mostradas» (hoy manual).
