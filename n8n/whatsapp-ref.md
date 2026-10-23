# WhatsApp ↔ referencia (`Ref: R-XXXXXX`)

Estado: **pendiente**.

- Manual (desde el inicio): el asesor copia `Ref: R-XXXXXX` del chat al CRM; n8n completa la atribución.
- Automático (después): API de WhatsApp Business (Cloud API) en n8n lee el `Ref` del mensaje entrante. Verificar coexistencia con la app de WhatsApp Business en el mismo número.
- [ ] Endpoint en el sitio, autenticado y solo para n8n: atribución por `ref_code` (sin datos de contacto).
