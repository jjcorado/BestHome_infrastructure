> **Documento de diseño vigente (v3).** Copia a este repositorio, el 2026-10-25, del archivo del proyecto `work/crm/diseno-espocrm.md`. Las extensiones para analítica, recorridos y privacidad están en la sección 10 y en [campos-espocrm.md](campos-espocrm.md). Donde el texto diga «Oportunidad» léase **Búsqueda** y «Operación», **Cierre**.

# Diseño de EspoCRM para Best Home (v3, dos niveles, versión ligera)

Basado en el Manual Operativo del Proceso de Ventas, Clientes Compradores. Propuesta aún no implementada.

> **Nombres definitivos (24 oct):** en este documento "Oportunidad" = **Búsqueda** y "Operación" = **Cierre** ("Cierre de venta" o "Cierre de renta" según el campo Tipo). Se renombran en el Gestor de entidades; el resto del texto se actualizará al implementarlo.

**Principio rector:** que los asesores lo usen es más importante que tenerlo completo. Pocos campos obligatorios, pocas etapas, pocas tareas automáticas, y que todo lo repetitivo lo llene el sistema (n8n), no la persona.

## 1. Modelo

| Entidad | Qué es | Quién la crea |
|---|---|---|
| **Contacto** | La persona: nombre, teléfono, correo, procedencia (Querétaro / foráneo), notas | n8n o el asesor |
| **Oportunidad** | La **búsqueda** del cliente: qué busca, presupuesto, forma de pago, origen y atribución | n8n o el asesor |
| **Operación** (entidad nueva) | Una **casa concreta** en negociación, hija de la Oportunidad | El asesor, cuando el cliente elige casa |

- **No se usa Lead** (se desactiva). Entra directo el Contacto + la Oportunidad en "Nuevo". Referidos: Oportunidad con origen "Referido" y campo "Referido por" (enlace a un Contacto).
- **No hay entidad Propiedad** en el CRM: la Operación guarda un código opcional (`EB-`/`BH-`), descripción libre y enlace.
- Si una casa se cae, la **Operación pasa a Caída**. **El asesor decide** el estatus de la Oportunidad (Seguimiento, Pausado o Perdido); el sistema no lo cambia solo, porque a veces el cliente ya no es viable (p. ej. crédito rechazado). Varias casas = varias Operaciones.
- **Recorrido** = la Reunión del calendario renombrada. Cada recorrido registra las propiedades mostradas y la retroalimentación. Así se sabe cuántos recorridos lleva un cliente y qué casas vio, sin campos de conteo manuales.
- **Filtro de spam:** n8n valida antes de crear (teléfono con formato válido, campos mínimos, Turnstile en formularios). Lo que aun así sea basura se marca **Perdido con motivo "Spam / no válido"** y se excluye de las métricas.

## 2. Etapas

**Oportunidad (búsqueda):**
1. Nuevo (hay que responder; incluye los que nunca contestan)
2. Contacto inicial (ya se respondió, se espera respuesta)
3. Perfilamiento (da información; opciones; se agenda el recorrido, que va dentro de esta etapa)
4. Seguimiento (después del recorrido, hasta que el cliente elige casa)
5. En operación (hay al menos una Operación activa)
6. Ganado
- Pausado / Perdido (motivo)

**Operación (casa):**
1. Negociación → 2. Oferta y apartado (1 %) → 3. Contrato promesa (10 %) → 4. Escrituración → 5. Entregada
- **Caída** con motivo: crédito no pasó, crédito menor al esperado, precio, términos de entrega, otro.

**Sincronía (fórmula de EspoCRM, sin trabajo del asesor):**
- Se crea una Operación activa → Oportunidad pasa a "En operación".
- Operación Entregada → Oportunidad "Ganado".
- Operación Caída y no quedan activas → **no se cambia la etapa sola**: n8n crea una tarea "Decidir: Seguimiento, Pausado o Perdido" para el asesor, y la Oportunidad aparece en la vista "En operación sin operación activa" hasta que decida. Si elige Seguimiento, n8n programa la tarea de retoma en 7–14 días.

## 3. Campos

**Obligatorios (solo estos):**
- Contacto: nombre y **teléfono o correo** (al menos uno; el formulario del sitio no obliga a dar teléfono para no perder contactos).
- Oportunidad: origen, asesor.
- Operación: propiedad (código o texto), precio.
- Caída: motivo.

**Opcionales (se llenan cuando se sabe):**
- Contacto: correo, procedencia, notas.
- Oportunidad: presupuesto máximo, forma de pago (propios, bancario, Infonavit, Fovissste, Cofinavit, mixto), horizonte (hasta 3, 3–6, 6–12, más de 12 meses), zonas, recámaras, nota libre de requerimientos, precalificación (no aplica / pendiente / aprobada), broker (texto).
- Operación: tipo de vendedor (Desarrollo, Constructor, Particular, Otra inmobiliaria), **comisión % estimada y editable** (sugerida por tipo), fecha y monto del apartado y vigencia, enganche, fechas de contrato promesa, escrituración y entrega. **Notaría** es opcional, pero se vuelve obligatoria al pasar la Operación a "Escrituración" (regla dinámica del Gestor de entidades).

**Los llena n8n (no el asesor):** `ref_code`, tipo de lead (`contacto` | `whatsapp`), propiedad de interés del sitio, atribución (utm_*, gclid, gbraid, wbraid, fbclid, landing_page, referrer, first_visit_at), hora de entrada y de primera respuesta.

**Recorrido (Reunión renombrada):** fecha, propiedades mostradas (códigos o texto), retroalimentación (positiva / neutra / negativa) y notas. No es una etapa.

**Operación, campos que llena n8n si hay código de propiedad** (para el análisis, sin entidad Propiedad): zona, tipo de propiedad y precio de lista, tomados del catálogo público del sitio (nunca `bh_priv_*`).

**Penalizaciones y devoluciones del apartado o enganche:** casos muy raros (menos de cinco en más de 10 años); se registran en las notas de la Operación, sin campo propio.

## 4. Automatizaciones (n8n), solo las esenciales
1. Alta del lead: sitio / EasyBroker / WhatsApp → Contacto + Oportunidad con atribución y asesor.
2. Tarea de seguimiento a las 24–48 h del recorrido.
3. Tarea "Decidir estatus" cuando una Operación cae y no quedan activas; si el asesor elige Seguimiento, tarea de retoma en 7–14 días.
3b. **Crédito:** sin tarea forzada a 24 h. La precalificación con el broker corre en paralelo; una vista guardada "Crédito sin precalificar" lo mantiene a la vista del asesor.
4. Recordatorio del apartado antes de los 14 días.
5. Tarea de "¿seguir, pausar o perder?" por inactividad, con plazo **por etapa** (por definir; no cambia la etapa sola).
6. Medición del SLA de primera respuesta (primero se mide; la alerta se decide después). **La hora de entrada es la del lead original** (cuándo el cliente envió el formulario o el mensaje), no la hora en que n8n recibe el webhook. Meta realista: 10–15 min.
6b. **Deduplicación antes de crear:** n8n busca primero en EspoCRM por teléfono normalizado a 10 dígitos y por correo en minúsculas. Si el Contacto existe y tiene una Oportunidad activa, solo agrega una nota (y tarea si aplica) a esa Oportunidad; si no tiene ninguna activa, crea una Oportunidad nueva para el mismo Contacto. Sin teléfono ni correo, crea el registro y lo marca para revisión.
6c. Consulta de propiedades: n8n usa el catálogo del propio sitio con `visibilidad=todas` (incluye privadas y archivadas; los datos no se borran), y el precio puede venir vacío si está oculto. Si no hay código (otra inmobiliaria, modelo sin alta), los datos se capturan a mano.
7. **Análisis de datos:** exportar Oportunidades, Operaciones y Recorridos desde n8n a Google Sheets / BigQuery + Looker Studio, y enviar conversiones a GA4 / Google Ads / Meta. Por definir cómo se combina con GA4.
8. Más adelante: Evolution API registrará en el CRM la fecha del último mensaje del asesor y del cliente como datos de apoyo. **Nunca cambiará etapas por sí sola**, y la inactividad se mide con la última actividad real del asesor (mensaje, llamada, recorrido, nota o tarea), no con respuestas sueltas del cliente como "ok". La interpretación del contenido de los mensajes queda para esa fase futura.

## 5. Medidas para que se use
- Vistas guardadas por asesor: "Mis oportunidades sin movimiento" y "Mis tareas de hoy".
- Revisión semanal corta con el gerente sobre esas vistas.
- Probar con un solo asesor 2 semanas antes de abrirlo a todos; recortar campos que no se usen.
- Revisar a los 30 días qué campos están vacíos y eliminarlos.

## 6. Roles
- **Administrador:** Javier y el gerente.
- **Asesor:** dos (un tercero después). Ven y editan lo suyo; sin borrar ni exportar.
- Brokers externos: sin acceso.
- Asignación inicial por turnos.

## 7. Privacidad
- Metas `bh_priv_*` nunca viajan al CRM, n8n ni webhooks. La comisión por operación vive en el CRM (así lo define PROJECT_ARCHITECTURE); `bh_priv_commission` del sitio es solo la de la captación.
- El CRM es interno: puede guardar ubicaciones y nombres de desarrollos; la regla de no enviarlos al cliente antes de la cita es operativa.

## 7b. Renta
La **Búsqueda es igual** para compra y renta. Lo que cambia es el **Cierre**, con un campo **Tipo: Venta | Renta** (por defecto Venta) y etapas que se muestran según el tipo (lógica dinámica de opciones):
- **Venta:** Negociación → Oferta y apartado → Contrato promesa → Escrituración → Entregada | Caída.
- **Renta (propuesta, a validar con el proceso real):** Negociación → Oferta y apartado → Calificación con la póliza → Contrato (datos y elaboración) → Firma y entrega → Entregada | Caída.
- Campos solo de venta (notaría, enganche 10 %, escrituración) se ocultan en renta. Campos de renta: empresa de la póliza, resultado de la calificación (pendiente, aprobada, rechazada), monto de renta, depósito.
- La firma del contrato es la entrega de la casa.

## 8. Orden de implementación
1. Usuarios y roles.
2. Desactivar Lead; ajustar Oportunidad (etapas y campos) y crear Operación en Administración › Gestor de entidades.
3. Fórmulas de sincronía y vistas guardadas.
4. Flujo n8n → EspoCRM con datos de prueba.
5. Piloto con un asesor.
6. Leads reales (requiere dominio y `api.` público).
7. Evolution API (después).

## 9. Pendiente de definir
- Plazos de inactividad por etapa.
- Ventana y canal de la alerta de SLA.
- Revisión final de campos tras releer el manual.

## 10. Extensiones para analítica, recorridos y privacidad (2026-10-25)
Sin cambiar los principios de este documento (sin Lead, pocos campos, lo repetitivo lo llena n8n):
- **Recorrido = Reunión renombrada**, con una entidad hija **Visita** (una por casa visitada): ID de la propiedad, calificación en 4 niveles (no le gustó / sin reacción / le gustó / le encantó) y un solo comentario. Sustituye a los campos «propiedades mostradas» y «retroalimentación» del Recorrido. Detalle: [recorridos.md](recorridos.md).
- **Campos nuevos en Contacto, Búsqueda y Cierre**, con permisos por campo, listados en [campos-espocrm.md](campos-espocrm.md). Los que llena el asesor son pocos y opcionales; el resto lo llena n8n.
- **Atribución y `ref_code`** viven en la Búsqueda (no hay Lead). Si el mismo cliente llega otra vez, los `ref_code` adicionales se guardan en la Búsqueda activa y cada contacto se registra como evento en BigQuery.
- **Privacidad:** baja de marketing, bloqueo legal y anonimización en BigQuery: [privacidad-arco.md](privacidad-arco.md).
- **Datos hacia BigQuery:** [../n8n/bigquery-looker.md](../n8n/bigquery-looker.md).
