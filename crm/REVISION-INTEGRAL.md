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

## 3. Flujos de n8n
Se consolidaron en **seis flujos de baja frecuencia, disparados por eventos salvo los recordatorios con fecha** para no saturar el KVM 2, con el asistente MCP como prioridad 1 y sin depender de Evolution API: ver [../n8n/plan-automatizaciones.md](../n8n/plan-automatizaciones.md). Las fórmulas de EspoCRM hacen solas la primera respuesta, `dias_ciclo` y la sincronía de etapas.

## 4. Qué pedimos a los asesores (y qué no)
**Lo único que escriben:** nombre y teléfono o correo (v3); en cada Visita, el **ID de la casa** y tocar la reacción; el motivo al marcar Perdido. **Opcional:** comentario por visita, comentario del recorrido, tipo de inmueble, presupuesto mínimo.
**No escriben:** fechas de entrada y primera respuesta, atribución, datos de la casa, `dias_ciclo`, asesor anterior (todo n8n).
**Dejar para después del piloto (y activar solo si se usan):** `fraccionamientos`, `condicion_buscada`, `caracteristicas_deseadas`. Lo que se **vendió** igual queda completo en el Cierre porque n8n copia las características desde WordPress; lo que el cliente **quería** es lo opcional.

## 5. Prueba de uso con asesores (recomendada)
Sigue v3 §5: un asesor, 2 semanas, en celular. Guion de 5 tareas cronometradas: (1) abrir su Búsqueda, (2) crear un Recorrido pegando 3 IDs, (3) marcar vista y reacción de las 3 Visitas, (4) agregar un comentario general, (5) marcar una Búsqueda como Perdida con motivo.
Criterios de éxito: recorrido completo en **menos de 2 minutos**; **al menos 70 %** de los recorridos con Visitas; ningún campo obligatorio saltado por error; el asesor reporta que no tuvo que salir del CRM. A los 30 días: revisar campos vacíos y quitar los que no se usen (v3 §5). Pendiente: verificar cómo se ve EspoCRM en el celular (app o navegador) antes del piloto.

## 6. Decisiones (2026-10-26) y preguntas abiertas
**Decididas por Javier:**
1. `fecha_cierre` = fecha de escrituración (venta) o de firma (renta), porque no siempre hay entrega.
2. `no_contactar_marketing`: solo lectura para los asesores.
3. Contactos de EasyBroker: los importantes se capturan a mano al principio; después, importación desde archivo CSV revisando duplicados (teléfono a 10 dígitos y correo en minúsculas), como `origen_contacto = heredado` y sin marketing.
4. Las propiedades no se borran definitivamente (hay que confirmarlo en el plugin).
5. Consulta por ID exacto en la API de catálogo: todas las propiedades tienen ID único y permanente (`EB-XX0000` / `BH-XX0000`). Hoy la API solo busca por texto (`q`); falta agregar el parámetro `eb_id` que devuelva exactamente esa casa (cambio pequeño en `rest-catalogo.php`).
6. Correo masivo y recordatorios: [../n8n/correo-marketing.md](../n8n/correo-marketing.md).

**Siguen abiertas (revisar con contador o abogado, o después del piloto):**
- ¿Best Home es «actividad vulnerable» por intermediar inmuebles? Condiciona qué se puede borrar y por cuánto tiempo.
- Plazos de inactividad por etapa y alerta de SLA.
- Cuántos contactos con correo hay y volumen mensual de envíos, para elegir plan de correo.

## 7. Riesgos principales
- Pedir demasiado a los asesores: se mitiga con n8n, pocos campos y el piloto.
- Atribución incompleta (consentimiento, bloqueadores, cambio de dispositivo): se acepta y se documenta.
- Re-identificación: la barrera real es acceso restringido, sin identidad ni dirección exacta en BigQuery, y anonimización por solicitud.
- Cambios de etapa sin evento: sin el flujo 3 la historia no se reconstruye; activarlo desde el primer día.
- Dependencia de BigQuery: baja; el esquema vive en el repo y se exporta mensualmente.
