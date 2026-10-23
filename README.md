# Best Home Querétaro — documentación de infraestructura

Documentación del proyecto (besthomeqro.com), separada del código. Sin contraseñas, llaves ni tokens: esos valores viven solo en `wp-config.php`, `.env` y los paneles correspondientes.

Repos de código: tema `jjcorado/besthome-theme` · plugin `jjcorado/besthome-eb-sync`.

## Índice

| Carpeta / archivo | Contenido |
|---|---|
| [sitio-web/ARQUITECTURA.md](sitio-web/ARQUITECTURA.md) | Tema, plantillas, JS, CSS y reseñas de Google |
| [sitio-web/PLUGIN.md](sitio-web/PLUGIN.md) | Plugin de sincronización con EasyBroker: CPT, taxonomías, metas, hooks, sync |
| [sitio-web/LEADS.md](sitio-web/LEADS.md) | Endpoint de leads, outbox, reintentos, atribución |
| [sitio-web/CONTRATO-DATOS.md](sitio-web/CONTRATO-DATOS.md) | Campos de propiedad, IDs, estados, reglas de la sync |
| [sitio-web/API-CATALOGO.md](sitio-web/API-CATALOGO.md) | API REST de catálogo para n8n / IA |
| [sitio-web/GTM.md](sitio-web/GTM.md) | Plan de GTM / GA4 / conversiones |
| [servidor/](servidor/) | VPS, Cloudflare, Docker, CloudPanel y checklist de migración |
| [crm/](crm/) | Elección y diseño del CRM |
| [n8n/](n8n/) | Una automatización por archivo |
| [evolution-api/](evolution-api/) | WhatsApp vía Evolution API |
| [PENDIENTES.md](PENDIENTES.md) | Lo que falta (Fases 2 y 3) |
| [FUTURO.md](FUTURO.md) | Ideas sin fecha |
| [DECISIONES.md](DECISIONES.md) | Decisiones tomadas |
| [CAMBIOS.md](CAMBIOS.md) | Bitácora de cambios |

## Reglas
- Al cerrar una funcionalidad estructural, actualizar el archivo de su tema y `PENDIENTES.md` / `CAMBIOS.md`.
- Nunca pegar secretos aquí; describir solo el nombre de la constante o variable.
