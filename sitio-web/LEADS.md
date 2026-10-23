# Leads: outbox, despacho y atribución

Parte de `includes/leads.php`, `includes/leads-admin.php` y `includes/attr.php` del plugin ([PLUGIN.md](PLUGIN.md)). El envío al CRM lo recibe n8n: ver [../n8n/flujo-leads.md](../n8n/flujo-leads.md).

**Endpoint** `POST /wp-json/besthome/v1/lead` → `{ok, ref_code}` o `{ok:false, error}`. `type`: `formulario` | `whatsapp` (si falta: `formulario`; los valores antiguos `contacto` / `ficha` se aceptan y se guardan como `formulario`). El contexto del formulario va en `payload.context` (`ficha` si hay propiedad resuelta, si no `contacto`). `property_id` acepta ID de WordPress o `eb_id`. IP con `bh_client_ip()` (mu-plugin; `HTTP_CF_CONNECTING_IP` solo con `BH_BEHIND_CF === true`).
- `formulario`: honeypot `website`, 20 s por IP, Turnstile salvo entorno `local`, `name` + `phone` o `email`. Teléfono MX → `+52XXXXXXXXXX`; con `+` de otro país → `+XXXXXXXX` (8–15 dígitos); menos de 8 dígitos → 400. Email en minúsculas.
- `whatsapp`: honeypot y 2 s por IP; sin nombre, teléfono ni correo. `ref_code` `R-XXXXXX` (el navegador lo genera y se conserva si es válido; si ya existe en la tabla, el registro se descarta). Solo registro: **no se envía a n8n por iniciativa del sitio**; n8n lo consulta cuando llega un mensaje con `Ref:` (ver «Consulta para n8n»).
- Payload: llaves históricas de primer nivel (`event` … `fbclid`, utm = último contacto) + `uuid`, `type`, `context`, `ref_code`, `created_at`, `property`, `qualification_option`, `origin` (`data-wa-origin`), `attribution` (`last` + `gbraid`/`wbraid`, `first` de la cookie `bh_attr` o `attribution_first`, `_fbc`, `_fbp`, `_ga` crudo, `ga_client_id` = `_ga` sin `/^GA\d+\.\d+\./` (= `user_pseudo_id` de BigQuery) y `fbc_synthetic` = `fb.1.{ms}.{fbclid}` solo si falta `_fbc`: fbclid del último contacto con `fbclid_at`, o del primero con `first_visit_at`; sin marca, la del lead; nunca reemplaza un `_fbc` real), `journey`, `consent` (`BH_PRIVACY_VERSION`, def. `'1'`), `technical` (`ip_hash` HMAC-SHA256 con `wp_salt('auth')`, `user_agent` ≤ 255).

**Tabla** `{prefix}bh_leads` (dbDelta; opción `bh_leads_db_version` = `1.2`; se crea en la activación y en `admin_init` si la versión difiere): `id`, `uuid` (UNIQUE), `ref_code` (**UNIQUE**; en formularios, si colisiona se genera otro código y se reintenta hasta 5 veces), `created_at` (UTC), `type`, `status`, `attempts`, `next_retry_at`, `locked_until`, `first_attempt_at` (primer intento real de envío), `last_error`, `crm_id`, `synced_at`, `purged_at` (cuándo se borraron los datos personales), `name`, `phone`, `email`, `post_id`, `eb_id`, `utm_source`, `utm_campaign`, `ip_hash`, `payload_json`. Índices `(status, next_retry_at)` y `created_at`.

**Estados**: `pending` (por enviar o en espera de reintento) → `sending` (reclamado, candado `locked_until` 2 min) → `synced` | `failed`; `local` = WhatsApp aún sin confirmar en el CRM (no se despacha). `synced` también lo fija n8n con `POST /lead/{ref}/crm` (formularios o WhatsApp). Sin configuración del webhook (URL, clave o https fuera de local) el lead queda `pending` con `last_error='config'`, sin sumar `attempts` ni fijar `first_attempt_at` (no caduca), y se revisa cada 15 min (`next_retry_at` = UTC + 15 min). Mientras haya leads así: aviso en el Escritorio y en Propiedades › Leads, y como máximo un correo al día (transient `bh_lead_config_alert`, 24 h; aparte de `bh_lead_alert_lock`, 1 h).

**Despacho** `BH_EB_Leads::dispatch_lead($id, $timeout, $manual = false)`: tras insertar, en `shutdown` (`ignore_user_abort`; con `fastcgi_finish_request` timeout 15 s, sin él 4 s). Reclamo atómico con `UPDATE … WHERE status='pending' OR (status='sending' AND locked_until < UTC)`; POST de `payload_json` con `X-Automation-Key`, `redirection` 0.
- 2xx → `synced` (+ `crm_id` si la respuesta lo trae).
- 400 / 422 → `failed` + alerta.
- 401/403/404/408/429/5xx/WP_Error/otro → `attempts++`, `pending` con espera 5 min / 15 min / 1 h / 6 h; si pasaron más de 48 h desde `first_attempt_at` (sin él, `created_at`) y no es manual → `failed` + alerta; 401/403/404 también alertan.
- Alertas por `wp_mail` sin datos personales (id, uuid, código, enlace al admin), máximo 1 por hora (transient `bh_lead_alert_lock`). Si el insert falla se envía el lead completo por correo (sin límite) y se responde 500.

**Atribución** `POST /wp-json/besthome/v1/attr` (`BH_EB_Attr`): emite una sola vez la cookie `bh_attr` (JSON de utm_*, `gclid`, `gbraid`, `wbraid`, `fbclid`, `landing_page`, `referrer`, `first_visit_at`; 90 días, `SameSite=Lax`, legible por JS); si ya existe responde 204 sin tocarla; 5 s por IP. Es `attribution.first` del lead; `attribution.last` = parámetros del body o cookies del mismo nombre.

**Vista Propiedades › Leads** (`includes/leads-admin.php`, capacidad `bh_view_leads`): 50 por página, filtro por estado (todos | pending | synced | failed | local) y búsqueda en `ref_code`, nombre, teléfono y correo. Columnas: fecha (zona de WP), tipo, origen, estado, contacto + `ref_code`, propiedad (eb_id → editar), utm_source / utm_campaign, intentos + último error (o `crm_id`). Acciones: **Reintentar** (pending/failed → `pending`, `locked_until` NULL y `dispatch_lead($id, 15, true)`; aviso con el resultado), **Ver JSON** (`<details>`), **Eliminar** (ARCO: `confirm()` y borrado de la fila).

**Cron** `bh_eb_retry_leads` (intervalo `bh_leads_five`, 300 s; se programa en la activación y en `admin_init`, se quita al desactivar): hasta 10 leads `pending` con `next_retry_at <= UTC` o `sending` con candado vencido, cada uno con `dispatch_lead($id, 15)`.

## Consulta para n8n (WhatsApp) y confirmación del CRM

Autenticados con `X-BH-Key` (`BH_API_KEY`, la misma llave del catálogo; ver [API-CATALOGO.md](API-CATALOGO.md)).

- `GET /wp-json/besthome/v1/lead/{ref_code}` → `{ok, lead}`; 404 si no existe. Devuelve el payload sin `name`, `phone`, `email`, `message`, `technical` ni `consent` (eso vive en el CRM): `uuid`, `ref_code`, `type`, `context`, `status`, `created_at`, `synced_at`, `crm_id`, `property`, `qualification_option`, `origin`, utm, `attribution` (con `ga_client_id`, `_fbc`, `fbc_synthetic`), `journey`. No cambia estado.
- `POST /wp-json/besthome/v1/lead/{ref_code}/crm` con `{"crm_id": "…"}` → `{ok:true}` (400 sin `crm_id`, 404 si no existe). Marca `synced`, guarda `crm_id` y `synced_at`. Sirve para confirmar que el lead de WhatsApp quedó en el CRM (así deja de ser candidato a borrado) y como confirmación de un formulario.
- Flujo WhatsApp: clic → fila `local` → el cliente envía el mensaje con `Ref: R-XXXXXX` → n8n lee el mensaje (Evolution API), hace `GET`, crea/actualiza el contacto en el CRM guardando `ref_code` → `POST …/crm`.

## Retención (cron diario `bh_eb_purge_leads`)

| Qué | Regla |
|---|---|
| `whatsapp` en `local` (nadie escribió ni n8n confirmó) | Se **borra** a los 30 días de `created_at`. |
| `whatsapp` en `synced` | Se conserva (no tiene datos personales): historial para BigQuery / Looker Studio. |
| `formulario` en `synced` | A los 30 días de `synced_at` se vacían `name`, `phone`, `email` y, en `payload_json`, `name`, `phone`, `email`, `message`; se pone `purged_at`. Se conservan `ref_code`, `uuid`, atribución, propiedad, `crm_id`. |
| `formulario` en `failed` | Igual, a los 90 días de `created_at`. |
| `formulario` en `pending` | Nunca se purga (sigue esperando envío). |

Un lead con `purged_at` no se reenvía (`dispatch_lead()` lo omite). Los datos personales viven en el CRM, no aquí. Pendiente: reflejar el plazo en el aviso de privacidad y subir `BH_PRIVACY_VERSION`.
