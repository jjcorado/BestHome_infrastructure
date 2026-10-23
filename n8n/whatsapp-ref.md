# WhatsApp ↔ referencia (`Ref: R-XXXXXX`)

Estado: **pendiente**.

- Manual (desde el inicio): el asesor copia `Ref: R-XXXXXX` del chat al CRM; n8n completa la atribución.
- Automático (después): API de WhatsApp Business (Cloud API) en n8n lee el `Ref` del mensaje entrante. Verificar coexistencia con la app de WhatsApp Business en el mismo número.
- [x] Endpoint en el sitio (2026-10-23): `GET /wp-json/besthome/v1/lead/{ref_code}` (atribución sin datos de contacto) y `POST …/lead/{ref_code}/crm` para confirmar el guardado; detalle en [../sitio-web/LEADS.md](../sitio-web/LEADS.md).
- Los leads de WhatsApp **no se empujan a n8n**: n8n consulta solo cuando llega un mensaje con `Ref:`. Sin mensaje en 30 días, el registro se borra.
- [ ] Flujo n8n: leer `Ref: R-XXXXXX` del mensaje entrante → `GET` → crear/actualizar contacto en el CRM con `ref_code` → `POST …/crm` con el `crm_id`.
