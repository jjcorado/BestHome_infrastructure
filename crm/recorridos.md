# Recorridos (hoja de recorrido y registro de propiedades mostradas)

Estado: **diseño (2026-10-24)**. Sustituye a «Propiedad mostrada» de [campos-analitica.md](campos-analitica.md). Datos hacia BigQuery: [../n8n/bigquery-looker.md](../n8n/bigquery-looker.md).

## Principio de uso
Móvil primero y **opcional**: lo único obligatorio es pegar los IDs de las casas. Todo lo demás tiene valor por defecto o es un toque. Meta: capturar un recorrido completo en menos de 2 minutos, y que la hoja sea más útil que la tabla manual de antes (precio y metros a la mano), para que el asesor la use por conveniencia y no por obligación.

## Modelo (en el CRM, dos entidades)
- **Recorrido** (hijo de Búsqueda, opcional): `fecha`, `asesor`, `comentario_general`, `estado` (programado | realizado | cancelado).
- **Parada** (hija de Recorrido): `hora_visita`, `orden`, `id_propiedad` (texto), `origen` (propia | otra inmobiliaria), `vista` (sí | no), `motivo_no_vista` (cerrada | dueño no disponible | cliente canceló | otro), `reaccion` (le gustó | tal vez | no le gustó), `motivo_descarte` (precio | zona | tamaño | estado | distribución | otro), `comentario`, y **copia al momento**: ubicación (zona/fraccionamiento), tipo, condición, precio, m² construcción, m² terreno, `url_ficha`.
- Las propiedades **no** se guardan como entidad en el CRM (decisión vigente). `id_propiedad` es solo texto; la copia al momento hace que el reporte sobreviva aunque la casa cambie de precio o se archive.

## De dónde salen los datos de la casa
**Fuente única: la base de datos de WordPress** (CPT `propiedad`), que será el inventario principal cuando se deje EasyBroker. La hoja en wp-admin lee con `bh_prop()`; n8n lee con la API de catálogo (`GET /propiedades`, `X-BH-Key`).

Orden de búsqueda al pegar un ID o un link:
1. **Propia / ya sincronizada**: se busca `bh_eb_id` (`EB-` / `BH-`) en WordPress. Trae precio, ubicación, m², link a la ficha completa. Hoy el sitio ya tiene `bh_price`, `bh_construction`, `bh_lot`, `bh_location_name` (CONTRATO-DATOS).
2. **De otra inmobiliaria**: si el ID no existe en el sitio, se marca `origen = otra inmobiliaria`, se guarda el ID externo y se pide una captura mínima (ubicación, precio, m², link). El ID externo (a menudo de EasyBroker) es solo texto de referencia: **no se consulta la API de EasyBroker**, que se dejará de usar.
3. Un link del sitio o de EasyBroker se reduce al ID automáticamente.

## Interfaz (recomendación)
**Hoja de recorrido** como página móvil dentro de wp-admin (Propiedades › Recorridos), con capacidad propia tipo `bh_view_leads` (p. ej. `bh_view_recorridos`) y nonce, igual que la vista de Leads. Razón: toda la información de propiedades ya vive en WordPress y la regla del proyecto prohíbe exponer `bh_priv_*` en frontend; una página con sesión de asesor lo cumple. No usar enlaces con token público si muestran metas privadas.
Flujo:
1. **Antes**: desde la Búsqueda (botón/enlace con `busqueda_id`) o sin cliente. Pegar IDs/links, una por línea; fijar hora de cada parada (opcional). La hoja muestra una tarjeta por casa: hora, precio, ubicación, m² de construcción y terreno, link a la ficha completa, nota interna visible solo al asesor.
2. **Durante**: toque en una parada para marcar *vista / no vista*; nada más es obligatorio.
3. **Después**: por casa, 3 botones de reacción; si «no le gustó», chips con el motivo (un toque). Comentario libre opcional (sirve el dictado del teclado). Un comentario general del recorrido.
4. **Guardar**: envía a n8n, que crea Recorrido y Paradas en el CRM con la Búsqueda. Sin cliente, queda «sin asignar» y se vincula después.
Otra opción más barata para una primera versión: en el CRM, un campo de texto «IDs del recorrido» en la Búsqueda, y n8n crea las Paradas con los datos de la casa; el asesor las completa luego en el CRM. Menos código, peor experiencia móvil. **Decisión pendiente de Javier.**
Descartado: Google Sheets (rápido, pero no se vincula al cliente ni a las propiedades).

## Reportes que habilita
- Por casa: fechas en que se mostró, a cuántos, reacciones y comentarios (sale del CRM; los comentarios libres no van a BigQuery).
- Por cliente: qué se le enseñó, qué le gustó y por qué descartó (patrón de motivos → ajusta la búsqueda).
- Global (BigQuery): motivos de descarte por zona/precio, casas más mostradas vs. vendidas, propias vs. de otras inmobiliarias, visitas realizadas por cierre.

## Qué NO hacer (para que se use)
- Sin campos obligatorios salvo el ID; sin captura de datos de la casa a mano si el ID existe.
- Sin pantallas por casa con más de 3 toques.
- No pedir que el asesor llene el CRM en el recorrido: la hoja es la herramienta; el CRM recibe los datos.
