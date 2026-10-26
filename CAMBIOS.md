# CAMBIOS

Bitácora de cambios de **infraestructura y documentación** (los cambios de código viven en los repos del tema y del plugin). Más reciente arriba.

## 2026-10-25 (catálogo: filtros)
- Tema 0.1.54: los chips de Precio, Construcción y Terreno muestran el rango elegido; «Limpiar filtros» en escritorio es el botón azul compacto de móvil; los pares mín/máx llevan botón «Aplicar» (ya no se envían al perder foco).
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
