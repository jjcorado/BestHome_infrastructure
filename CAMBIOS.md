# CAMBIOS

Bitácora de cambios de **infraestructura y documentación** (los cambios de código viven en los repos del tema y del plugin). Más reciente arriba.

## 2026-10-24
- (2026-10-25) Lista final de campos y permisos de EspoCRM en `crm/campos-espocrm.md`; el interés general del recorrido se elimina (redundante con la reacción); `fecha_cierre` exacta; la anonimización usa IDs nuevos consistentes entre tablas.
- Cierre simplificado: `ingreso_inmobiliaria`, `comision_pct` opcional y `compartida_otra_inmobiliaria` (sí/no), sin `comision_cobrada`.
- Se diseña la analítica con BigQuery y Looker Studio: `n8n/bigquery-looker.md` (fuentes, flujo CRM → BigQuery seudonimizado, unión por `ref_code`, costos, privacidad, orden de pasos) y `crm/campos-analitica.md` (campos y entidades del CRM, incl. «Propiedad mostrada»). Solo documentación.
- Se diseña el registro de recorridos (`crm/recorridos.md`: hoja móvil, entidades Recorrido y Parada) en lugar de «Propiedad mostrada», y se añaden reportes objetivo (CAC por rango de precio, demanda vs. oferta, propiedades estancadas) a `n8n/bigquery-looker.md`.
- Principios de la capa de datos (independiente de la herramienta, inventario solo desde WordPress, n8n único integrador). Recorridos ya no consultan EasyBroker.
- Recorridos rediseñados dentro del CRM (Recorrido → Visita, reacción de 3 niveles, comentarios abiertos); se descarta la hoja en wp-admin. Se añaden leads perdidos/sin respuesta, configuración de n8n (sin historial de ejecuciones exitosas, cargas paginadas), `etapas_log` por evento y decisión de propiedades que no se borran.
- Analítica: primera atención y reasignación automáticas, catálogo `motivo_caida` (calidad del lead vs falla de venta), sin inventario por recorrido, métricas elegidas (sin LTV), y reglas de BigQuery para cierres (semana, precio y comisión redondeados). Luego Javier descartó el redondeo: precio y comisión exactos.
- Nuevo `crm/privacidad-arco.md`: baja de marketing, lista de exclusión, bloqueo legal en el CRM y anonimización en BigQuery al ejercer ARCO.
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
