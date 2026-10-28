# Plan mínimo de automatizaciones (KVM 2, sin WhatsApp automático)

Estado: **diseño (2026-10-26)**. Restricciones de Javier: primero un asistente que lea y escriba en el CRM (Claude, y WhatsApp más adelante); todo debe poder funcionar **sin Evolution API**; no saturar el servidor.

## Recursos
KVM 2 (2 vCPU, 8 GB) compartido por WordPress, EspoCRM + MariaDB y n8n + Postgres; n8n está limitado a **1 GB / 1 CPU** ([../servidor/ESTADO-ACTUAL.md](../servidor/ESTADO-ACTUAL.md)). El cuello de botella son los núcleos, no la cantidad de flujos: con decenas de eventos al día, el número de ejecuciones es irrelevante; lo que pesa son cargas grandes, ejecuciones en paralelo y guardar historial.
Reglas:
- `EXECUTIONS_DATA_SAVE_ON_SUCCESS=none`, guardar solo errores y podar el historial (`EXECUTIONS_DATA_PRUNE=true`, retención corta).
- Límite de concurrencia bajo (`N8N_CONCURRENCY_PRODUCTION_LIMIT`, p. ej. 3).
- Nada de consultas masivas programadas: eventos pequeños en tiempo real y solo una reconciliación semanal, de noche, paginada en lotes de 100.
- Sin flujos que consulten cada minuto: webhooks y un par de cronómetros diarios.
- Lo que EspoCRM pueda hacer solo (fórmulas), no se hace en n8n.

## Qué hace WordPress por sí mismo
- **Historial de precio por evento**: cada vez que cambia el precio de una propiedad (por la sync o a mano), el plugin guarda una fila con ID, precio anterior, precio nuevo y fecha. Es un gancho al guardar la meta de precio en un archivo propio (sin tocar `sync.php`). Hoy solo existe `bh_priv_price_log`, que es texto privado; la tabla nueva solo lleva precios públicos y avisa por webhook a n8n, que la inserta en BigQuery.

## Qué hace EspoCRM por sí mismo (fórmulas, gratis)
- `fecha_primera_respuesta`: se llena sola cuando la etapa de la Búsqueda **sale de «Nuevo»** por primera vez (la medición es cuándo el asesor la marcó como atendida; vigilar con la vista «Nuevos sin responder»).
- `dias_ciclo` del Cierre (de `fecha_entrada` a `fecha_cierre`) y la sincronía de etapas de v3.
- Vistas guardadas y recordatorios.
Evolution API, cuando exista, solo refinará la hora real del primer mensaje; no es requisito.

## Flujos (seis, la mayoría de baja frecuencia)
Principio: lo que necesita reacción inmediata se **dispara por un evento** (webhook al guardar); lo que es análisis va en la corrida nocturna (si n8n se cae, se pone al corriente), y solo los recordatorios con fecha usan cron.
| # | Flujo | Disparo | Notas |
|---|---|---|---|
| 1 | **Asistente (MCP)** | Bajo demanda | **Prioridad 1.** Ver [asistente-ia-mcp.md](asistente-ia-mcp.md) |
| 2 | Alta de contacto del sitio | Webhook | Contacto + Búsqueda o nota (v3 6b) |
| 3 | Completar Visita / Cierre | Webhook de EspoCRM | Datos de la casa desde la API de catálogo; la reconciliación semanal repara las que queden sin datos |
| 4 | **Espejo a BigQuery por evento** | Webhooks de EspoCRM (`Búsqueda.fieldUpdate.etapa` y del asesor asignado → fila en `etapas_log`; alta o cambio de Búsqueda, Recorrido, Visita y Cierre → upsert de esa fila) y webhook de WordPress al cambiar un precio (→ `historial_precios`) | Cada evento es un JSON pequeño; n8n guarda primero el crudo y luego inserta. Un cron **semanal** ligero reconcilia por `modifiedAt` lo que se haya perdido y, el día 1, exporta `etapas_log` y `historial_precios` a Cloud Storage |
| 5 | Correos del día | Cron diario | Agradecimiento, recordatorios de escritura, pendientes de reseña; el boletín mensual el día fijado ([correo-marketing.md](correo-marketing.md)) |
| 6 | ARCO | Manual (casilla del administrador) | Bloqueo y anonimización ([../crm/privacidad-arco.md](../crm/privacidad-arco.md)) |
Se descartó leer el Stream de EspoCRM cada noche: el webhook por evento es más simple y en tiempo real; la reconciliación semanal cubre los eventos perdidos si n8n estuvo caído (verificar además si EspoCRM reintenta los webhooks fallidos).

## Orden
**0. Sin n8n (ya):** proyecto de Google Cloud, exportaciones de GA4 y Search Console, campos y fórmulas en EspoCRM, vistas guardadas.
**1. Asistente (flujo 1)** y alta de contacto (2).
**2.** Flujo 3, luego el espejo a BigQuery (4); el webhook de etapas conviene activarlo desde el primer día.
**3.** Correos (5; la baja de correo la maneja EspoCRM sola) y ARCO (6).
**4. Después:** Evolution API y el canal de WhatsApp del asistente.
