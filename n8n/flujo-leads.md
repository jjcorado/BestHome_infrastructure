# Flujo de leads → CRM

Origen: outbox `wp_bh_leads` del sitio ([../sitio-web/LEADS.md](../sitio-web/LEADS.md)). Estado: **pendiente** (Fase 3).

- [ ] Flujo de leads: webhook con Header Auth (`X-Automation-Key`) → deduplicar en CRM por `uuid`/teléfono/correo → crear oportunidad → responder 200 con `{"ok": true, "crm_id": "…"}` **solo después** de escribir en el CRM. Leads `type=whatsapp` no crean oportunidad (solo atribución).
- [ ] Cambiar `BH_WEBHOOK_LEAD_URL` de Make/temporal a n8n; apagar Make tras validar con leads reales.

Mientras `BH_WEBHOOK_LEAD_URL` no esté configurada, los leads esperan en `pending` (`last_error='config'`) sin caducar y se envían solos al configurarla.
