# Flujo de leads → CRM

Origen: outbox `wp_bh_leads` del sitio ([../sitio-web/LEADS.md](../sitio-web/LEADS.md)). Estado: **pendiente** (Fase 3).

- [ ] Flujo de leads: webhook con Header Auth (`X-Automation-Key`) → deduplicar en CRM por `uuid`/teléfono/correo → crear oportunidad → responder 200 con `{"ok": true, "crm_id": "…"}` **solo después** de escribir en el CRM. Los leads de WhatsApp **no llegan por este webhook** (ver [whatsapp-ref.md](whatsapp-ref.md)).
- [ ] Cambiar `BH_WEBHOOK_LEAD_URL` de Make/temporal a n8n; apagar Make tras validar con leads reales.

Mientras `BH_WEBHOOK_LEAD_URL` no esté configurada, los leads esperan en `pending` (`last_error='config'`) sin caducar y se envían solos al configurarla.

## Arquitectura de leads (decidida 2026-10-23)

Principio: **el CRM es la fuente de verdad de los datos personales**; `wp_bh_leads` conserva referencia, atribución y estado. Detalle de campos y retención: [../sitio-web/LEADS.md](../sitio-web/LEADS.md).

| | Formulario | WhatsApp |
|---|---|---|
| Quién inicia | El sitio empuja a n8n (`BH_WEBHOOK_LEAD_URL`, `X-Automation-Key`) | n8n consulta al sitio cuando llega un mensaje con `Ref: R-XXXXXX` |
| Datos personales | Vienen en el payload → al CRM | Ya están en el chat/CRM; el sitio no los tiene |
| Confirmación | Respuesta 200 con `crm_id` (o `POST /lead/{ref}/crm`) | `POST /wp-json/besthome/v1/lead/{ref}/crm` con `crm_id` |
| Sin contacto | — | La fila `local` se borra a los 30 días |

Reglas para el flujo n8n:
- Guardar siempre `ref_code` (y `uuid`) en el contacto/oportunidad del CRM, también cuando el lead llega por otra vía y se asocia después.
- Formulario: crear oportunidad con la atribución del payload (`attribution`, `journey`, `property`); responder 200 con `crm_id` solo tras escribir en el CRM; si ya existe el contacto, enlazar la oportunidad nueva.
- WhatsApp: `GET /lead/{ref}` (llave `X-BH-Key`, la del catálogo) → comparar propiedad/presupuesto con el texto → crear o actualizar el contacto en el CRM → `POST /lead/{ref}/crm`. 404 = ref desconocido o ya purgado: crear el contacto sin atribución.
- Un mismo cliente que repite el paso genera otro `ref_code`; el anterior sin mensaje caduca solo.
- Contactos sin `Ref:` (recomendación, número en una manta, llamada): crear en el CRM con origen `manual`/`referido`, sin fila en el sitio.
- Cola de reintentos: revisar en el admin (Propiedades › Leads) los `pending`/`failed`; los `local` sin `crm_id` son WhatsApp aún no confirmados.
- Datos para BigQuery/Looker: copiar `wp_bh_leads` (atribución, propiedad, estado, `ref_code`) y unir con las etapas/ventas del CRM por `ref_code` ([bigquery-looker.md](bigquery-looker.md)); copiar antes de la purga si se necesitan datos personales (preferible no).
