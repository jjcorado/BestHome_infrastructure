# Elección del CRM

Es el cuello de botella actual de la Fase 3.

## Criterios obligatorios
- API completa de lectura y escritura (contactos, oportunidades, notas, tareas).
- Campos personalizados: `lead_id`, `ref_code`, `client_id` (`_ga`), `gclid`/`gbraid`/`wbraid`, `_fbc`/`_fbp`, UTMs first y last.
- Webhooks al cambiar etapa de una oportunidad.
- Permisos por usuario/asesor.
- Registro de conversaciones de WhatsApp (nativo o vía n8n).


## Opciones evaluadas
- Odoo Online gratis: **descartado** (sin API externa en el plan gratuito; solo en Custom).
- HubSpot, Odoo de pago: pendientes de evaluar costo.
- **EspoCRM (open source, autoalojado):** candidato. API REST completa, webhooks nativos, campos y entidades personalizables. Los webhooks de cambio de etapa vienen en la versión gratuita; los workflows los hace n8n (sin Advanced Pack, que es de pago). Puede probarse en el mismo KVM 2 (es ligero: PHP + MySQL); el cambio a KVM 4 solo si el servidor lo pide. Implica administrar uno mismo seguridad, actualizaciones y respaldos de datos de clientes. Verificar opciones de app móvil.

## Decisión (2026-10-23)
Se eligió EspoCRM autoalojado y ya está instalado en Docker (`~/espocrm`). Pendiente: campos personalizados, webhooks de etapa y flujo desde n8n.
