# Campos del CRM y del sitio para analítica

Estado: **diseño (2026-10-24)**. Entidades del CRM: Lead, Contacto, Búsqueda (antes Oportunidad), Cierre. Destino de los datos y reglas de privacidad: [../n8n/bigquery-looker.md](../n8n/bigquery-looker.md). Regla general: **catálogos (enum / multi-enum) en vez de texto libre**, con los mismos valores en CRM, sitio y BigQuery; sin catálogo no hay gráficas.

## Lead (hacer ya)
`ref_code` (único), `canal_entrada` (formulario | whatsapp | manual | referido | llamada), UTM first y last (source, medium, campaign, term, content), `gclid`/`gbraid`/`wbraid`, `fbclid`, `_fbc`/`_fbp`, `ga_client_id`, `landing_page_first`, `eb_id_origen` (propiedad desde la que escribió), `fecha_primer_contacto`.

## Contacto
Solo identidad y contacto. Para analítica: `cliente_hash` lo calcula n8n (no se captura a mano). `es_recompra` / número de operaciones.

## Búsqueda (lo que el cliente quiere)
`operacion` (venta | renta), `tipos_inmueble` (casa | departamento | terreno | local | otro), `zonas` (multi), `fraccionamientos` (multi), `presupuesto_min/max`, `condicion` (nueva | usada | desarrollo), `recamaras_min`, `caracteristicas_deseadas` (multi; catálogo abajo), `etapa`, **`motivo_caida`** (precio | zona | compró con otro | desistió | financiamiento | sin respuesta | otro) + nota interna (no sale a BigQuery), `fecha_cita`, `fecha_visita` (además del log de cambios de etapa en n8n).

## Propiedad mostrada (entidad nueva, hija de Búsqueda)
Una fila por propiedad enseñada o enviada: `eb_id` (EB-/BH-; texto, no relación al sitio), `fecha`, `medio` (enviada por WhatsApp | visita presencial | recorrido virtual | ficha web), `reaccion` (le interesó | descartada | pendiente), `motivo_descarte` (precio | zona | tamaño | estado | distribución | otro), y **copia al momento**: zona, fraccionamiento, tipo, condición y precio (la propiedad puede cambiar o archivarse). Captura manual al inicio (lista rápida en la Búsqueda); automatizable después con n8n (links del catálogo enviados por WhatsApp, `GET /propiedades`).

## Cierre
`tipo_operacion` (venta | renta), `eb_id`, `precio_lista`, `precio_cierre`, `comision_monto`, `comision_pct`, `lado` (vendedor | comprador | ambos), `tipo_vendedor` (particular | desarrollador | inmobiliaria/asesor | otro), `fecha_cierre`, `dias_hasta_cierre` (calculado), `forma_pago` (contado | crédito bancario | Infonavit/Fovissste | mixto), y **copia de la propiedad al cierre**: `zona`, `fraccionamiento`, `tipo_inmueble`, `condicion`, `recamaras`, `banos`, `m2_construccion`, `m2_terreno`, `caracteristicas` (multi). Relación Cierre → Búsqueda → Lead. Si la operación se cae: Búsqueda con `motivo_caida`, sin Cierre.

## Catálogo de características (multi-enum compartido)
Empezar con: roof garden, recámara en planta baja, jardín, terraza, alberca, cuarto de servicio, bodega, estudio, cochera techada (1/2/3+), seguridad 24 h, casa club, mascotas, vista, esquina. Ampliable; cada valor es un slug estable (`roof_garden`, `recamara_pb`…).

## Sitio (verificar antes de empezar)
- Cada característica del catálogo debe existir como término de `amenidad` (y ser consultable en `GET /propiedades`) para copiarla al Cierre y a «Propiedad mostrada» sin teclearla. Falta confirmar en el código qué términos existen, y si hay campo/taxonomía de **fraccionamiento** y de **condición** (nueva | usada | desarrollo); si no existen, proponer agregarlos al CPT (`includes/cpt.php`) y a `bh_prop()`/contrato de datos.
- Comisión y precio de cierre **no** van al sitio ([../DECISIONES.md](../DECISIONES.md)); `bh_priv_commission` es la de captación y tampoco sale a BigQuery.
- `lead_generado` ya envía `lead_id` = `ref_code`; no requiere cambios para la unión.
