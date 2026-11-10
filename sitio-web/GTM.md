# GTM, GA4 y conversiones

Contenedor **GTM-TDXS7V6X**. Versión publicada: **v19 «v3.0 - Estandarización de conversiones (lead_generado único)»** (2026-11-09). Eventos que emite el sitio: ver `main.js` en [ARQUITECTURA.md](ARQUITECTURA.md#assetsjsmainjs-hooks-por-atributo-data-).

## Regla de oro

**`lead_generado` es el ÚNICO evento de conversión.** Se distingue por el parámetro `lead_type` (`contacto` | `whatsapp`). `lead_id` = `ref_code` = `event_id` de Meta (deduplicación). `form_lead_attempt` (intento de contacto incompleto) es **solo para audiencias, nunca conversión**.

## Eventos del dataLayer

| Evento | Uso |
|---|---|
| `lead_generado` | Conversión. Parámetros: `lead_type`, `lead_id`, `property_id`, `origin`, `property_price`, `property_zone`, `property_operation`, `qualification_option`, `utm_*` |
| `form_lead_attempt` | Intento de formulario sin completar. Audiencia/remarketing, no conversión |
| `modal_whatsapp_open` | Apertura del modal de WhatsApp. Audiencia |
| `form_error` | Error de formulario; `reason`: `validacion`, `turnstile`, `servidor`, `red` |
| `view_item` | Remarketing dinámico |

## Variables

- `DLV - lead_type`, `DLV - lead_id`, `DLV - property_id`, `DLV - property_price`, `DLV - property_operation`.
- `JS - Lead Value` (valor de la conversión en MXN, solo pesos):
  - sin precio → 50
  - renta → precio / 100; venta → precio / 10,000
  - mínimo 50 (redondeado)

```js
function () {
  var p = Number({{DLV - property_price}}) || 0;
  var op = String({{DLV - property_operation}} || '').toLowerCase();
  if (p <= 0) { return 50; }
  var v = Math.round(p / (op === 'renta' ? 100 : 10000));
  return Math.max(v, 50);
}
```

## Activadores en uso

- `CE - lead_generado - contacto` (evento `lead_generado` y `DLV - lead_type` = `contacto`)
- `CE - lead_generado - whatsapp` (ídem `whatsapp`)
- `CE - form_lead_attempt`
- `CE - modal_whatsapp_open`

## Nombres estándar de conversión por plataforma

| Qué | GA4 | Google Ads | Meta | OpenAI Ads |
|---|---|---|---|---|
| Contacto por formulario | `lead_generado` + `lead_type=contacto` | conversión propia (etiqueta GTM, principal) | `Lead` con `lead_type=contacto` | `lead_form` |
| Contacto por WhatsApp | `lead_generado` + `lead_type=whatsapp` | conversión propia (etiqueta GTM, secundaria o principal según estrategia) | `Lead` con `lead_type=whatsapp` | `lead_whatsapp` |
| Intento incompleto | `form_lead_attempt` | (no es conversión) | `FormLeadAttempt` (personalizado, audiencia) | evento personalizado de audiencia |
| Apertura de modal WA | `modal_whatsapp_open` | (no es conversión) | `WhatsAppOpen` (audiencia) | — |

- Meta: `fbq('track','Lead',{value:{{JS - Lead Value}},currency:'MXN',lead_type:…,property_id:…},{eventID:{{DLV - lead_id}}})`.
- Google Ads: etiqueta de seguimiento de conversiones (ID 997762102) + Conversion Linker; una conversión por canal, con `transaction_id = lead_id`. Las conversiones viejas importadas de GA4 quedaron como **secundarias**; las de llamadas se deciden aparte.
- GA4: dimensión personalizada `Lead Type` (parámetro `lead_type`); nombres de dimensiones en inglés. Las demás dimensiones se registran según la lista de abajo.
- OpenAI Ads: `oaiq("measure","custom",{type:"custom"},{custom_event_name:"…"})`. Falta la etiqueta base del píxel (ver PENDIENTES).

## Eliminado en v19

- Etiquetas GA4 «Click WA» y «Lead Calificado Web».
- El conteo de WhatsApp ya no se hace por clic: se cuenta al confirmar el modal (`lead_generado` + `whatsapp`).

## Por hacer (ver también PENDIENTES.md)

- [ ] Borrar 10 activadores huérfanos en la próxima publicación: Cambio de URL, Click class, Click en WhatsApp (dinámico), Click id, Click WA, Click WA 2, click wa 3, click wa 4, Contacto_Enviado, Envío de formulario, Visibilidad - Formulario Enviado Exitoso.
- [ ] GA4: marcar `lead_generado` como evento clave y desmarcar los antiguos.
- [ ] Quitar de Google Ads las importaciones de GA4 viejas (tras 2–4 semanas de datos limpios) y limpiar inactivas (contact_us, whatsapp_click, form_start, form_submit).
- [ ] Conversiones mejoradas con `user_data` (correo y teléfono).
- [ ] Remarketing dinámico: `view_item` → Google Ads (ecommerce.items) y Meta `ViewContent` (`content_ids`).
- [ ] Eventos de interacción a GA4: `form_error`, `contact_click`, `ficha_pdf`, `share`, `media_open`, `search`, `select_item`, `social_click`.
- [ ] GA4: dimensiones personalizadas (`media_type`, `method`, `content_type`, `item_list_name`, `form_origin`, `reason`, `location`, `network`, `property_id`); retención a 14 meses.
- [ ] Campañas de Meta que optimicen por eventos personalizados viejos: moverlas a `Lead`.
- [ ] Verificar en el panel de OpenAI Ads los nombres `lead_whatsapp` y `lead_form`.
- [ ] Borrar los leads de prueba.
