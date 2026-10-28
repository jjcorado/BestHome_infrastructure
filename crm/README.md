# CRM

Estado: **EspoCRM 10.0.11 instalado** en el VPS (solo por túnel SSH, backup diario 3:10) y con la estructura base configurada (ver [CONFIGURACION.md](CONFIGURACION.md); infraestructura en [../servidor/ESTADO-ACTUAL.md](../servidor/ESTADO-ACTUAL.md)). Criterios y opciones evaluadas: [ELECCION-CRM.md](ELECCION-CRM.md).

Reglas ya decididas ([../DECISIONES.md](../DECISIONES.md)): ventas, precio de cierre y comisiones de cada operación viven en el CRM, no en el sitio. El sitio es la fuente de verdad de los leads (outbox) y n8n los lleva al CRM ([../n8n/flujo-leads.md](../n8n/flujo-leads.md)).

Diseño de entidades, etapas y automatizaciones: [DISENO-ESPOCRM.md](DISENO-ESPOCRM.md) (v3: sin Lead). Campos y permisos para analítica: [campos-espocrm.md](campos-espocrm.md).

Campos personalizados requeridos (ahora en la Búsqueda): `lead_id`, `ref_code`, `client_id` (`_ga`), `gclid` / `gbraid` / `wbraid`, `_fbc` / `_fbp`, UTMs first y last.

Racional de los campos de analítica: [campos-analitica.md](campos-analitica.md).
Baja de marketing, cancelación (ARCO) y bloqueo legal: [privacidad-arco.md](privacidad-arco.md).
