# Lista final de campos y permisos de EspoCRM (para crear)

Estado: **propuesta cerrada para crear (2026-10-25)**. Racional y reportes: [campos-analitica.md](campos-analitica.md), [recorridos.md](recorridos.md), [privacidad-arco.md](privacidad-arco.md), [../n8n/bigquery-looker.md](../n8n/bigquery-looker.md). Reglas: catálogos (enum) en vez de texto; solo el ID de la casa es obligatorio; lo que no sea del asesor lo llena n8n.

Leyenda de acceso: **A** = asesor puede ver/editar; **Adm** = solo administrador (permisos por campo en los roles de EspoCRM); **n8n** = lo escribe n8n.

## Lead (una fila por `ref_code`; un Contacto puede tener varios Leads)
| Campo | Tipo | Acceso |
|---|---|---|
| `ref_code` | texto único (R-XXXXXX) | A (lectura) |
| `canal_entrada` | formulario \| whatsapp \| manual \| referido \| llamada | A (lectura) |
| `eb_id_origen` | texto (propiedad desde la que escribió) | A (lectura) |
| `fecha_entrada` | fecha-hora | n8n |
| `fecha_primera_atencion` | fecha-hora (primer mensaje o llamada del asesor) | n8n / A (lectura) |
| `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term` (first y last) | texto | **Adm** |
| `campaign_id` (Ads/Meta, lo resuelve n8n) | texto | **Adm** |
| `gclid`, `gbraid`, `wbraid`, `fbclid`, `_fbc`, `_fbp`, `ga_client_id` | texto | **Adm** |
| `landing_page_first`, `referrer_first` | texto | **Adm** |

## Contacto
Identidad y contacto (los demás campos estándar de EspoCRM) más:
| Campo | Tipo | Acceso |
|---|---|---|
| `no_contactar_marketing` | casilla | **Adm** (el asesor lo ve en solo lectura si se decide) |
| `fecha_baja_marketing`, `canal_baja` (correo \| whatsapp \| manual) | fecha-hora / enum | **Adm** |
| `bloqueo_legal`, `fin_bloqueo` | casilla / fecha | **Adm** |
| `origen_contacto` | nuevo \| heredado (importado de EasyBroker u otro) | **Adm** |
`cliente_hash` no es campo: lo calcula n8n.

## Búsqueda
| Campo | Tipo | Acceso |
|---|---|---|
| `etapa` | Nuevo → Contactado → Recorrido agendado → Recorrido realizado → Negociación → Ganado \| Perdido | A |
| `asesor` | usuario asignado | A |
| `operacion` | venta \| renta | A |
| `tipos_inmueble` | multi: casa \| departamento \| terreno \| local \| otro | A |
| `zonas`, `fraccionamientos` | multi | A |
| `presupuesto_min`, `presupuesto_max` | moneda | A |
| `condicion` | nueva \| usada \| desarrollo | A |
| `recamaras_min` | número | A |
| `caracteristicas_deseadas` | multi (catálogo compartido) | A |
| `motivo_caida` | no contesta o datos falsos \| presupuesto insuficiente \| buscaba otra zona o producto \| compró o rentó con otro \| financiamiento no aprobado \| se enfrió \| otro (solo al pasar a Perdido) | A |
| `ref_code_origen` | texto (copia del Lead de origen, lo pone n8n) | n8n |
Cambios de `etapa` y de `asesor` generan evento a n8n (`etapas_log`).

## Recorrido (hijo de Búsqueda; opcional)
`fecha`, `asesor`, `ids_a_visitar` (texto, opcional, pegar varios IDs), `comentario_general` (texto, opcional). **Sin** campo de interés ni de resultado: es redundante con la reacción de cada visita.

## Visita (hija de Recorrido; una por casa visitada)
| Campo | Tipo | Quién |
|---|---|---|
| `id_propiedad` | texto (EB-… / BH-… / ID externo) | A (**único obligatorio**) |
| `hora_visita` | hora | A |
| `vista` | sí \| no (por defecto sí) | A |
| `motivo_no_vista` | cerrada \| dueño no disponible \| cliente canceló \| otro | A |
| `reaccion` | le gustó \| neutral \| no le gustó | A |
| `comentario` | texto libre (no sale a BigQuery) | A |
| `origen` | propia \| otra inmobiliaria | n8n |
| `zona`, `fraccionamiento`, `tipo`, `condicion`, `precio`, `m2_construccion`, `m2_terreno`, `url_ficha` | copia al momento | n8n |

## Cierre (hijo de Búsqueda)
| Campo | Tipo | Notas |
|---|---|---|
| `tipo_operacion` | venta \| renta | |
| `id_propiedad` | texto | |
| `fecha_cierre` | **fecha exacta** | sin redondeo |
| `dias_ciclo` | entero | n8n o fórmula: de `fecha_entrada` a `fecha_cierre` |
| `precio_lista`, `precio_cierre` | moneda | exactos |
| `comision_cobrada` | moneda, sin IVA | total cobrado por la operación |
| `ingreso_inmobiliaria` | moneda, sin IVA | lo que **le queda a Best Home** (comisión cobrada menos lo compartido con otra inmobiliaria, si aplica). *Por confirmar la definición con Javier; el pago al asesor es un costo interno y no se resta aquí.* |
| `lado` | vendedor \| comprador \| ambos | |
| `tipo_vendedor` | particular \| desarrollador \| inmobiliaria o asesor \| otro | |
| `forma_pago` | contado \| crédito bancario \| Infonavit/Fovissste \| mixto | |
| `asesor` | usuario | |
| `zona`, `fraccionamiento`, `tipo_inmueble`, `condicion`, `recamaras`, `banos`, `m2_construccion`, `m2_terreno`, `caracteristicas` | copia al momento (n8n desde WordPress) | |
| `bloqueo_legal`, `fin_bloqueo` | casilla / fecha | **Adm** |

## Permisos de borrado
Los roles de asesor **no pueden eliminar** Contacto, Lead, Búsqueda, Recorrido, Visita ni Cierre. Solo el administrador.

## Catálogo compartido de características (multi)
roof garden, recámara en planta baja, jardín, terraza, alberca, cuarto de servicio, bodega, estudio, cochera techada (1/2/3+), seguridad 24 h, casa club, mascotas, vista, esquina. Mismos slugs que los términos `amenidad` del sitio.
