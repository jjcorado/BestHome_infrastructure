# Correo masivo y recordatorios

Estado: **diseño (2026-10-26)**. Depende de los campos de [../crm/campos-espocrm.md](../crm/campos-espocrm.md) y de la baja de marketing de [../crm/privacidad-arco.md](../crm/privacidad-arco.md). Sujeto a revisión legal (finalidades secundarias del aviso de privacidad).

## Qué se quiere enviar
| # | Correo | Naturaleza | Respeta la baja de marketing |
|---|---|---|---|
| A | Boletín mensual (artículos del blog + recomendaciones generales) | Marketing | **Sí** |
| B | Agradecimiento al cerrar + solicitud de reseña en Google Business | Relacional / marketing ligero | **Sí** (conservador) |
| C | Recordatorio a los 3, 4 y 6 meses de recoger escritura, certificados, hoja del Registro Público y factura del notario (+ reseña si procede) | **Servicio** de la operación | No el recordatorio; la solicitud de reseña sí |
| D | Recomendaciones por zona o propiedad a quien las pidió o marcó interés | Marketing | **Sí** |
Nunca se envía a quien tenga `bloqueo_legal` fuera de los avisos que la autoridad exija, ni a quien exista en la lista de exclusión.

## Consentimiento y listas (dos marcas en Contacto)
- `suscrito_boletin` (sí/no): se activa por el formulario de suscripción del sitio con **doble confirmación**, o cuando el asesor lo ofrece y el cliente acepta. Con esta marca se manda A y D.
- `no_contactar_marketing`: interruptor general que siempre gana. Los asesores la ven en solo lectura (decisión de Javier).
- Contactos heredados de EasyBroker: **no reciben marketing** hasta que interactúen o acepten (enviarles antes el aviso de privacidad). Sí pueden recibir mensajes uno a uno de su asesor.
- Cada correo de marketing lleva enlace de baja visible y cabecera de baja de un clic; la baja regresa a EspoCRM por n8n (webhook) y se agrega a la lista de exclusión.

## Dónde se envía: opciones
Los planes de correo del hosting limitan envíos por hora o día y no sirven para masivos; además enviar masivo desde el VPS daña la reputación del dominio y muchos proveedores bloquean el puerto 25. **Se envía por un servicio de reparto** y el CRM sigue siendo la fuente de verdad del consentimiento.
| Opción | Ventajas | Desventajas |
|---|---|---|
| **EspoCRM Mass Email** + SMTP de un proveedor (SES, Brevo, Mailgun) | Máxima integración: listas por filtros del CRM, baja en el mismo registro | Editor de plantillas sencillo, estadísticas pobres, límite de envíos por hora configurable |
| **n8n + Brevo** (recomendada) | Editor visual, páginas de baja y estadísticas listas, API y nodo en n8n, plan gratuito con tope diario (verificar cifras vigentes) | Dos sistemas: hay que sincronizar la baja y la lista |
| **n8n + Mailchimp** | Muy conocido | Plan gratuito más limitado y de pago más caro; mismo trabajo de sincronización |
| **Listmonk (autoalojado) + Amazon SES** | Control total, costo mínimo por envío | Más mantenimiento; hay que cuidar entregabilidad |
**Recomendación:** Brevo para empezar (envíos de A y D como campañas; B y C como correos transaccionales disparados por n8n), con el CRM como origen de la lista. Si el volumen crece o se quiere más control, se migra a Listmonk + SES sin cambiar los flujos, porque n8n es quien decide a quién se envía.
Necesario sea cual sea la opción: dominio autenticado (SPF, DKIM y DMARC), remitente en un subdominio (p. ej. `mail.besthomeqro.com`) para proteger el correo principal, y respuestas a un buzón atendido. Para saber el tamaño del plan: cuántos contactos con correo hay y cuántos correos al mes (el boletín mensual por 1 000 contactos son 1 000 envíos).

## Flujos de n8n
| Flujo | Disparo | Acción |
|---|---|---|
| Suscripción | Formulario del sitio (con Turnstile) | Correo de confirmación; al confirmar, `suscrito_boletin = sí` en el Contacto o se crea el Contacto |
| Boletín mensual | Cron mensual | Lee los últimos artículos del blog por la API de WordPress, arma el resumen, sincroniza la lista (solo `suscrito_boletin = sí`, sin baja ni bloqueo) y lanza la campaña |
| Baja | Webhook del proveedor | `no_contactar_marketing = sí`, fecha y canal; lista de exclusión (hash de correo) |
| Agradecimiento | Cierre llega a **escrituración** (`fecha_cierre`) | Correo de gracias; si no hay baja de marketing, incluye el enlace de reseña. Se guarda `resena_solicitada` (fecha) |
| Recordatorio de escritura | +3, +4 y +6 meses desde `fecha_cierre` si `escritura_recogida` no está marcada | Correo de servicio con la lista de documentos; se detiene al marcarla |
| Recomendaciones por zona | Mensual o por solicitud | Busca en la API de catálogo propiedades de la zona y el rango; solo a `suscrito_boletin = sí`; máximo 1 al mes por persona |
| Eventos de correo | Webhooks del proveedor | Envío, clic y baja a BigQuery por `cliente_hash` (las aperturas son poco fiables; se prioriza el clic) |
Los enlaces del correo llevan UTM (`utm_source=email`, `utm_campaign=<nombre>`) para verlos en GA4.

## Reseñas de Google
Pedir la reseña a **todos** los clientes, no solo a los satisfechos, y **sin ofrecer incentivos**: las políticas de Google lo prohíben. Usar el enlace directo de «escribir reseña» del perfil de Google Business.

## Campos nuevos (ver campos-espocrm.md)
Contacto: `suscrito_boletin`. Cierre: `escritura_recogida` (casilla, la marca el asesor o el administrador), `fecha_escritura_recogida`, `resena_solicitada` (fecha, la escribe n8n).
