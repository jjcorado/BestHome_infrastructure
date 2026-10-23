# Contrato de datos de propiedad (independencia de EasyBroker)


Qué debe tener una `propiedad` para que el sitio funcione **sin EasyBroker**. Las altas propias (`bh_source = propio`) se capturan contra este contrato; las de EasyBroker lo cumplen vía la sync. Registro de metas, REST y privacidad: [Campos meta](PLUGIN.md#campos-meta-propiedad); lectura en plantillas: `bh_prop()` ([functions.php](ARQUITECTURA.md#functionsphp)).

### Campos
Obligatorio: **Sí** = el sitio falla o la propiedad no debe publicarse sin él; **Rec.** = recomendado (sin él se pierde una función: mapa, feed, filtros); **No** = opcional.

| Campo / meta | Tipo | `bh_prop()` | Oblig. | Origen actual en EasyBroker | Notas |
|---|---|---|---|---|---|
| `post_title` | string | — | Sí | `title` | Base del slug |
| `post_content` | HTML | — | Rec. | `description` | Descripción de la ficha |
| `post_status` | `draft` \| `publish` | — | Sí | `status` (`published` → `publish`) | Ver [Estados](#estados) |
| `post_date` | datetime | — | Sí | `created_at` (solo en el alta) | Orden por defecto si no hay `bh_sort_date` |
| `post_name` | string | — | Sí | `sanitize_title(title)-{id}` (alta) | Estable: no cambia al editar el título |
| Imagen destacada | adjunto | — | No | — | Va antes de `bh_images` en la ficha PDF |
| `bh_eb_id` | string | `eb_id` | Sí | `public_id` | ID público (ver [IDs](#ids)); en propias guarda el `BH-…` (nombre histórico de la meta) |
| `bh_internal_id` | string | — | No | `internal_id` | Búsqueda por ID en admin y `?q=` |
| `bh_operation` | `sale` \| `rental` | — | Sí | `operations[].type` | Debe coincidir con el término `operacion` |
| Término `operacion` | venta \| renta | `kind` | Sí | `operations[].type` | Ruta del catálogo |
| Término `tipo_propiedad` | casa \| departamento \| terreno | `tipo`, `kind` | Sí | `property_type` | |
| Término `zona` | padre › colonia | — | Rec. | `location.name` → `zone_parents()` | Sin zona no aparece en rutas por zona |
| Términos `amenidad` | lista | — | No | `features` (mapeo) | |
| `bh_price` | number | `price` | Sí | `operations[].amount` (× `lot_size` si es por m²) | Siempre el total; 0 si no se muestra |
| `bh_show_price` | boolean | `show` | Sí | `show_prices` | Falso → “Consultar precio” |
| `bh_currency` | `MXN` \| `USD` | `cur` | Sí | `operations[].currency` | Def. MXN |
| `bh_price_unit` | string | — | No | `operations[].unit` | Solo informativo |
| `bh_price_review` | `1` | — | No | calculado por la sync | Aviso de precio por m² sospechoso |
| `bh_bedrooms`, `bh_bathrooms`, `bh_half_bathrooms`, `bh_parking` | number | `bed`, `bath`, `half`, `park` | Rec. (No en terreno) | `bedrooms`, `bathrooms`, `half_bathrooms`, `parking_spaces` | |
| `bh_construction`, `bh_lot` | number (m²) | `cons`, `lot` | Rec. (`bh_lot` Sí en terreno) | `construction_size`, `lot_size` | |
| `bh_floors` | number | — | No | `floors` / `levels`… | Solo si > 0 |
| `bh_lat`, `bh_lng` | number | `lat`, `lng` | Rec. | `location.latitude/longitude` | Mapa, PDF y JSON-LD |
| `bh_location_name` | string | `loc` | Sí | `location.name` | Texto “Colonia, Municipio, Estado” |
| `bh_images` | array de URL | `images` | Sí (≥ 1) | `property_images[].url` | Sin foto no entra al feed; ver [Fotos](#fotos-pendiente-de-implementar) |
| `bh_features` / `bh_features_full` | array | `feat` | No | `features` | |
| `bh_video_url`, `bh_tour_url` | URL | — | No | `videos`, `virtual_tour` o metabox | |
| `bh_status`, `bh_updated_at` | string | — | Solo EB | `status`, `updated_at` | Control de la sync; vacíos en propias |
| `bh_source` | `easybroker` \| `propio` | — | Sí | fijo `easybroker` | Sin meta = `easybroker` |
| `bh_visibility` | `publica` \| `privada` \| `archivada` | — | Sí | — (admin; la sync solo archiva / desarchiva) | Sin meta = `publica`; ver [Estados](#estados) |
| `bh_visibility_prev` | `publica` \| `privada` | — | No | — | Visibilidad antes de archivar; la sync la restaura al desarchivar |
| `bh_archived_at`, `bh_archived_by` | UTC, `sync` \| `manual` | — | No | — | Solo en archivadas; `sync` = ya no está en EasyBroker |
| `bh_sort_date` | `Y-m-d H:i:s` UTC | — | No | — (“Subir al inicio”) | Ver [inc/catalog.php](ARQUITECTURA.md#inccatalogphp--urls-y-consulta-del-catálogo) |
| `bh_featured`, `bh_luxury`, `bh_luxury_manual`, `bh_lock_terms`, `_bh_badge` | boolean / string | — | No | — (admin; `bh_luxury` auto) | |
| `bh_views` | number | — | No | — (tema) | |
| `bh_priv_address` | string | — | No | — | “Ubicación o modelo” |
| `bh_priv_owner_name` | string | — | No | — | Dueño o constructor |
| `bh_priv_owner_phone` | string | — | No | — | |
| `bh_priv_commission` | string | — | No | — | Comisión pactada de la captación (no la de cada operación) |
| `bh_priv_onedrive` | URL | — | No | — | |
| `bh_priv_verified_at` | `Y-m-d` | — | No | — (admin o `BH_EB_Price_Checks::record()`) | Fecha de verificación del precio |
| `bh_priv_notes` | texto | — | No | `private_description` (solo en el alta, si está vacía) | Notas internas del equipo |
| `bh_priv_notes_original` | texto | — | No | `private_description` (solo en el alta) | Copia de solo lectura de la nota de EB |
| `bh_priv_price_log` | texto | — | No | calculado por la sync | Historial de precio |

Todas las `bh_priv_*` siguen las reglas de privacidad de [Campos meta](PLUGIN.md#campos-meta-propiedad).

### IDs
- **`EB-XX0000`** (los `public_id` existentes): se conservan **para siempre**, también después de cancelar EasyBroker. Están en slugs, feed (`g:id`), URLs compartidas, QR, PDFs y leads; no se renombran.
- **`BH-XX0000`**: propiedades propias. 2 letras + 4 dígitos, aleatorios y únicos (comprobar contra `bh_eb_id` antes de asignar), sin `I` ni `O`. Mismo uso que los `EB-…` (meta `bh_eb_id`, slug `…-bh-xx0000`).
- **`R-XXXXXX`**: código de referencia de leads (`ref_code`, ver [Leads](LEADS.md)). Nunca se usa para propiedades.

### Estados
| Estado | WordPress | Comportamiento |
|---|---|---|
| Borrador | `post_status = draft` | No abre en público |
| Publicada | `publish` + `bh_visibility = publica` | En listados, feed y búsqueda |
| Privada | `publish` + `bh_visibility = privada` | Abre por URL directa con `noindex`; fuera de listados y REST anónima ([inc/catalog.php](ARQUITECTURA.md#inccatalogphp--urls-y-consulta-del-catálogo)) |
| Archivada (vendida o retirada) | `bh_visibility = archivada` (cualquier `post_status`; selector “Visibilidad” en “Control interno” o `hide_missing()`) | Fuera de listados, búsqueda, feed, sitemap y REST anónima (mismo filtro que las privadas, `bh_is_unlisted()`). La ficha abre con `noindex`, aviso `.farch` “Esta propiedad ya no está disponible”, similares antes de la galería, sin formulario de lead ni `#sheet-wa` (botón a `#sheet-wa-gen`, origin `archivada`), JSON-LD `SoldOut`, sin `view_item` y sin PDF para el público (botón, `ficha-print.php`, assets de `assets/ficha/` y `?print=`; `bh_can_print_ficha()`: sí con `edit_posts`) |

Ventas, precio de cierre y comisiones de cada operación viven en el **CRM**, no en el sitio.

### Reglas de la sync
Detalle del flujo: [Flujo de sincronización](PLUGIN.md#flujo-de-sincronización-bh_eb_syncrun).
- **Excluidas**: `bh_source = propio` se omite en el listado, en `upsert()` y en `hide_missing()` (que ya no pasa a borrador: archiva). La sync nunca crea ni modifica una propia.
- **Escribe siempre**: `post_title`, `post_content`, `post_status`; metas `bh_eb_id`, `bh_internal_id`, `bh_operation`, `bh_price`, `bh_show_price`, `bh_currency`, `bh_price_unit`, `bh_bedrooms`, `bh_bathrooms`, `bh_half_bathrooms`, `bh_parking`, `bh_construction`, `bh_lot`, `bh_lat`, `bh_lng`, `bh_location_name`, `bh_status`, `bh_updated_at`, `bh_images`, `bh_features`, `bh_features_full`, `bh_source` (= `easybroker`), `bh_price_review`; términos `operacion` y `tipo_propiedad`.
- **Escribe con condición**: `bh_visibility`, `bh_visibility_prev`, `bh_archived_at`, `bh_archived_by` (archiva en `hide_missing()` guardando la visibilidad previa; al reaparecer desarchiva y la restaura, solo si `bh_archived_by = sync`); `bh_video_url`, `bh_tour_url`, `bh_floors` (solo si EB trae valor); `bh_luxury` (salvo `bh_luxury_manual`); `zona` y `amenidad` (salvo `bh_lock_terms`); `bh_priv_price_log` y `bh_priv_verified_at` (al cambiar el precio); `bh_sort_date` (solo con `bh_bump_on_price_drop` y baja de precio); `bh_priv_notes_original` / `bh_priv_notes` (solo en el alta y si están vacías); `post_name` y `post_date` (solo en el alta).
- **Nunca toca**: `bh_featured`, `bh_luxury_manual`, `bh_lock_terms`, `_bh_badge`, `bh_views`, imagen destacada y el resto de `bh_priv_*` (`address`, `owner_name`, `owner_phone`, `commission`, `onedrive`).
- **`post_date`**: en actualizaciones se reenvía la fecha existente (sin ella `wp_insert_post()` pondría “ahora”); en el alta se usa `created_at` de EB.

### Fotos (pendiente de implementar)
- Destino: `uploads/propiedades/{ID}/` (`{ID}` = `EB-…` o `BH-…`), como **adjuntos de WordPress** del post.
- Original de máximo **2560 px** (lado largo); generar solo los tamaños que usa el sitio (hoy `bh-card` 800×600 y `large`); el resto se desactiva para esta carpeta.
- Las de EasyBroker se migran con su ID `EB-…` **antes** de cancelar EasyBroker; hoy se sirven desde `assets.easybroker.com` (`bh_images`) y solo la ficha PDF tiene copia local ([Impresión / PDF](ARQUITECTURA.md#impresión--pdf-ficha-técnica)).

### Pendiente a futuro
- Banderas de publicación por portal (qué propiedad sale en cada portal / feed).
- Endpoint de inventario para n8n / IA (lectura del contrato sin metas privadas).
