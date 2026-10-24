# Recorridos y visitas (registro de propiedades mostradas)

Estado: **diseño (2026-10-24), revisado con las observaciones de Javier y de Gemini**. Sustituye a «Propiedad mostrada» de [campos-analitica.md](campos-analitica.md). Datos hacia BigQuery: [../n8n/bigquery-looker.md](../n8n/bigquery-looker.md).

## Principio de uso
Todo ocurre **dentro del CRM** (EspoCRM, móvil): el asesor no sale a otra pantalla. Opcional y mínimo: lo único que se escribe a mano es el ID de la casa; precio, ubicación y metros los llena n8n. Una pantalla por visita con 3 toques como máximo.

## Modelo
- **Recorrido** (hijo de Búsqueda; opcional): `fecha`, `asesor`, `ids_a_visitar` (texto, un ID o link por línea; opcional), `resultado` (quiere ofertar o apartar | lo piensa o consulta con su familia | quiere ver más opciones | sin interés), `comentario_general`.
- **Visita** (hija de Recorrido; una por casa, terreno o departamento visitado): `hora_visita`, `id_propiedad` (texto), `origen` (propia | otra inmobiliaria), `vista` (sí | no), `motivo_no_vista` (cerrada | dueño no disponible | cliente canceló | otro), `reaccion` (**le gustó | neutral | no le gustó**), `comentario` (abierto; **sin catálogo de motivos**), y los campos que llena n8n: ubicación (zona y fraccionamiento), tipo, condición, precio, m² de construcción, m² de terreno, `url_ficha`, todos como copia al momento.
- «Neutral» = comentó pero no descarta. No hay motivo de descarte estructurado por decisión de Javier; el porqué vive en el comentario, que no sale a BigQuery. Si más adelante hace falta medir motivos, se agrega un campo opcional sin romper nada.
- Las propiedades no son entidad del CRM: `id_propiedad` es texto y la copia al momento hace que el reporte sobreviva si la casa cambia de precio o se archiva.

## Cómo se captura (desde el celular, en el CRM)
1. Desde la Búsqueda, crear el Recorrido y, si se quiere, pegar todos los IDs en `ids_a_visitar`: n8n crea las Visitas en orden, ya con precio, ubicación y metros. (Alternativa: «agregar visita» una por una con solo el ID.)
2. Durante o después: abrir cada Visita y marcar vista sí/no y reacción; comentario opcional (sirve el dictado del teclado).
3. Al final: `resultado` y comentario general del recorrido.
Si el asesor no usa nada de esto, no pasa nada: el CRM y los reportes funcionan, solo falta ese detalle.

## Quién llena los datos de la casa (n8n)
EspoCRM dispara un webhook al guardar la Visita (o el Recorrido con IDs). n8n:
1. Busca el ID en el inventario de WordPress: `GET /wp-json/besthome/v1/propiedades?q=<ID>&visibilidad=todas` (llave `X-BH-Key`). Hay que **confirmar que `q` encuentre `bh_eb_id`** y, si no, agregar el filtro `eb_id`.
2. Si existe, escribe en la Visita precio, ubicación, tipo, condición, m², link a la ficha, `origen = propia`. La API no devuelve `bh_priv_*` (incluida la dirección exacta «ubicación o modelo»).
3. Si no existe, deja `origen = otra inmobiliaria`; el asesor puede escribir esos datos en el comentario o en campos opcionales (precio, m²). El ID externo es solo referencia: no se consulta la API de EasyBroker.

**Fuente única del inventario: la base de datos de WordPress** (CPT `propiedad`), que seguirá siendo el inventario cuando se deje EasyBroker.

## IDs y propiedades que no se borran
- Los `EB-XXXXXX` actuales son permanentes y **no cambian** al dejar EasyBroker; las propiedades nuevas usarán `BH-XX0000`. Por eso los IDs guardados en visitas, cierres y GA4 siguen válidos y las URLs no cambian por esto.
- Una propiedad **nunca se borra, solo cambia de estado** (`publica | privada | archivada`). Así cualquier ID histórico se resuelve siempre. Pendiente: confirmar que el plugin impide el borrado definitivo (papelera y «eliminar permanentemente») a los roles que no sean administrador, y documentarlo en [../sitio-web/CONTRATO-DATOS.md](../sitio-web/CONTRATO-DATOS.md).

## Reportes que habilita
- Por casa: cuándo se mostró, a quién, cuántos la vieron, reacción y comentarios (CRM).
- Por cliente: qué se le enseñó, qué le gustó y cuál fue el resultado del recorrido.
- Global (BigQuery): reacción por zona, precio y condición; casas más mostradas vs. vendidas; visitas por cierre; resultado de recorridos por asesor.

## Descartado
- Hoja móvil en wp-admin: obliga a cambiar de sistema en la calle.
- Google Sheets: difícil de enlazar a la Búsqueda y de extraer de forma confiable.
