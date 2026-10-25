# Revisión integral: CRM, recorridos, analítica y privacidad (2026-10-25)

Une lo decidido en el diseño v3 del CRM ([DISENO-ESPOCRM.md](DISENO-ESPOCRM.md)) con lo agregado el 24–25 de octubre. Detalle: [campos-espocrm.md](campos-espocrm.md), [recorridos.md](recorridos.md), [privacidad-arco.md](privacidad-arco.md), [../n8n/bigquery-looker.md](../n8n/bigquery-looker.md).

## 1. Lo que cambió respecto a mis propuestas del 24 de octubre
| Antes (propuesta) | Ahora (alineado con v3) |
|---|---|
| Entidad Lead en el CRM | **No hay Lead.** Entra Contacto + Búsqueda. `ref_code` y atribución viven en la Búsqueda. |
| Etapas «Recorrido agendado/realizado/Negociación» | Se mantienen las etapas v3 (Nuevo, Contacto inicial, Perfilamiento, Seguimiento, En operación, Ganado, Pausado/Perdido). El recorrido se mide con la Reunión, no con una etapa. |
| Cierre con `lado`, `forma_pago`, `comision_cobrada` | Se usa la Operación/Cierre de v3 + `ingreso_inmobiliaria`, `compartida_otra_inmobiliaria`, `fecha_cierre`, `dias_ciclo`. La forma de pago se queda en la Búsqueda. |
| Recorrido nuevo con Visitas | El Recorrido es la **Reunión renombrada** (v3); se le agrega la entidad hija Visita. Se retiran sus campos «propiedades mostradas» y «retroalimentación». |
| Motivos de caída en Búsqueda | `motivo_perdida` fusiona el de v3 («Spam / no válido», excluido de métricas) con el catálogo nuevo. La Operación conserva su propio motivo de caída de v3. |

## 2. Cómo queda el modelo
Contacto (persona) → Búsqueda (qué busca, origen, atribución, etapa) → Recorrido (Reunión) → Visita (una casa), y Búsqueda → Cierre (una casa en negociación; etapa Negociación … Entregada | Caída). Sin entidad Propiedad: IDs `EB-`/`BH-` como texto, datos de la casa copiados por n8n desde WordPress.

**Cliente que vuelve a escribir.** El formulario o el mensaje trae otro `ref_code`. Por la deduplicación de v3 (6b), n8n no crea otra Búsqueda si hay una activa: agrega una nota y guarda el nuevo `ref_code` en `ref_codes_adicionales`. Para no perder la atribución de ese segundo contacto, n8n lo registra además como evento en la tabla `leads` de BigQuery (un registro por contacto con el sitio), ligado a la Búsqueda. Así el CRM sigue sin Lead y el análisis sigue viendo todos los contactos.

## 3. Flujos de n8n (estado y qué falta)
| # | Flujo | Disparo | Escribe | Estado |
|---|---|---|---|---|
| 1 | Alta de contacto del sitio | Webhook del formulario / mensaje WhatsApp con `Ref:` | Contacto + Búsqueda (o nota si hay activa) + evento `leads` | Diseñado (v3 §4.1, 6b); falta construir |
| 2 | Primera respuesta | Primer mensaje saliente (Evolution) o primer cambio a «Contacto inicial» | `fecha_primera_respuesta` | Diseñado; Evolution después |
| 3 | Eventos de etapa y asesor | Webhook de EspoCRM al cambiar `etapa` o asesor | `etapas_log` en BigQuery | **Nuevo**, diseñado |
| 4 | Completar Visita / Cierre | Webhook al guardar Visita (o Recorrido con IDs) o Cierre | Datos de la casa desde la API de catálogo de WordPress | **Nuevo** (confirmar filtro por ID en la API) |
| 5 | CRM → BigQuery | Diario, paginado, por `modified_at` | `busquedas`, `recorridos`, `visitas`, `cierres` (lista blanca, `cliente_hash`) | **Nuevo** |
| 6 | Foto diaria del catálogo | Cron | Tabla de catálogo en BigQuery | **Nuevo** |
| 7 | Exportación mensual a Cloud Storage | Cron | CSV/Parquet de `etapas_log` y catálogo | **Nuevo** |
| 8 | Baja de marketing | Casilla o enlace de baja | Marca en Contacto + lista de exclusión | **Nuevo**, antes de cualquier correo |
| 9 | ARCO: bloqueo y anonimización | Casilla del administrador | Bloqueo en CRM; `UPDATE` consistente en BigQuery; `arco_log` | **Nuevo** |
| 10 | Tareas de v3 (seguimiento 24–48 h, decidir estatus, apartado, inactividad) | Ver v3 | Tareas | Diseñado en v3 |
| 11 | Correos automáticos | — | — | **Después** de crear los campos y el flujo 8 |
Los flujos 3, 4 y 8 son los que más pesan para no perder datos; 5, 6 y 7 pueden esperar a tener datos reales.

## 4. Qué pedimos a los asesores (y qué no)
**Lo único que escriben:** nombre y teléfono o correo (v3); en cada Visita, el **ID de la casa** y tocar la reacción; el motivo al marcar Perdido. **Opcional:** comentario por visita, comentario del recorrido, tipo de inmueble, presupuesto mínimo.
**No escriben:** fechas de entrada y primera respuesta, atribución, datos de la casa, `dias_ciclo`, asesor anterior (todo n8n).
**Dejar para después del piloto (y activar solo si se usan):** `fraccionamientos`, `condicion_buscada`, `caracteristicas_deseadas`. Lo que se **vendió** igual queda completo en el Cierre porque n8n copia las características desde WordPress; lo que el cliente **quería** es lo opcional.

## 5. Prueba de uso con asesores (recomendada)
Sigue v3 §5: un asesor, 2 semanas, en celular. Guion de 5 tareas cronometradas: (1) abrir su Búsqueda, (2) crear un Recorrido pegando 3 IDs, (3) marcar vista y reacción de las 3 Visitas, (4) agregar un comentario general, (5) marcar una Búsqueda como Perdida con motivo.
Criterios de éxito: recorrido completo en **menos de 2 minutos**; **al menos 70 %** de los recorridos con Visitas; ningún campo obligatorio saltado por error; el asesor reporta que no tuvo que salir del CRM. A los 30 días: revisar campos vacíos y quitar los que no se usen (v3 §5). Pendiente: verificar cómo se ve EspoCRM en el celular (app o navegador) antes del piloto.

## 6. Preguntas abiertas (con mi recomendación)
1. **`fecha_cierre`**: ¿fecha de escrituración/firma (recomendado, cuando se ingresa la comisión) o de entrega?
2. **Asesores y `no_contactar_marketing`**: ¿solo lectura (recomendado, evita incluir a alguien en un envío) o oculto?
3. **Contactos heredados de EasyBroker**: ¿se importan al CRM? Recomendado: sí, como `origen_contacto = heredado`, con aviso de privacidad en el primer contacto y sin campañas masivas hasta tenerlo.
4. **¿Es Best Home «actividad vulnerable»** por intermediar inmuebles? Condiciona cuánto se puede borrar y por cuánto tiempo conservar; requiere contador o abogado.
5. **Plazos de inactividad por etapa** y **alerta de SLA** (pendientes de v3).
6. **Filtro por ID en la API de catálogo** (`q` con `bh_eb_id`) y confirmar que las propiedades no se pueden borrar definitivamente.

## 7. Riesgos principales
- Pedir demasiado a los asesores: se mitiga con n8n, pocos campos y el piloto.
- Atribución incompleta (consentimiento, bloqueadores, cambio de dispositivo): se acepta y se documenta.
- Re-identificación: la barrera real es acceso restringido, sin identidad ni dirección exacta en BigQuery, y anonimización por solicitud.
- Cambios de etapa sin evento: sin el flujo 3 la historia no se reconstruye; activarlo desde el primer día.
- Dependencia de BigQuery: baja; el esquema vive en el repo y se exporta mensualmente.
