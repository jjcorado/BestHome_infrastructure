# Plugin `besthome-easybroker-sync`

Ruta: `plugins/besthome-easybroker-sync/`. Los leads (outbox, atribución, vista admin) están en [LEADS.md](LEADS.md); el contrato de datos de propiedad en [CONTRATO-DATOS.md](CONTRATO-DATOS.md); la API de catálogo en [API-CATALOGO.md](API-CATALOGO.md); las reseñas de Google en [ARQUITECTURA.md](ARQUITECTURA.md#reseñas-de-google).

```
besthome-easybroker-sync.php   Bootstrap: constantes, requires, init de clases, activación/desactivación
includes/cpt.php               BH_EB_CPT          — CPT, taxonomías, metas, metaboxes, columnas, toggles AJAX
includes/api.php               BH_EB_API          — Cliente HTTP de EasyBroker
includes/sync.php              BH_EB_Sync         — Sincronización incremental/completa y mapeo
includes/leads.php             BH_EB_Leads        — Endpoint REST de leads
includes/leads-admin.php       BH_EB_Leads_Admin  — Propiedades › Leads
includes/attr.php              BH_EB_Attr         — Endpoint de atribución (cookie bh_attr)
includes/admin.php             BH_EB_Admin        — Página Ajustes › EasyBroker Sync
includes/price-checks.php      BH_EB_Price_Checks — Propiedades › Verificación de precios y tabla bh_price_checks
includes/rest-catalogo.php     BH_EB_Rest_Catalogo — API de catálogo para n8n / IA (ver API-CATALOGO.md)
includes/google-places.php     BH_EB_Google_Places — Reseñas de Google (cron diario; ver ARQUITECTURA.md)
mu/bh-security.php             Endurecimiento global y SMTP (cargado desde mu-plugins/bh-security.php)
README.txt                     Instalación y constantes de wp-config.php
```

### Configuración (`wp-config.php`)
`BH_EB_API_KEY` (o opción `bh_eb_api_key`), `BH_EB_STAGING` (→ `api.stagingeb.com`), `BH_WEBHOOK_LEAD_URL` + `BH_WEBHOOK_LEAD_KEY` (header `X-Automation-Key`), `BH_TURNSTILE_SECRET_KEY` / `BH_TURNSTILE_SITE_KEY`, `BH_LUXURY_HOUSE_MIN` (def. 7 500 000), `BH_LUXURY_LAND_MIN` (def. 3 000 000), `BH_GTM_ID` (tema; si existe, manda sobre el Personalizador), `BH_API_KEY` (API de catálogo), `BH_GOOGLE_PLACES_KEY` / `BH_GOOGLE_PLACE_ID` / `BH_GOOGLE_MAX_STALE_DAYS` (reseñas de Google). Los valores reales viven solo en `wp-config.php` del servidor, nunca en estos documentos.

### Hooks registrados
| Clase | Hook | Callback / propósito |
|---|---|---|
| bootstrap | `register_activation_hook` | registra CPT, `ensure_terms()`, tabla `bh_leads`, capacidad `bh_view_leads`, flush rewrites, `schedule()` (sync y leads) |
| bootstrap | `register_deactivation_hook` | `unschedule()` (sync y leads), flush rewrites |
| `BH_EB_CPT` | `init` | `register()` — CPT, taxonomías, metas |
| | `admin_init` | `maybe_seed()` — siembra términos (opción `bh_terms_seeded` = `'3'`) |
| | `add_meta_boxes` / `save_post_propiedad` | metaboxes “Clasificación” y “Control interno” |
| | `manage_propiedad_posts_columns` / `..._custom_column` | columnas (sin las nativas de fecha/autor/taxonomías): casilla, imagen 60×60 (destacada o 1.ª foto de EB), Título, ID, Precio, Operación, Zona, Tipo, Modificada, Orden (`bh_sort_date`), Destacada, Lujo, Visibilidad (Pública / Privada / Archivada + fecha y “EasyBroker” si la archivó la sync), Visitas |
| | `manage_edit-propiedad_sortable_columns` + `posts_clauses` | ordenar por Precio (`bh_price`), Zona (padre › colonia, LEFT JOIN de términos), Modificada, Orden (`bh_sort_date`, LEFT JOIN) y Visibilidad (`bh_visibility`, sin meta = `publica`) |
| | `post_row_actions` + `admin_footer-edit.php` | acciones bajo el título: “Copiar enlace” (URL pública; respaldo con `<textarea>` en HTTP), “PDF (Marca)” (`?print=brand`) y “PDF (Sin marca)” (`?print=nobrand`, nueva pestaña) y “Subir al inicio” (`admin-post.php?action=bh_bump`, nonce `bh_bump_{ID}` + `edit_post`: `bh_sort_date` = ahora UTC; también como botón AJAX en Verificación de precios; con la opción `bh_bump_on_price_drop` la sync lo hace sola cuando baja el precio) |
| | `add_meta_boxes` + `save_post_propiedad` (`save_badge`) | metabox lateral “Etiqueta Destacada”: `<input list>` + `<datalist>` (Roof garden, Cuarto de servicio, Recámara en planta baja, De una planta, Alberca en fraccionamiento, Amplio jardín) o texto libre ≤ 40; meta `_bh_badge` (vacío = se borra), nonce `bh_badge`. La tarjeta del catálogo solo muestra esta etiqueta (ya no hay “Roof Garden” automático). |
| | `rest_api_init` → `register_rest_field` | campos de solo lectura en `/wp-json/wp/v2/propiedad`: `bh_eb_id`, `bh_operation`, `bh_price` (null si el precio está oculto), `bh_bedrooms`, `bh_bathrooms`, `bh_half_bathrooms`, `bh_construction`, `bh_lot`, `bh_zona` (“Padre › Colonia”) |
| | `pre_get_posts` (prioridad 20) + `posts_search` | si `s` parece un ID (una palabra con dígitos: EB-JJ8368, JJ8368, 8368) también busca en `bh_eb_id` / `bh_internal_id`; aplica en wp-admin y en el catálogo (`?q=`) |
| | `wp_ajax_bh_toggle_meta` + `admin_footer-edit.php` | toggles AJAX Destacada/Lujo en la lista |
| `BH_EB_Sync` | `cron_schedules` | intervalo `bh_fifteen` (15 min) |
| | `bh_eb_incremental_sync` (cron 15 min) | `run('incremental')` |
| | `bh_eb_full_sync` (cron semanal) | `run('full')` |
| | `bh_eb_continue` (evento único) | continúa una sync parcial o tras 429 |
| | filtro `besthome_zone_parents` | personaliza zonas padre y sus palabras clave |
| `BH_EB_Leads` | `rest_api_init` | `POST /wp-json/besthome/v1/lead` |
| | `admin_init` | `maybe_install()` — tabla `bh_leads` y cron de reintentos |
| | `cron_schedules` / `bh_eb_retry_leads` | intervalo `bh_leads_five` (5 min) → `retry()` |
| `BH_EB_Leads_Admin` | `admin_init` / `admin_menu` | `add_cap()` (`bh_view_leads` al administrador si falta); submenú `edit.php?post_type=propiedad&page=bh-leads` |
| | `admin_post_bh_lead_{retry,delete}` | nonce por lead + `bh_view_leads` |
| `BH_EB_Attr` | `rest_api_init` | `POST /wp-json/besthome/v1/attr` |
| `BH_EB_Admin` | `admin_menu` | `options-general.php?page=bh-eb-sync` |
| | `admin_post_bh_eb_{sync_now,sync_full,force,test,save_key,save_price_settings}` | acciones de la página (nonce `bh_eb_action`, `manage_options`) |
| | `admin_notices` | `price_review_notice()` — aviso de propiedades con `bh_price_review = 1` en `edit-propiedad` |
| `BH_EB_Price_Checks` | `admin_init` / `admin_menu` | `maybe_install()` (tabla `bh_price_checks`); submenú `edit.php?post_type=propiedad&page=bh-price-checks` (`manage_options`) |
| | `wp_ajax_bh_pc_{save,verify,bump}` | edición en línea (`bh_priv_address`, `bh_priv_owner_name`, `bh_priv_commission`, `bh_priv_notes`), verificación sin cambio y subir al inicio (nonce `bh_pc`) |
| | `admin_post_bh_pc_export` | exportación CSV con escape anti-inyección de fórmulas |

### Custom Post Type
- **`propiedad`** — público, archivo y slug `propiedades`, `show_in_rest`, supports `title, editor, excerpt, thumbnail` (imagen destacada opcional), icono `dashicons-admin-home`. Slug de post estable: `sanitize_title(título)-{public_id}`.

### Taxonomías
| Taxonomía | Objetos | Jerárquica | Rewrite | Términos base |
|---|---|---|---|---|
| `operacion` | propiedad, post | no | `operacion` | venta, renta |
| `tipo_propiedad` | propiedad, post | no | `tipo` | casa, departamento, terreno |
| `zona` | propiedad, post | **sí** (padre › colonia) | `zona` | se crean al sincronizar |
| `amenidad` | propiedad | no | `amenidad` | una-planta, recamara-en-planta-baja, roof-garden, estudio-home-office, sala-de-tv, cuarto-de-juegos (“Salón de juegos”), cuarto-de-servicio, alberca-privada, alberca-en-fraccionamiento, terraza |

Zonas padre (`BH_EB_Sync::zone_parents()`): Juriquilla · Querétaro Moderno · Centro / Álamos / Campanario · Corregidora · Jurica · San Miguel de Allende. Coincidencia exacta normalizada (sin acentos) de cada parte de `location.name`; sin coincidencia → sin zona y se registra en el log.

### Campos meta (`propiedad`)
**Públicos** (`register_post_meta`, `show_in_rest`):

| Meta | Tipo | Origen | Notas |
|---|---|---|---|
| `bh_eb_id` | string | EB `public_id` | clave de upsert (`find_post()`) |
| `bh_internal_id` | string | EB | |
| `bh_operation` | string | EB `operations[].type` | `sale` / `rental` |
| `bh_price` | number | EB | 0 si `show_prices` es falso |
| `bh_show_price` | boolean | EB | |
| `bh_currency` | string | EB | def. MXN |
| `bh_bedrooms`, `bh_bathrooms`, `bh_half_bathrooms`, `bh_parking` | number | EB | |
| `bh_construction`, `bh_lot` | number | EB | m² |
| `bh_lat`, `bh_lng` | number | EB | mapa / JSON-LD |
| `bh_location_name` | string | EB `location.name` | base de la asignación de zona |
| `bh_status` | string | EB | `published` → `publish`, otro → `draft` |
| `bh_updated_at` | string | EB | omite detalle si no cambió |
| `bh_video_url`, `bh_tour_url` | string | EB o metabox | EB vacío no sobrescribe |
| `bh_floors` | number | EB | solo si > 0 |
| `bh_featured` | boolean | admin | destacada en Home |
| `bh_luxury` | boolean | auto/admin | auto: venta y precio ≥ umbral |
| `bh_luxury_manual` | boolean | admin | si `1`, la sync no recalcula lujo |
| `bh_lock_terms` | boolean | admin | si `1`, la sync no toca zona ni amenidades |
| `bh_views` | number | tema (beacon REST `besthome/v1/view/{id}`) | también en `post` |
| `bh_visibility_prev`, `bh_archived_at`, `bh_archived_by` | string, string (UTC), `sync` \| `manual` | sync o metabox | solo con `bh_visibility = archivada`; ver [Estados](CONTRATO-DATOS.md#estados) |

**No registrados** (escritos por la sync, arrays serializados): `bh_images` (URLs), `bh_features` (nombres), `bh_features_full` (`[{name, category}]`).

**Privados** (`show_in_rest:false`, `auth_callback edit_posts`; nunca en frontend, REST, feeds ni webhook): `bh_priv_address`, `bh_priv_owner_name`, `bh_priv_owner_phone`, `bh_priv_commission`, `bh_priv_onedrive`, `bh_priv_verified_at`, `bh_priv_notes`, `bh_priv_notes_original` (nota de EB del alta), `bh_priv_price_log` (solo la sync, historial de precio).

### Flujo de sincronización (`BH_EB_Sync::run`)
1. Lock por transient `bh_eb_sync_lock` (5 min); presupuesto de 25 s por ejecución.
2. Lista `/properties` paginado (50) con `search[updated_after]` = `bh_eb_last_sync` (incremental) o completo.
3. Por item: `find_post()` (SQL directo con `$wpdb->prepare`) excluye `auto-draft` pero incluye `trash` (papelera), prefiriendo el post fuera de la papelera, para no crear duplicados en blanco si un usuario borra un post por error; si el resultado está en papelera se omite y se registra en el log. Salta si `bh_updated_at` coincide; si no, `GET /properties/{id}` → `map_detail()` → `upsert()` → `assign_terms()` (operación y tipo siempre; zona y amenidades salvo `bh_lock_terms`).
4. Presupuesto agotado → `bh_eb_continue` en 60 s; HTTP 429 → reintento en 5 min.
5. Modo `full` completo → `hide_missing()` archiva las publicadas que ya no existen (`BH_EB_CPT::archive($id, 'sync')`, sin cambiar `post_status`). Si una archivada por la sync (`bh_archived_by = sync`) reaparece en el listado, `unarchive()` restaura `bh_visibility_prev` (def. `publica`); las archivadas a mano se respetan.
6. Guarda `bh_eb_last_sync` (−120 s de traslape) y escribe en el log (`bh_eb_log`, 40 líneas) + resumen de features/ubicaciones sin mapear.

Opciones: `bh_eb_last_sync`, `bh_eb_log`, `bh_eb_api_key`, `bh_terms_seeded`, `bh_visibility_db` (`'2'`: migración única en `init` de la antigua meta `bh_archived = 1` a `bh_visibility = archivada`), `bh_blog_cats_seeded` (tema). Archivos: `uploads/bh-ficha/` (caché de fotos de la ficha PDF; se puede borrar, se regenera al imprimir). Transients: `bh_eb_sync_lock`, `bh_eb_test_out`, `bh_lead_{md5(ip)}`, `bh_lead_wa_{md5(ip)}`, `bh_lead_alert_lock`, `bh_attr_{md5(ip)}`, `bh_leads_notice_{user_id}`, `bh_feed_ads` y `bh_v_{md5(ip|id)}` (tema).
