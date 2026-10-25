# Campos del CRM y del sitio para analítica

Estado: **diseño (2026-10-24)**. La lista definitiva para crear los campos está en [campos-espocrm.md](campos-espocrm.md); este archivo conserva el racional. Entidades del CRM: Lead, Contacto, Búsqueda (antes Oportunidad), Cierre. Destino de los datos y reglas de privacidad: [../n8n/bigquery-looker.md](../n8n/bigquery-looker.md). Regla general: **catálogos (enum / multi-enum) en vez de texto libre**, con los mismos valores en CRM, sitio y BigQuery; sin catálogo no hay gráficas.

## Leads perdidos y sin respuesta (también se analizan)
Todo lead queda en el CRM y en BigQuery aunque no avance. Campos para eso: `fecha_primera_respuesta` (del cliente), `etapa` con valores `sin_respuesta` y `descartado`, y el origen completo (canal, campaña, `referrer` de first-touch). Permite ver, por ejemplo, qué fuente (anuncios, buscadores, asistentes de IA como ChatGPT) trae contactos que no contestan o que no encontraban lo que buscaban. La tasa de respuesta y el tiempo a la primera respuesta se calculan en BigQuery.

## Atención y asignación
`fecha_primera_atencion` (automática, ver bigquery-looker 2d) y registro de cada cambio de `asesor` asignado (webhook a n8n). El asesor no captura nada extra.

## Lead (hacer ya)
`ref_code` (único), `canal_entrada` (formulario | whatsapp | manual | referido | llamada), UTM first y last (source, medium, campaign, term, content), `gclid`/`gbraid`/`wbraid`, `fbclid`, `_fbc`/`_fbp`, `ga_client_id`, `landing_page_first`, `eb_id_origen` (propiedad desde la que escribió), `fecha_primer_contacto`.

## Contacto
Solo identidad y contacto. Para analítica: `cliente_hash` lo calcula n8n (no se captura a mano). `es_recompra` / número de operaciones.

## Búsqueda (lo que el cliente quiere)
`operacion` (venta | renta), `tipos_inmueble` (casa | departamento | terreno | local | otro), `zonas` (multi), `fraccionamientos` (multi), `presupuesto_min/max`, `condicion` (nueva | usada | desarrollo), `recamaras_min`, `caracteristicas_deseadas` (multi; catálogo abajo), `asesor` (usuario asignado), `etapa`, **`motivo_caida`** (precio | zona | compró con otro | desistió | financiamiento | sin respuesta | otro) + nota interna (no sale a BigQuery), `fecha_cita`, `fecha_visita` (además del log de cambios de etapa en n8n).

## Recorrido y Visita (propiedades mostradas)
Sustituye a «Propiedad mostrada». Entidades, campos, origen de los datos de la casa y hoja móvil: [recorridos.md](recorridos.md). Mantener mínimo lo obligatorio: el asesor solo pega IDs.

## Cierre
`tipo_operacion` (venta | renta), `eb_id`, `precio_lista`, `precio_cierre`, `comision_monto`, `comision_pct`, `lado` (vendedor | comprador | ambos), `tipo_vendedor` (particular | desarrollador | inmobiliaria/asesor | otro), `fecha_cierre`, `asesor`, `dias_hasta_cierre` (calculado), `forma_pago` (contado | crédito bancario | Infonavit/Fovissste | mixto), y **copia de la propiedad al cierre**: `zona`, `fraccionamiento`, `tipo_inmueble`, `condicion`, `recamaras`, `banos`, `m2_construccion`, `m2_terreno`, `caracteristicas` (multi). Relación Cierre → Búsqueda → Lead. Si la operación se cae: Búsqueda con `motivo_caida`, sin Cierre.

## Catálogo de características (multi-enum compartido)
Empezar con: roof garden, recámara en planta baja, jardín, terraza, alberca, cuarto de servicio, bodega, estudio, cochera techada (1/2/3+), seguridad 24 h, casa club, mascotas, vista, esquina. Ampliable; cada valor es un slug estable (`roof_garden`, `recamara_pb`…).

## Sitio (verificar antes de empezar)
- Cada característica del catálogo debe existir como término de `amenidad` (y ser consultable en `GET /propiedades`) para copiarla al Cierre y a «Propiedad mostrada» sin teclearla. Falta confirmar en el código qué términos existen, y si hay campo/taxonomía de **fraccionamiento** y de **condición** (nueva | usada | desarrollo); si no existen, proponer agregarlos al CPT (`includes/cpt.php`) y a `bh_prop()`/contrato de datos.
- Comisión y precio de cierre **no** van al sitio ([../DECISIONES.md](../DECISIONES.md)); `bh_priv_commission` es la de captación y tampoco sale a BigQuery.
- `lead_generado` ya envía `lead_id` = `ref_code`; no requiere cambios para la unión.
