# Privacidad: baja de marketing, cancelación (ARCO) y bloqueo legal

Estado: **diseño (2026-10-24)**. Sujeto a revisión de un asesor legal (la ley de datos se reformó en 2025 y hay obligaciones antilavado si Best Home es actividad vulnerable). Contexto de datos: [../n8n/bigquery-looker.md](../n8n/bigquery-looker.md).

## Dos tipos de solicitud
1. **Oposición / baja de marketing**: «ya no me manden publicidad». No se borra nada: se marca y todo envío (correo, WhatsApp, anuncios con listas) debe respetar la marca.
2. **Cancelación**: «borren mis datos». Depende de si hay operación cerrada.

## Baja de marketing (se implementa antes de enviar cualquier correo automático)
- Campos en Contacto: `no_contactar_marketing` (sí/no), `fecha_baja`, `canal_baja`.
- **Todo flujo de n8n que envíe correo o mensajes de campaña filtra esta marca** (regla obligatoria, sin excepciones). Correos de campaña con enlace de baja que activa la marca por n8n.
- **Lista de exclusión** mínima que sobrevive a cualquier borrado: solo hash (HMAC) del correo y del teléfono, para no volver a contactar a quien pidió que no se le contacte. Sin nombre ni otros datos.
- Audiencias de anuncios (Google/Meta con correo o teléfono con hash): excluir a esos contactos al subir listas.

## Cancelación
**A. Sin operación cerrada** (lead o búsqueda que no cerró): el administrador confirma y se borra el contacto del CRM (y sus chats en Evolution, si existen); se agrega a la lista de exclusión; BigQuery se anonimiza (abajo). Confirmar con el asesor legal si hay algún caso en que deba conservarse.
**B. Con operación cerrada**: no se borra; se **bloquea** (conservación por obligación legal, plazo de 5 años a verificar con el asesor legal):
- Casilla `bloqueo_legal` en Contacto y Cierre, y fecha `fin_bloqueo` = fecha de cierre + 5 años.
- El registro pasa a un equipo de EspoCRM que solo ve el administrador (ACL por equipos), se desasigna del asesor y desaparece de sus listas y búsquedas. Se activa `no_contactar_marketing`.
- Se conserva solo lo necesario para la obligación (identificación, operación, documentos); se borran notas, conversaciones y preferencias que no lo sean.
- Permisos de EspoCRM: los roles de asesor **no pueden eliminar** Contacto, Búsqueda ni Cierre; solo el administrador.
- Al llegar `fin_bloqueo`, n8n avisa al administrador para revisar y borrar definitivamente (no es automático).
- Uso del dato bloqueado: solo atender a la autoridad; nunca marketing ni prospección.

## BigQuery: anonimización al ejercer cancelación u oposición
Un flujo de n8n, disparado por el administrador desde el CRM (casilla en el Contacto), calcula el `cliente_hash` y ejecuta un `UPDATE` parametrizado en todas las tablas de `besthome_crm`:

| Se conserva (para reportes de canal, costo e ingreso) | Se quita o se generaliza |
|---|---|
| `canal_entrada`, `utm_*`, `campaign_id`/grupo de anuncios (guardados desde el origen), fuente y medio | `cliente_hash` → `BORRADO_ARCO`; `ref_code`, `busqueda_id`, `cierre_id` → identificador aleatorio nuevo |
| Etapas y su orden, `motivo_caida`, reacciones de visitas, asesor | `ga_client_id`, `gclid`, `gbraid`, `wbraid`, `fbclid`, `_fbc`, `_fbp` → vacíos |
| Tipo de operación y de inmueble, condición, características, zona | `id_propiedad`/`eb_id` → vacío; fraccionamiento → solo zona |
| Precio de cierre, comisión, días de ciclo | Fechas exactas → solo el mes |
| | Cualquier texto libre (nunca llega a BigQuery) |

Así el reporte «qué canal trajo qué venta y a qué costo» sigue funcionando; solo desaparece el vínculo con una persona, una propiedad concreta y un instante exacto. Se marca `anonimizado = verdadero` y se registra la solicitud en `arco_log` (fecha, tipo, resultado, **sin datos personales**).
Además:
- WordPress: si todavía existe el lead en `wp_bh_leads`, borrar la fila desde Propiedades › Leads (hoy ya existe «Eliminar»).
- GA4: los datos crudos solo llevan `ref_code` aleatorio y `user_pseudo_id`; sin el CRM no identifican a nadie. Si se quisiera borrarlo también, existe la API de eliminación de usuarios de GA4 (opcional).
- Evolution API: borrar los chats de ese número en su Postgres en los casos tipo A.

## Reglas por defecto de BigQuery (sin solicitud)
Precio y comisión **exactos** (los reportes financieros no pueden tener margen de error) y fechas exactas. La barrera es: sin nombre, contacto, calle, número ni clave catastral; `cliente_hash` con secreto solo en n8n; y acceso al proyecto de Google Cloud restringido a cuentas propias con verificación en dos pasos.
