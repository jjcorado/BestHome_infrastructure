# Análisis de datos: BigQuery y Looker Studio

Estado: **pendiente**.

- [ ] Proyecto de Google Cloud con facturación (el sandbox borra tablas a los 60 días).
- [ ] GA4 → BigQuery (exportación diaria).
- [ ] n8n copia `wp_bh_leads` (sin datos personales: los formularios los pierden a los 30 días de `synced_at`; copiar antes o excluirlos) y las etapas/ventas del CRM a BigQuery. Unión: `_ga` del lead = `user_pseudo_id` de GA4; `lead_id` = registro del CRM.
- [ ] Looker Studio: embudo canal → visitas → leads → citas → ventas, por campaña y zona.
