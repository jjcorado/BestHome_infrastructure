# Análisis de datos: BigQuery y Looker Studio

Estado: **diseño listo (2026-10-24); ejecución por fases**. Campos del CRM y del sitio que lo hacen posible: [../crm/campos-analitica.md](../crm/campos-analitica.md).

## Principio
BigQuery es una **base de análisis sin identidad**: nunca recibe nombre, teléfono, correo, mensajes, notas libres, calle/número ni IP. El CRM conserva la identidad y la llave para volver a ella. Todo se une por `ref_code` y por un `cliente_hash` interno.

## Principios de la capa de datos
- **Independiente de la herramienta**: BigQuery y Looker son el destino actual, no el modelo. El esquema de las tablas propias (`besthome_crm`) se documenta y versiona aquí, con nombres `snake_case` y tipos estándar, para leerlo igual desde Python, Power BI o Metabase. La lógica de negocio va en vistas SQL estándar, no en campos calculados de Looker.
- **Inventario desde WordPress**: toda foto o consulta de propiedades sale de la base de WordPress (API de catálogo), nunca de EasyBroker, que es temporal.
- **Un solo integrador**: n8n, por API y con lista blanca. Sin lecturas directas a MySQL del CRM.
- **Modular**: una fuente o tabla por flujo de n8n, documentada en su propio archivo; agregar una fuente no cambia las demás.

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
- `etapas_log`: **una fila por cambio de etapa**, por evento (webhook de EspoCRM al cambiar `etapa`; n8n inserta la fila con su fecha). Una API REST solo devuelve el estado actual, así que esta historia no se puede reconstruir después: activarla desde el primer día. Enable también el registro de auditoría de EspoCRM en `etapa` (`busqueda_id`, etapa anterior/nueva, fecha). Se alimenta con el webhook de cambio de etapa del CRM; sin esto se pierde la historia.
- `recorridos` y `visitas`: una fila por visita (`id_propiedad`, origen propia/otra, vista sí/no, reacción le gustó/neutral/no le gustó, zona, tipo, condición, precio y m² al momento; sin comentarios libres) y una fila por recorrido con su `resultado`. Ver [../crm/recorridos.md](../crm/recorridos.md).
- `cierres`: ver campos en el CRM. Solo valores de catálogo y números.

Reglas del flujo:
- **Eficiencia de n8n**: `EXECUTIONS_DATA_SAVE_ON_SUCCESS=none` (no guardar historial de ejecuciones correctas), cargas diarias paginadas por lotes (p. ej. 100) con marca `modified_at` y sin traer tablas completas. Con decenas de cierres el volumen es chico; esto evita sorpresas al crecer.
- **Sin capa intermedia extra**: el CRM ya tiene su respaldo diario (base de datos + copia fuera del VPS), y BigQuery se reconstruye desde el CRM. Única tabla que no se reconstruye es `etapas_log` (y la foto del catálogo): protegerlas con exportación mensual a un bucket de Cloud Storage (CSV o Parquet), que también sirve como salida si algún día se migra de herramienta.
- **Lista blanca**: n8n arma cada fila campo por campo; nunca copia `payload_json` ni el registro completo.
- `cliente_hash` = HMAC-SHA256 del ID interno del contacto con un secreto que **solo vive en n8n** (credencial); permite contar clientes únicos y recompras sin poder reconstruir la identidad.
- Texto libre (notas, motivo de caída «otro», conversaciones) **no sale del CRM**.
- Fecha exacta de cierre, zona, fraccionamiento, precio y comisión sí; calle, número, nombre del dueño o constructor no.
- Con pocos cierres, zona + fraccionamiento + precio + fecha puede reidentificar una operación: las vistas de Looker deben agrupar (por mes, rango de precio) y el dataset se comparte solo con cuentas propias.
- Se carga por **upsert por llave** (`ref_code`, `busqueda_id`, `cierre_id`) para poder corregir y para borrar a petición ARCO (`DELETE … WHERE cliente_hash = …`).
- Los leads de WhatsApp usan el `GET /lead/{ref}` que n8n ya consulta (sin datos personales). Los clics sin mensaje (`local`, se borran a los 30 días) ya están en GA4 como `lead_generado`, no hace falta copiarlos.

## 2b. Reportes objetivo
- **CAC y retorno por rango de precio**: gasto (Google Ads/Meta por `campaign_id`) ÷ cierres, por rango de precio de cierre. Requiere que `utm_campaign` use la misma convención que el nombre/ID de campaña en Ads y que el auto-etiquetado de Google Ads esté activo (`gclid`).
- **Embudo y días a cierre** por asesor, tipo de propiedad y fuente (`etapas_log`).
- **Demanda vs. oferta**: búsquedas (CRM) y vistas de fichas (GA4 `view_item`) por zona/rango de precio contra el catálogo (foto diaria). Para filtros usados en el catálogo falta un evento (`search` está pendiente en [../sitio-web/GTM.md](../sitio-web/GTM.md)).
- **Propiedades estancadas**: vistas y leads por `eb_id` contra días publicada → candidatas a ajuste de precio.
- **Recorridos**: motivos de descarte, casas más mostradas vs. vendidas.

## 2c. Qué guardar hoy que luego no se puede reconstruir
Estado de etapas con fecha (`etapas_log`), foto diaria del catálogo (precio, estado, vistas), gasto diario por campaña (Ads/Meta), atribución first/last del lead (ya existe), fecha de la primera respuesta del cliente, asesor asignado en cada cambio, y fecha de publicación y de archivado de cada propiedad. Los reportes finales se definen después; estos datos se juntan desde ahora.
Reportes de negocio a definir: ingresos por cliente y por asesor en el tiempo (comisión de la inmobiliaria por cierre; clientes recurrentes por `cliente_hash`), cierres por asesor y por periodo, y lead → cierre por fuente incluyendo los perdidos y sin respuesta.

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
- Criterio práctico: no se cruzan juntos nombre/teléfono del cliente con la dirección exacta de la casa; BigQuery lleva `cliente_hash` + zona/fraccionamiento/precio. Es el esquema menos riesgoso, pero sigue siendo seudonimizado (no anónimo) y conviene revisión legal del aviso.
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
10. Hoja de recorrido móvil en el sitio ([../crm/recorridos.md](../crm/recorridos.md)); mientras tanto, las visitas se capturan en el CRM.
