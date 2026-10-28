# CAMBIOS

Bitácora de cambios de **infraestructura y documentación** (los cambios de código viven en los repos del tema y del plugin). Más reciente arriba.

## 2026-10-26
- `motivo_rechazo` en Visita: visible y obligatorio solo si «no le gustó». Catálogo de características = las 10 amenidades del sitio. Nuevo `PLAN-IMPLEMENTACION.md` revisado contra el código del plugin (la API no busca por ID, oculta el precio, no hay condición ni bloqueo de borrado).
- Visita: calificación de 4 niveles y un solo comentario; se conserva el Recorrido (es la Reunión del calendario). Correo: EspoCRM Mass Email con Brevo como SMTP; baja de correo nativa (sin flujo de n8n) y separada de ARCO. Historial de precio por evento en WordPress. Plan de n8n pasa a seis flujos.
- Tras la revisión de Gemini: el espejo a BigQuery pasa a webhooks por evento (etapas, asesor, registros y cambios de precio) con reconciliación semanal; confirmación humana e IDs verificados en el asistente MCP; puntos para el asesor legal; `motivo_rechazo` opcional en Visita queda pendiente de decisión.
- Se descarta la foto diaria del catálogo en BigQuery (solo, si hace falta, cambios de propiedades modificadas). La consulta de propiedades es por ID exacto (`eb_id`), no por texto.
- Nuevos `n8n/plan-automatizaciones.md` (siete flujos, reglas para el KVM 2, orden sin Evolution) y `n8n/asistente-ia-mcp.md` rediseñado como prioridad 1. La primera respuesta pasa a fórmula de EspoCRM y el historial de etapas a una corrida nocturna.
- Nuevo `n8n/correo-marketing.md` (boletín, agradecimiento y reseña, recordatorio de escritura, recomendaciones; proveedor, consentimiento y flujos). Decisiones: `fecha_cierre` = escrituración, `no_contactar_marketing` solo lectura para asesores, importación de contactos de EasyBroker por CSV con revisión de duplicados.

## 2026-10-25
- Se incorpora al repo el diseño v3 de EspoCRM (`crm/DISENO-ESPOCRM.md`, sin Lead) y se realinea todo con él: `crm/campos-espocrm.md` (campos sobre v3), `crm/recorridos.md` (Recorrido = Reunión + Visita), `crm/privacidad-arco.md`, `n8n/bigquery-looker.md` y nueva `crm/REVISION-INTEGRAL.md`.

## 2026-10-24
- (2026-10-25) Lista final de campos y permisos de EspoCRM en `crm/campos-espocrm.md`; el interés general del recorrido se elimina (redundante con la reacción); `fecha_cierre` exacta; la anonimización usa IDs nuevos consistentes entre tablas.
- Cierre simplificado: `ingreso_inmobiliaria`, `comision_pct` opcional y `compartida_otra_inmobiliaria` (sí/no), sin `comision_cobrada`.
- Se diseña la analítica con BigQuery y Looker Studio: `n8n/bigquery-looker.md` (fuentes, flujo CRM → BigQuery seudonimizado, unión por `ref_code`, costos, privacidad, orden de pasos) y `crm/campos-analitica.md` (campos y entidades del CRM, incl. «Propiedad mostrada»). Solo documentación.
- Se diseña el registro de recorridos (`crm/recorridos.md`: hoja móvil, entidades Recorrido y Parada) en lugar de «Propiedad mostrada», y se añaden reportes objetivo (CAC por rango de precio, demanda vs. oferta, propiedades estancadas) a `n8n/bigquery-looker.md`.
- Principios de la capa de datos (independiente de la herramienta, inventario solo desde WordPress, n8n único integrador). Recorridos ya no consultan EasyBroker.
- Recorridos rediseñados dentro del CRM (Recorrido → Visita, reacción de 3 niveles, comentarios abiertos); se descarta la hoja en wp-admin. Se añaden leads perdidos/sin respuesta, configuración de n8n (sin historial de ejecuciones exitosas, cargas paginadas), `etapas_log` por evento y decisión de propiedades que no se borran.
- Analítica: primera atención y reasignación automáticas, catálogo `motivo_caida` (calidad del lead vs falla de venta), sin inventario por recorrido, métricas elegidas (sin LTV), y reglas de BigQuery para cierres (semana, precio y comisión redondeados). Luego Javier descartó el redondeo: precio y comisión exactos.
- Nuevo `crm/privacidad-arco.md`: baja de marketing, lista de exclusión, bloqueo legal en el CRM y anonimización en BigQuery al ejercer ARCO.
## 2026-10-25 (landings)
- Plantilla `Landing` (tema 0.1.54): HTML y CSS propios por página, sin header ni footer; shortcodes `[bh_casas]`, `[bh_formulario]`, `[bh_campos_ocultos]`; noindex por defecto. `lead_generado` trae `origin`. Plugin: `leads.php` guarda campos `x_*` en `payload.extra`. Reglas en `sitio-web/LANDINGS.md`.

## 2026-10-25 (catálogo: filtros)
- Tema (versión por confirmar al integrar con landings; ambos trabajos usaron 0.1.54): los chips de Precio, Construcción y Terreno muestran el rango elegido; «Limpiar filtros» en escritorio es el botón azul compacto de móvil; los pares mín/máx llevan botón «Aplicar» (ya no se envían al perder foco).
- Rendimiento: los filtros numéricos (`pmin/pmax/rec/ban/cmin/cmax/tmin/tmax`) se resuelven con una sola SQL sobre `postmeta` y se pasan como `post__in` (antes un JOIN por filtro). La visibilidad no se duplica en la réplica de `bh_catalog_amenity_slugs()` (`bh_vis_done`). Medido en Local: 6 filtros 18 s → 0.55 s.
- Nota de entorno: Local WP trae `pm.max_children = 2`; una consulta pesada congela el sitio. Conviene 6–8 en Local.

## 2026-10-23 (estado del VPS)
- Nuevo `servidor/ESTADO-ACTUAL.md` con el estado real: UFW, Fail2Ban, SSH solo con llave, CloudPanel por túnel SSH, Docker, n8n 2.44.6 y EspoCRM 10.0.11 con límites y backups locales.
- Acuerdos finales de dominios, Evolution API (solo lectura) y CRM registrados en `DECISIONES.md`, `servidor/ARQUITECTURA-VPS.md` y `evolution-api/README.md`.
- `PENDIENTES.md` y `servidor/FASE2-MIGRACION.md` actualizados con lo hecho y lo que falta.

## 2026-10-23
- Leads: `type` pasa a `formulario | whatsapp` (contexto en `payload.context`), `ref_code` UNIQUE, endpoints `GET /lead/{ref}` y `POST /lead/{ref}/crm` para n8n, y retención por cron (WhatsApp sin confirmar 30 días; datos personales de formularios sincronizados 30 días). Plugin: `includes/leads.php`, `bh_leads_db_version` 1.2. Ver `sitio-web/LEADS.md`.
- Se crea este repositorio de documentación, separado del tema.
- `PROJECT_ARCHITECTURE.md` y `PENDIENTES.md` (antes en `themes/besthome/`) se dividen por tema; `arquitectura-vps-cloudflare.md` pasa a `servidor/ARQUITECTURA-VPS.md`.
- Se documenta el módulo de reseñas de Google verificado contra el código (plugin 0.3.4, tema 0.1.50).
- Se añaden `evolution-api/` y `crm/` como carpetas de estado «pendiente».
