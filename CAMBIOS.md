# CAMBIOS

Bitácora de cambios de **infraestructura y documentación** (los cambios de código viven en los repos del tema y del plugin). Más reciente arriba.

## 2026-10-24
- Se diseña la analítica con BigQuery y Looker Studio: `n8n/bigquery-looker.md` (fuentes, flujo CRM → BigQuery seudonimizado, unión por `ref_code`, costos, privacidad, orden de pasos) y `crm/campos-analitica.md` (campos y entidades del CRM, incl. «Propiedad mostrada»). Solo documentación.
- Se diseña el registro de recorridos (`crm/recorridos.md`: hoja móvil, entidades Recorrido y Parada) en lugar de «Propiedad mostrada», y se añaden reportes objetivo (CAC por rango de precio, demanda vs. oferta, propiedades estancadas) a `n8n/bigquery-looker.md`.
- Principios de la capa de datos (independiente de la herramienta, inventario solo desde WordPress, n8n único integrador). Recorridos ya no consultan EasyBroker.

## 2026-10-23
- Leads: `type` pasa a `formulario | whatsapp` (contexto en `payload.context`), `ref_code` UNIQUE, endpoints `GET /lead/{ref}` y `POST /lead/{ref}/crm` para n8n, y retención por cron (WhatsApp sin confirmar 30 días; datos personales de formularios sincronizados 30 días). Plugin: `includes/leads.php`, `bh_leads_db_version` 1.2. Ver `sitio-web/LEADS.md`.
- Se crea este repositorio de documentación, separado del tema.
- `PROJECT_ARCHITECTURE.md` y `PENDIENTES.md` (antes en `themes/besthome/`) se dividen por tema; `arquitectura-vps-cloudflare.md` pasa a `servidor/ARQUITECTURA-VPS.md`.
- Se documenta el módulo de reseñas de Google verificado contra el código (plugin 0.3.4, tema 0.1.50).
- Se añaden `evolution-api/` y `crm/` como carpetas de estado «pendiente».
