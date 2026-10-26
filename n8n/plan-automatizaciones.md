# Plan mínimo de automatizaciones (KVM 2, sin WhatsApp automático)

Estado: **diseño (2026-10-26)**. Restricciones de Javier: primero un asistente que lea y escriba en el CRM (Claude, y WhatsApp más adelante); todo debe poder funcionar **sin Evolution API**; no saturar el servidor.

## Recursos
KVM 2 (2 vCPU, 8 GB) compartido por WordPress, EspoCRM + MariaDB y n8n + Postgres; n8n está limitado a **1 GB / 1 CPU** ([../servidor/ESTADO-ACTUAL.md](../servidor/ESTADO-ACTUAL.md)). El cuello de botella son los núcleos, no la cantidad de flujos: con decenas de eventos al día, el número de ejecuciones es irrelevante; lo que pesa son cargas grandes, ejecuciones en paralelo y guardar historial.
Reglas:
- `EXECUTIONS_DATA_SAVE_ON_SUCCESS=none`, guardar solo errores y podar el historial (`EXECUTIONS_DATA_PRUNE=true`, retención corta).
- Límite de concurrencia bajo (`N8N_CONCURRENCY_PRODUCTION_LIMIT`, p. ej. 3).
- Cargas grandes **de noche y escalonadas** (el respaldo corre a las 3:00), en lotes de 100 y con una sola llamada de carga a BigQuery por lote, no fila por fila.
- Sin flujos que consulten cada minuto: webhooks y un par de cronómetros diarios.
- Lo que EspoCRM pueda hacer solo (fórmulas), no se hace en n8n.

## Qué hace EspoCRM por sí mismo (fórmulas, gratis)
- `fecha_primera_respuesta`: se llena sola cuando la etapa de la Búsqueda **sale de «Nuevo»** por primera vez (la medición es cuándo el asesor la marcó como atendida; vigilar con la vista «Nuevos sin responder»).
- `dias_ciclo` del Cierre (de `fecha_entrada` a `fecha_cierre`) y la sincronía de etapas de v3.
- Vistas guardadas y recordatorios.
Evolution API, cuando exista, solo refinará la hora real del primer mensaje; no es requisito.

## Flujos (siete, la mayoría de baja frecuencia)
| # | Flujo | Disparo | Notas |
|---|---|---|---|
| 1 | **Asistente (MCP)** | Bajo demanda | **Prioridad 1.** Ver [asistente-ia-mcp.md](asistente-ia-mcp.md) |
| 2 | Alta de contacto del sitio | Webhook | Contacto + Búsqueda o nota (v3 6b) |
| 3 | Completar Visita / Cierre | Webhook de EspoCRM | Datos de la casa desde la API de catálogo; la corrida nocturna repara las que queden sin datos |
| 4 | **Corrida nocturna única** | Cron 4:00 | Lee los cambios del día en EspoCRM (cambios de etapa y de asesor del Stream con auditoría activa) → `etapas_log`; sincroniza `busquedas`, `recorridos`, `visitas`, `cierres` a BigQuery; el día 1, exportación mensual a Cloud Storage de `etapas_log` |
| 5 | Correos del día | Cron diario | Agradecimiento, recordatorios de escritura, pendientes de reseña; el boletín mensual el día fijado ([correo-marketing.md](correo-marketing.md)) |
| 6 | Baja de marketing | Webhook del proveedor | Marca el Contacto y la lista de exclusión |
| 7 | ARCO | Manual (casilla del administrador) | Bloqueo y anonimización ([../crm/privacidad-arco.md](../crm/privacidad-arco.md)) |
La corrida nocturna lee el historial de EspoCRM en lugar de depender de un webhook por cada cambio: si n8n se cae un día, al siguiente se pone al corriente. (Verificar en la instalación que el Stream de EspoCRM es consultable por la API; si no, se usa un webhook para los cambios de etapa y asesor.)

## Orden
**0. Sin n8n (ya):** proyecto de Google Cloud, exportaciones de GA4 y Search Console, campos y fórmulas en EspoCRM, vistas guardadas.
**1. Asistente (flujo 1)** y alta de contacto (2).
**2.** Flujo 3, luego la corrida nocturna (4) cuando haya datos reales.
**3.** Correos (5, 6) y ARCO (7).
**4. Después:** Evolution API y el canal de WhatsApp del asistente.
