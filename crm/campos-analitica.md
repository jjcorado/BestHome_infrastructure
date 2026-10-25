# Campos de analítica: racional

La lista vigente para crear está en [campos-espocrm.md](campos-espocrm.md) y el diseño base en [DISENO-ESPOCRM.md](DISENO-ESPOCRM.md). Este archivo solo guarda el porqué.

- **Catálogos en vez de texto** para que se puedan agrupar en Looker; el texto libre se queda en el CRM.
- **Copia al momento** de zona, fraccionamiento, tipo, condición, m² y precio en Visita y Cierre: la propiedad puede cambiar o archivarse.
- **Perdidos y sin respuesta se conservan** con su fuente completa (para medir qué canal trae contactos que no contestan o buscaban otra cosa). `motivo_perdida` agrupa en calidad del lead (marketing) vs falla de venta (comercial) en las vistas de BigQuery.
- **Primera respuesta y reasignación** se registran solas (n8n, webhooks y, más adelante, Evolution API), nunca a mano.
- **Catálogo de características** compartido con el sitio (`amenidad`) para copiar lo que se vende sin teclearlo.
- Comisión y precio de cierre viven en el CRM ([../DECISIONES.md](../DECISIONES.md)); `bh_priv_commission` del sitio (captación) no sale a BigQuery.
