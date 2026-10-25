# n8n

n8n corre en el mismo VPS (Docker, `127.0.0.1:5678`, proxy inverso de CloudPanel en `n8n.besthomeqro.com`). Infraestructura: [../servidor/ARQUITECTURA-VPS.md](../servidor/ARQUITECTURA-VPS.md). Un archivo por automatización:

| Automatización | Archivo | Estado |
|---|---|---|
| Alta de leads en el CRM | [flujo-leads.md](flujo-leads.md) | Pendiente |
| WhatsApp ↔ referencia `R-XXXXXX` | [whatsapp-ref.md](whatsapp-ref.md) | Pendiente |
| Conversiones offline (Google Ads / Meta) | [conversiones-offline.md](conversiones-offline.md) | Pendiente |
| Análisis: BigQuery y Looker Studio | [bigquery-looker.md](bigquery-looker.md) | Pendiente |
| Asistente de IA vía MCP | [asistente-ia-mcp.md](asistente-ia-mcp.md) | Pendiente |

## Instalación
n8n 2.44.6 + Postgres 16 ya instalados en `~/n8n` (solo `127.0.0.1:5678`, 1 GB / 1 CPU, backup diario 3:00); detalle en [../servidor/ESTADO-ACTUAL.md](../servidor/ESTADO-ACTUAL.md).
- [x] Docker en `127.0.0.1:5678` (acceso por túnel SSH).
- [ ] Proxy inverso HTTPS en CloudPanel (`n8n.` con Access y `api.` solo `/webhook/`) cuando exista el dominio; `/etc/hosts` apuntando a 127.0.0.1.
- [ ] Panel protegido; ejecuciones fallidas guardadas; Error Workflow con alerta.
