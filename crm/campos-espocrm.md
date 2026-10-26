# Campos y permisos de EspoCRM para analítica (sobre el diseño v3)

Estado: **propuesta para crear (revisada 2026-10-25)**. Base: [DISENO-ESPOCRM.md](DISENO-ESPOCRM.md). **No hay entidad Lead**: Contacto + Búsqueda + Cierre. Racional y reportes: [../n8n/bigquery-looker.md](../n8n/bigquery-looker.md), [recorridos.md](recorridos.md), [privacidad-arco.md](privacidad-arco.md).

Reglas: catálogos en vez de texto; solo el ID de la casa es obligatorio en lo nuevo; lo que no sea del asesor lo llena n8n. **A** = asesor ve/edita; **A-lectura**; **Adm** = solo administradores (permisos por campo de los roles); **n8n** = lo escribe n8n.

## Contacto (agregar al diseño v3)
| Campo | Tipo | Acceso |
|---|---|---|
| `suscrito_boletin` | casilla (doble confirmación o aceptación al ofrecerlo) | A / n8n |
| `no_contactar_marketing` | casilla | n8n y Adm; **asesores: solo lectura** |
| `fecha_baja_marketing`, `canal_baja` (correo \| whatsapp \| manual) | fecha-hora / enum | Adm |
| `bloqueo_legal`, `fin_bloqueo` | casilla / fecha | Adm |
| `origen_contacto` | nuevo \| heredado (EasyBroker u otro CRM) | Adm |
`cliente_hash` no es campo: lo calcula n8n.

## Búsqueda (agregar al diseño v3; las etapas siguen siendo las de v3)
| Campo | Tipo | Acceso |
|---|---|---|
| `ref_code` (el del primer contacto), `ref_codes_adicionales` (texto, uno por línea) | texto | A-lectura / n8n |
| `canal_entrada` | formulario \| whatsapp \| manual \| referido \| llamada | A-lectura |
| `eb_id_origen` | texto (casa desde la que escribió) | A-lectura |
| `fecha_entrada` | fecha-hora (la del lead original) | n8n |
| `fecha_primera_respuesta` | fecha-hora; **fórmula de EspoCRM**: se llena al salir la etapa de «Nuevo» por primera vez | automática |
| `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term` (first y last), `campaign_id` | texto | Adm |
| `gclid`, `gbraid`, `wbraid`, `fbclid`, `_fbc`, `_fbp`, `ga_client_id`, `landing_page_first`, `referrer_first` | texto | Adm |
| `tipos_inmueble` | multi: casa \| departamento \| terreno \| local \| otro | A (opcional) |
| `motivo_perdida` | spam o no válido \| no contesta \| presupuesto insuficiente \| buscaba otra zona o producto \| compró o rentó con otro \| financiamiento no aprobado \| se enfrió \| otro (obligatorio solo al pasar a Perdido; «spam» se excluye de métricas) | A |
| `presupuesto_min` | moneda | A (opcional; v3 ya tiene el máximo) |
| `condicion_buscada`, `fraccionamientos`, `caracteristicas_deseadas` | enum / multi / multi | A (opcional; **después del piloto**, ver revisión) |
Ya existen en v3 y se reutilizan: asesor, origen, presupuesto máximo, forma de pago, horizonte, zonas, recámaras, nota, precalificación, broker, estatus. Cambios de **etapa** y de **asesor** generan evento a n8n (`etapas_log`).

## Recorrido (Reunión renombrada, v3) y Visita (entidad hija nueva)
Recorrido: fecha, asesor, `ids_a_visitar` (texto opcional para pegar varios), `comentario_general`. Se **retiran** «propiedades mostradas» y «retroalimentación» del Recorrido (pasan a las Visitas).
| Visita | Tipo | Quién |
|---|---|---|
| `id_propiedad` | texto (EB- / BH- / ID externo) | A (**único obligatorio**) |
| `hora_visita` | hora | A |
| `vista` / `motivo_no_vista` | sí \| no (defecto sí) / cerrada \| dueño no disponible \| cliente canceló \| otro | A |
| `calificacion` | no le gustó \| sin reacción \| le gustó \| le encantó | A |
| `comentario` | un solo texto libre, positivo o negativo (no sale a BigQuery) | A |
| `origen`, `zona`, `fraccionamiento`, `tipo`, `condicion`, `precio`, `m2_construccion`, `m2_terreno`, `url_ficha` | copia al momento | n8n |

## Cierre (la «Operación» de v3; agregar)
v3 ya trae: propiedad (código o texto), precio, tipo de vendedor, comisión % editable, apartado, enganche, fechas de promesa, escrituración y entrega, notaría, motivo de caída y campos de renta.
| Campo | Tipo | Notas |
|---|---|---|
| `fecha_cierre` | fecha exacta | **fecha de escrituración** (venta) o de firma (renta) |
| `escritura_recogida`, `fecha_escritura_recogida` | casilla / fecha | A (marca que el cliente ya recogió escritura, certificados, hoja del Registro Público y factura); detiene los recordatorios |
| `resena_solicitada` | fecha | n8n |
| `dias_ciclo` | entero | n8n: de `fecha_entrada` a `fecha_cierre` |
| `ingreso_inmobiliaria` | moneda, sin IVA | lo que ingresó a Best Home (ya es su parte si fue compartida) |
| `compartida_otra_inmobiliaria` | sí \| no | para ver dónde falta inventario propio |
| `precio_lista` | moneda | |
| `zona`, `fraccionamiento`, `tipo_inmueble`, `condicion`, `recamaras`, `banos`, `m2_construccion`, `m2_terreno`, `caracteristicas` | copia al momento (n8n desde WordPress; v3 ya prevé zona, tipo y precio de lista) | |
| `bloqueo_legal`, `fin_bloqueo` | casilla / fecha | Adm |
Se descartan de mi propuesta anterior: `lado`, `comision_cobrada` y `forma_pago` (v3 la tiene en la Búsqueda).

## Permisos
Los roles de asesor no pueden eliminar ni exportar (v3). Solo administradores borran Contacto, Búsqueda, Recorrido, Visita y Cierre.

## Catálogo compartido de características (multi)
roof garden, recámara en planta baja, jardín, terraza, alberca, cuarto de servicio, bodega, estudio, cochera techada (1/2/3+), seguridad 24 h, casa club, mascotas, vista, esquina. Mismos slugs que los términos `amenidad` del sitio.
