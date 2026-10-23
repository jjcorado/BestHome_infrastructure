# GTM, GA4 y conversiones (por preparar)

El contenedor actual lo usa el sitio viejo de EasyBroker: **no publicar antes de la migración**. Preparar en un espacio de trabajo nuevo. Eventos que emite el sitio: ver `main.js` en [ARQUITECTURA.md](ARQUITECTURA.md#assetsjsmainjs-hooks-por-atributo-data-).

- [ ] Variables de capa de datos: `lead_type`, `lead_id`.
- [ ] Activadores: `lead_generado` (contacto), `lead_generado` (whatsapp), `modal_whatsapp_open`, `form_lead_attempt`.
- [ ] Reasignar etiquetas existentes (GA4, Google Ads, Meta, OpenAI) a esos activadores. Borrar activadores viejos de clic (Click WA, Click id) y "Envío de formulario".
- [ ] Google Ads: una sola fuente por conversión (preferir etiqueta nativa). Formulario = principal; WhatsApp = secundaria. Las importadas desde GA4 pasan a secundarias.
- [ ] Conversiones mejoradas con `user_data` (correo y teléfono).
- [ ] Meta: `eventID = {{lead_id}}` en las conversiones (para deduplicar con Conversions API).
- [ ] Remarketing dinámico: `view_item` → Google Ads (ecommerce.items) y Meta `ViewContent` (`content_ids`).
- [ ] Eventos de interacción a GA4: `form_error`, `contact_click`, `ficha_pdf`, `share`, `media_open`, `search`, `select_item`, `social_click`.
- [ ] GA4: registrar dimensiones personalizadas (`lead_type`, `media_type`, `method`, `content_type`, `item_list_name`, `form_origin`, `reason`, `location`, `network`, `property_id`); retención de datos a 14 meses; anotación con la fecha del cambio.
- [ ] Nombres de etiquetas con la convención `Plataforma - Tipo - Qué`.
