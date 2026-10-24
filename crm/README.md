# CRM

Estado: **sin elegir** (candidato: EspoCRM autoalojado). Criterios y opciones evaluadas: [ELECCION-CRM.md](ELECCION-CRM.md).

Reglas ya decididas ([../DECISIONES.md](../DECISIONES.md)): ventas, precio de cierre y comisiones de cada operación viven en el CRM, no en el sitio. El sitio es la fuente de verdad de los leads (outbox) y n8n los lleva al CRM ([../n8n/flujo-leads.md](../n8n/flujo-leads.md)).

Campos personalizados requeridos: `lead_id`, `ref_code`, `client_id` (`_ga`), `gclid` / `gbraid` / `wbraid`, `_fbc` / `_fbp`, UTMs first y last.

Campos para analítica (BigQuery / Looker): [campos-analitica.md](campos-analitica.md).
