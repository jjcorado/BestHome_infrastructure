# ARQUITECTURA DEL SITIO — Best Home Querétaro

Sitio WordPress (Local by Flywheel) de catálogo inmobiliario. Las propiedades se sincronizan desde la API de EasyBroker hacia un CPT `propiedad`; el tema las presenta con catálogo filtrable, ficha, blog y captura de leads.

| Pieza | Ruta (desde `wp-content/`) | Versión | Repo git |
|---|---|---|---|
| Tema activo | `themes/besthome/` | style.css 0.1.0 · `BH_THEME_VERSION` 0.1.50 | propio |
| Plugin de sincronización | `plugins/besthome-easybroker-sync/` | `BH_EB_VERSION` 0.3.4 | propio |
| MU-plugin | `mu-plugins/local-by-flywheel-live-link-helper.php` | — | (de Local, no tocar) |
| MU-plugin de seguridad: endurecimiento global (XML-RPC, enumeración de usuarios, versión, comentarios apagados, alertas de administrador, SMTP) | `mu-plugins/bh-security.php` → carga `plugins/besthome-easybroker-sync/mu/bh-security.php` | — | en el plugin |

Dependencia: el tema **requiere** el plugin (CPT, taxonomías, metas, endpoint REST de leads y `BH_EB_Sync::zone_parents()`).

Convenciones: prefijo `bh_` (funciones, metas, opciones), `BH_` (constantes), `BH_EB_*` (clases del plugin). Código y comentarios en español. Sin build step: CSS/JS planos, sin jQuery.

---

Este archivo cubre el tema, el CSS y las reseñas de Google. El resto del sitio está en archivos hermanos: [PLUGIN.md](PLUGIN.md) · [LEADS.md](LEADS.md) · [CONTRATO-DATOS.md](CONTRATO-DATOS.md) · [API-CATALOGO.md](API-CATALOGO.md) · [GTM.md](GTM.md).

---

## Tema activo `themes/besthome/`

```
besthome/
├── style.css                 Cabecera del tema (sin estilos)
├── functions.php             Setup, assets, constantes, helpers globales, includes
├── header.php                <head>, header sticky, nav escritorio + menú móvil; abre <main>
├── footer.php                Cierra <main>, footer (logo, redes, legales, ©)
├── front-page.php            Home: hero .bh-hero + buscador, Zonas, Destacadas, banda de contacto, Del blog
├── home.php                  Blog: vista editorial (portada limpia) o “Todos los artículos” (?ver/?categoria/?orden/página > 1)
├── archive.php               Archivos del blog (categoría, etiqueta, fecha, autor) → vista “Todos los artículos”
├── single.php                Artículo (diseño .bhb-*): aside/riel/barra de propiedades de su zona, “Sigue leyendo”
├── single-propiedad.php      Ficha de propiedad (galería, tabs, ficha técnica, leads, similares)
├── archive-propiedad.php     Catálogo: chipbar, panel de filtros, conteo/orden, listado, paginación
├── taxonomy.php              operacion / tipo_propiedad / zona → reutiliza archive-propiedad.php
├── page.php                  Página genérica (.article)
├── page-contacto.php         Formulario de lead + datos de contacto
├── page-quienes-somos.php    Misión + infocards (contenido placeholder)
├── index.php · 404.php       Fallbacks
├── template-parts/
│   ├── property-card.php     Tarjeta del catálogo (carrusel, specs, “Ver detalles”, etiqueta `_bh_badge`)
│   ├── property-mini.php     Tarjeta compacta (home, similares)
│   ├── filters.php           Formulario del panel de filtros (<details> por sección)
│   ├── ficha-print.php       Ficha técnica para PDF (portada, descripción, características, ficha + mapa, galería); solo visible al imprimir
│   ├── post-card.php         Tarjeta de artículo .bhb-card ($args: variant ''|row|row-m, heading, excerpt, minutes)
│   ├── blog-archive.php      Vista “Todos los artículos” (pestañas, orden, grid 6/página, paginación, CTA)
│   └── blog-cta.php          Puente blog → catálogo (.bhb-cta); $args sheet ('wa-gen' por omisión) y origin ('blog_cta'); la portada lo reutiliza como banda de contacto
├── inc/
│   ├── catalog.php           URLs limpias, rewrite rules, estado de filtros, pre_get_posts, chips
│   ├── seo.php               H1/<title> dinámico, canonical, GTM, JSON-LD, dataLayer
│   ├── feed.php              Feed XML /feed/catalogo-ads/ (Meta / Google Ads)
│   ├── admin-qr.php          Propiedades › Generador QR (URL + utm_source/utm_campaign, utm_medium=qr; SVG y PNG)
│   ├── google-rating.php     Reseñas de Google: insignias, línea de la ficha, bloque de portada y de Quiénes somos (ver «Reseñas de Google»)
│   ├── ficha.php             Copias locales de las fotos de EasyBroker para la ficha PDF (uploads/bh-ficha/{id}/, ≤1024 px)
│   ├── customizer.php        Personalizador: Portada (hero, 4 zonas, banda de contacto), Footer y Redes, Blog CTA, WhatsApp, Reseñas de Google, Analítica y GTM, logo oscuro
│   └── related.php           Propiedades similares, posts relacionados, propiedades por zona
└── assets/
    ├── css/variables.css     Tokens (claro/oscuro)
    ├── css/layout.css        Base, header, footer, rejillas
    ├── css/components.css    Componentes
    ├── js/main.js            Todo el JS del frontend (vanilla, IIFE)
    ├── ficha/                Ficha PDF (solo en is_singular('propiedad'), versión = filemtime): qr.js (BHQR), ficha-print.js (BHFicha), ficha-print.css
    └── img/logo.png
```

### functions.php
- **Constantes**: `BH_THEME_VERSION`, `BH_WHATSAPP`, `BH_PHONE_LABEL`, `BH_PHONE_TEL`, `BH_EMAIL`.
- **Theme supports**: `title-tag`, `post-thumbnails`, `html5`; tamaño de imagen `bh-card` (800×600 crop).
- **Assets** (orden obligatorio): Google Font Inter → `variables.css` → `layout.css` → `components.css`. Leaflet 1.9.4 (cdnjs) solo en `is_singular('propiedad')`. `main.js` con `wp_localize_script('BH', { lead, attr, whatsapp, view, noBrand })` (`whatsapp` ofuscado con `bh_obf()`; `view` = URL del contador solo en `post`/`propiedad` para visitantes no logueados; `noBrand` = `"1"` solo en la ficha y con `edit_posts`). Turnstile solo en ficha y `/contacto/` si existe `BH_TURNSTILE_SITE_KEY`.
- `wp_head` prioridad 1: script inline que aplica `data-theme` desde `localStorage['bh-theme']` o, si no hay elección guardada, desde `prefers-color-scheme` del sistema (evita parpadeo).
- Contador `bh_views` (`post`/`propiedad`, solo visitantes no logueados): ya **no** se ejecuta sincrónicamente en `wp_head`. `main.js` envía un `navigator.sendBeacon` asíncrono (una vez por sesión) a `POST /wp-json/besthome/v1/view/{id}`, protegido por un límite de 1 hora por IP+post mediante transients (`bh_v_{md5(ip|id)}`); el incremento es atómico (`UPDATE … meta_value + 1`).
- **Helpers globales**:
  - `bh_icon($name,$size)` → SVG inline `.i` (phone, mail, menu, x, moon, sun, search, chev, left, right, pin, bed, bath, ruler, land, car, share, file, link, chat, sliders, fb, ig, yt, in).
  - `bh_prop($id)` → array normalizado de la propiedad (`id, eb_id, price, show, cur, bed, bath, half, park, cons, lot, lat, lng, loc, images, feat, kind, tipo`). **Usar siempre esto en plantillas** en vez de leer metas sueltas.
  - Formato: `bh_money`, `bh_num`, `bh_m2`, `bh_baths`, `bh_price_html` (“Consultar precio” si no se muestra), `bh_short_loc`.
  - Navegación: `bh_zones_menu()` (Juriquilla, Zibatá, Querétaro Moderno), `bh_blog_url()`.
  - Contacto: `bh_obf($s)` (base64 del texto invertido) para teléfono, correo y WhatsApp; en el HTML van como `data-c` (href) y `data-l` (texto, con respaldo “Ver teléfono” / “Ver correo” sin JS) y `main.js` los reconstruye.
  - Blog: `bh_post_bg`, `bh_post_media` (miniatura o degradado, home/ficha); `bh_blog_sections()` (zonas, proceso-y-consejos, queretaro-y-mercado → nombre y descripción; se crean una vez en `admin_init`, opción `bh_blog_cats_seeded`); `bh_blog_is_editorial($q)`, `bh_blog_order()` (`recientes|leidos|antiguos`), `bh_blog_list_url($cat,$orden)` (`?ver=todos` o `?categoria=`); `bh_reading_time($id)` (palabras / 200, mín. 1); `bh_post_cat($id)`, `bh_post_zone($id)` (prefiere la colonia).
  - Más leídos: cualquier `WP_Query` con `'bh_order_views' => true` ordena por `bh_views` numérico DESC y luego fecha (filtro `posts_clauses` con LEFT JOIN: incluye artículos sin visitas). No usar con `get_posts()` (suprime filtros).
  - `init` (prioridad 20): `register_taxonomy_for_object_type('zona','post')` como respaldo del registro del plugin.
  - `bh_youtube_embed($url)` → URL youtube-nocookie.
  - `bh_turnstile_widget()`.
  - `bh_wa_sheet(['gen' => true])` → `#sheet-wa-gen` (footer.php, todas las páginas): selector Venta/Renta y mensaje `bh_wa_generic_msg()` (Personalizador › WhatsApp; def. “Hola, quiero información sobre propiedades en {operacion} en Querétaro”). `{operacion}` se imprime como “venta” y, si está, `data-wa-tpl` guarda la plantilla para `main.js`.
  - `bh_wa_options($op)` + `bh_wa_sheet($c)` → `#sheet-wa`, solo en la ficha (presupuesto opcional: “Presupuesto acorde a la propiedad” por defecto; venta: rangos de 4, 6 y 9 MDP; renta: menos de $15,000, $15,000–$25,000, $25,000–$40,000, más de $40,000; nunca abre wa.me directo).
  - `bh_lead_message($title,$eb_id,$price,$cur)` → “Me interesa la propiedad [Título] ([ID]) por [Precio]”, mismo texto en el formulario de la ficha y en WhatsApp.
  - `bh_utm_fields()` → ocultos `utm_source`, `utm_medium`, `utm_campaign` (formularios de la ficha, contacto y WhatsApp).

### header.php / footer.php
- **header.php**: construye `$venta` (Casas/Departamentos/Terrenos × zonas del menú + “Casas de lujo” → `segmento=lujo`) usado por el submenú de escritorio (`.dd > .ddp > .ddc > .ddcol`; títulos de grupo como `div.bh-nav-title`, sin `<h*>` antes del `<h1>`) y por el menú móvil (`#mobile-menu .mm-sub`). Enlaces: Venta, Renta, Quiénes somos, Contacto, Blog. Contacto tel/mail (`.dcontact`), botón de tema (`[data-theme-toggle]`), acciones móviles (`.mob-actions`, `[data-menu-toggle]`). Abre `<main id="main">`.
- **footer.php**: `.site-footer` con `.frow` (logo invertido + `.socials`, enlaces `#` pendientes), `.flinks` (privacidad, términos `#`), `.copy`.

### Plantillas de página
| Plantilla | Contenido clave |
|---|---|
| `front-page.php` | `.bh-hero`: `<picture>` con imagen de escritorio y de móvil (`bh_hero_img_desktop` / `bh_hero_img_mobile`, IDs de adjunto; sin móvil usa la de escritorio, sin ninguna degradado de `--primary`), sombreado y `.bh-card` (eyebrow, H1, texto, línea tricolor 60/15/25 a todo el ancho y buscador `.bh-search` GET al archivo); textos `bh_hero_{eyebrow,title,sub}` con `bh_hero_defaults()` (el H1 nunca queda vacío). `.zgrid/.ztile` (4 zonas), destacadas (`bh_featured=1`, fallback últimas 6) en `.hscroll` con `property-mini`, banda de contacto (`blog-cta.php` con `bh_home_cta_{title,text,wa_label,visit_label}`, enlace a `/contacto/`, `#sheet-wa-gen`, origen `home_banner`), 3 posts. |
| `archive-propiedad.php` | H1 oculto (`bh_catalog_heading()`), `.chipbar` (botón Filtros + chips activos), `.catlayout` = `aside#filters.filters` + `.catmain` (`.countrow` con select `orden`, `.plist` de `property-card`, `.pagination`, `.empty`). 12 por página. |
| `single-propiedad.php` | `.fwrap` = `.fleft` + `aside.fform`. Tabs `.ftabs` (Fotos/Video/Recorrido 3D/Mapa) → `.gallery` / `.mediabox` (iframes lazy `data-src`) / `.pmap`. Visor `#lbox`. Barra sticky `.fsub` (`.fnav` anclas + `.sharemenu`, `.fpricerow`). `.fcontent`: `.ftitle1`, `.stats`, `.fdesc/.desc` + “Leer más”, `.fchar` (`.abadges` enlazan al catálogo, `.clist`, `.ftable`). Similares + artículos (`.fextra .simgrid`). Móvil: `.bbar` + `.sheet-info` + `#sheet-wa`. Formulario de lead con mensaje prellenado. |
| `home.php` | **Editorial** (sin `?ver`, `?categoria`, `?orden` y en página 1; el loop principal trae solo 1 post = hero): `.bhb-hero` (`.bhb-feature` + `aside.bhb-trending` “Lo más leído”, 3 por `bh_views` con fallback a fecha), una `.bhb-section` con `.bhb-rail` de 3 posts por cada categoría de `bh_blog_sections()` que exista y tenga posts (“Ver todas” → `?categoria=`), botón “Ver todos los artículos” (`?ver=todos`) y `.bhb-cta`. **Todos los artículos**: `template-parts/blog-archive.php`. |
| `archive.php` | Categoría / etiqueta / fecha / autor → `template-parts/blog-archive.php`: `.bhb-crumbs`, `.bhb-page-title`, `.bhb-filterbar` (`.bhb-tabs` → `?categoria=`, `form.bhb-sort` con `select[data-autosubmit]` `orden`), `.bhb-grid` de `post-card`, `.bhb-pagination` (6 por página), `.bhb-cta`. |
| `single.php` | `.bhb-art-head` (migas, pills categoría + zona, título, extracto manual como entradilla, portada, “Actualizado el … · N min de lectura”), `.bhb-art-layout` = `article.bhb-art-main` (`.bhb-prose` con `the_content()`, etiquetas, `details.bhb-share`: copiar, WhatsApp, Facebook, LinkedIn, X, correo; sin Instagram) + `aside.bhb-zonebox`. Con término de `zona` (`bh_post_zone`): una consulta de 3 propiedades (`bh_zone_properties`) → 2 filas `.bhb-row` en el aside (escritorio), riel `.bhb-rail` con `property-card` y `.bhb-zonebar` sticky (móvil); sin zona no se imprime ninguno y el layout pasa a `.bhb-art-layout--solo`. “Sigue leyendo”: 3 `post-card` `row-m` de `bh_related_posts`. |
| `page-contacto.php` / `page-quienes-somos.php` | Contacto: `.twocol` con `.formcard` (`form[data-lead]`) y `.infocard.contact-card` (texto, `.ccontact` tel/correo ofuscados y botón `.btn-wa` → `#sheet-wa-gen`, origen `contacto`). Quiénes somos: `.twocol` con `.infocard.about-card` (H1 + texto) y la foto `.about-photo` (imagen destacada, o `.ph`) a la derecha (en móvil, debajo), y `.bgrid` con hasta 3 `.infocard`; textos desde el Personalizador (`bh_about_*`, `bh_about_defaults()`). Se resuelven por slug (`contacto`, `quienes-somos`). |

### inc/catalog.php — URLs y consulta del catálogo
- **Rutas limpias** (rewrite `top`, query vars `bh_op`, `bh_tp`, `bh_z1`, `bh_z2`):
  `/venta/` · `/venta/casas/` · `/venta/casas/juriquilla/` · `/venta/casas/juriquilla/cumbres-del-lago/` · `/venta/propiedades/juriquilla/` (+ `/page/N/`).
- **Operación obligatoria**: siempre `venta` o `renta` (sin “Todas”); si la ruta o `?op=` no la traen, `bh_current_filters()` y `pre_get_posts` asumen `venta` (`/operacion/{slug}/` usa su término). No genera chip.
- **Parámetros GET**: `op`, `tp[]`, `z[]`, `amenidad[]` (alias `car[]`), `segmento=lujo`, `q`, `pmin/pmax`, `rec`, `ban`, `cmin/cmax`, `tmin/tmax`, `orden` (`recientes|precio_asc|precio_desc`), `region=todas`.
- Funciones: `bh_current_filters()` (estado = ruta + GET, memoizado por petición), `bh_build_catalog_url($op,$tp,$zona,$args)`, `bh_catalog_url($args)`, `bh_state_url($s)`, `bh_active_filters()` (chips `[texto, url_sin_filtro, sección]`, incluye Precio, Construcción y Terreno), `bh_zone_path()`, `bh_zone_tax_clause()`.
- `template_redirect`: archivo con GET → 301 a la ruta limpia.
- `pre_get_posts`: arma `tax_query` (AND; amenidades con `operator AND`) y `meta_query` numérico; **excluye San Miguel de Allende** por defecto salvo que se filtre por zona o `region`. Blog (home, categoría, etiqueta, fecha, autor): sin sticky; portada editorial → 1 post y `no_found_rows`; si no, 6 por página, `categoria` (solo home), `orden=antiguos` (ASC) / `orden=leidos` (`bh_order_views`). Catálogo en `orden=recientes` (predeterminado): `bh_order_sort` → `posts_clauses` con LEFT JOIN de `bh_sort_date` (UTC, “Subir al inicio”) y `COALESCE(bh_sort_date, post_date_gmt) DESC`.

### inc/seo.php
Meta description (`wp_head` prioridad 1, `bh_meta_description()` recortada a ~160 caracteres con `bh_meta_clean()`): portada (lema del sitio o texto genérico), inicio del blog, singular (extracto o 25 palabras; sin texto → título + genérico), catálogo (`term_description()` o `bh_catalog_heading()`), categorías/etiquetas. JSON-LD `Blog` en el inicio del blog. `bh_is_catalog()`, `bh_catalog_heading()` (p. ej. “Casas de lujo en venta en Zibatá, Querétaro Moderno con Roof garden”) usado en H1 y `document_title_parts`; canonical del catálogo; GTM (`bh_gtm_id()`: constante `BH_GTM_ID` o Personalizador › Analítica y GTM `bh_gtm_id`, validado `GTM-XXXX`): `<script>` en `wp_head` prioridad 1 y `<noscript>` en `wp_body_open` (header.php lo llama tras `<body>`); JSON-LD `RealEstateListing` + `dataLayer` `view_item` en ficha; `BreadcrumbList` + `ItemList` en catálogo. `bh_zone_parts($id)` → `[padre, hijo]`.

### inc/feed.php
`/feed/catalogo-ads/` (RSS 2.0 + namespace `g:`), cache en transient `bh_feed_ads` (30 min, se invalida en `save_post_propiedad` y al cambiar `bh_visibility`, `bh_price`, `bh_show_price`, `bh_currency`, `bh_images` o `bh_luxury`). Omite propiedades sin foto o sin precio. `custom_label_0=lujo`.

### inc/related.php
`bh_similar_properties()` (misma operación y tipo, colonia → zona padre, precio ±35 %), `bh_related_posts()` (misma zona o su padre, misma categoría salvo “Sin categoría”, u operación + tipo; se rellena con recientes), `bh_zone_properties()` (destacadas primero).

### assets/js/main.js (hooks por atributo `data-*`)
| Selector | Comportamiento |
|---|---|
| `[data-theme-toggle]` | Alterna `data-theme` en `<html>` y lo guarda en `localStorage`. |
| `[data-menu-toggle]` / `#mobile-menu` | Menú móvil (`.is-open`). |
| `[data-filter-toggle]`, `#filters`, `[data-filter-close]`, `[data-open-filter]` | Panel de filtros móvil (fixed bajo `.chipbar`), abrir sección desde chip; recuerda secciones abiertas y scroll. |
| `.range` | Min/máx se envían al salir del par o con Enter. |
| `select[data-autosubmit]` | Envía el formulario al cambiar. |
| `.prop-card[data-images]` | Carrusel + swipe en tarjeta. |
| `.gallery`, `.ftabs [data-tab]`, `[data-panel]` | Galería de la ficha y pestañas (carga diferida de iframes/mapa). |
| `#lbox` | Visor de fotos (flechas, puntos, Esc, ←/→). |
| `[data-map]` | Leaflet + OpenStreetMap (CARTO exige llave; OSM bloquea dominios temporales/IP, no usar nip.io en pruebas), círculo de 500 m sin pin. |
| `[data-readmore]`, `[data-share-toggle]`, `[data-bhb-copy]`, `[data-scroll]` | Ficha: leer más, compartir, copiar, anclas con offset. El PDF (`[data-bh-print]`) está en `assets/ficha/ficha-print.js` (ver “Impresión / PDF”). |
| `details.bhb-share`, `[data-bhb-copy]` | Artículo: copiar enlace (“Enlace copiado”), cerrar al hacer clic fuera o con Esc. |
| `[data-open-sheet]` / `[data-close-sheet]` | Hojas `#sheet-{info,wa,wa-gen}`. El blog ya no imprime `#sheet-wa`: sus botones (`.bhb-cta`, `.bhb-zonebox`) abren `#sheet-wa-gen`. Nunca hay `wa.me` en el HTML. |
| `[data-wa-origin]` | Origen del botón de WhatsApp: `ficha_form`, `ficha_bar`, `blog_cta`, `blog_zona`, `header`, `home_banner`, `archivada`. Va en `modal_whatsapp_open` (`origin` + `property_id` = eb_id aparte si hay propiedad) y como `origin` en el lead. |
| `form[data-wa-tpl]` | Reemplaza `{operacion}` por “venta”/“renta” al abrir y al cambiar el selector, salvo que el usuario haya editado el texto. |
| `form[data-lead]` | POST a `BH.lead` (REST del plugin) con UTMs de sesión. |
| dataLayer | `window.dataLayer` se inicializa al inicio de `main.js`. Conversión unificada `lead_generado` con `lead_type: 'contacto'` (respuesta OK del formulario) o `'whatsapp'` (al abrir wa.me), más `property_*`, `qualification_option` (presupuesto elegido en WhatsApp) y `utm_source` / `utm_campaign`, y `lead_id` = `ref_code` (formulario: el que devuelve el servidor; WhatsApp: el que genera el navegador y el servidor conserva), el mismo valor de `wp_bh_leads.ref_code`. **event_id para Meta Pixel y Conversions API = ref_code.** Es el **único** evento de conversión (se retiraron `generate_lead_form`, `generate_lead_whatsapp` y `click_phone_header`). Aparte, `view_item` en la ficha (inc/seo.php) para remarketing dinámico de catálogo. |
| `form[data-wa]` | Paso intermedio → abre `wa.me` con el mensaje, el presupuesto (si se eligió; en `#sheet-wa-gen` no hay línea de operación), “Origen: [source] - Campaña: [campaign]” si hay UTMs y “Ref: R-XXXXXX”. Antes de abrir, `sendBeacon` a `BH.lead` con `type=whatsapp`, `ref_code`, `qualification_option`, `origin` y atribución (status `local`). |
| `[data-bhb-copy]` | Copiar enlace (blog y ficha): “Enlace copiado” y cierra su menú; sin `navigator.clipboard` (HTTP) usa un `<textarea>` temporal. |
| UTMs | `sessionStorage['bh-utm']` (último contacto; incluye `gclid`, `gbraid`, `wbraid`, `fbclid` y `fbclid_at`, ms de la captura de un fbclid nuevo, que viaja en cada lead); `fillUtm()` llena los ocultos al cargar y al enviar. |
| Atribución | Sin cookie `bh_attr` y llegando con utm/click ids o desde otro dominio: `POST BH.attr` (respaldo en `localStorage['bh_attr_js']`). `attribution()` agrega `attribution_first`, `attribution_last`, click ids, `_fbc`/`_fbp`/`_ga` y `landing_page` a cada lead. |
| `[data-c]` / `[data-l]` | Decodifica contacto ofuscado (`bh_obf`): `href` y texto visible. |
| `BH.view` | `navigator.sendBeacon` al contador de visitas (una vez por sesión, `sessionStorage`). |

---

---

## CSS global y componentes reutilizables

Orden de carga: `variables.css` → `layout.css` → `components.css`. Regla del proyecto: **cada selector vive en un solo archivo**, con sus variantes móvil/escritorio al final de ese archivo. Breakpoint único: móvil ≤ 1023 px, escritorio ≥ 1024 px (también `hover:hover` / `hover:none`). El CSS aplica el modo oscuro vía `:root[data-theme="dark"]` (sin `@media (prefers-color-scheme)`); el valor inicial de `data-theme` lo fija el script de `wp_head` a partir de `localStorage['bh-theme']` o, si no hay elección guardada, de `prefers-color-scheme`.

### Tokens (`variables.css`)
- **Temáticos** (cambian en oscuro): `--primary` (#283890), `--accent` (#bf1e2e), `--accent-hover`, `--text`, `--canvas`, `--canvas-a`, `--surface`, `--pill`, `--border`, `--stripe`, `--footer`, `--shadow`, `--ph-ratio` (16/10).
- **Fijos**: `--brand-navy`, `--white`, `--on-accent`, `--media-bg`, `--footer-*`, `--glass`, `--dot`, `--hover-tint`, `--overlay(-strong)`, `--ztile-gradient`, `--ph-gradient`, `--shadow-{btn,bar,form,popover,menu,panel,hover}`.
- **Escala**: `--radius-xs|sm|(8px)|lg|pill`; `--container` 1200, `--container-narrow` 960, `--container-text` 760; `--gutter` 20px, `--gutter-sm` 16px; `--header-h` 76px → 104px en escritorio; `--blog-ratio`.

### Utilidades base (`layout.css`)
`.container`, `.section`, `.section-head`, `.screen-reader-text`, `.i` (SVG de `bh_icon`), `[hidden]`, `.icon-btn`, `.desk-only` (dentro de `.site-header`).

### Componentes reutilizables (`components.css`)
| Componente | Clases | Dónde se usa |
|---|---|---|
| Botón | `.btn` (rojo) · `.btn-blue` · `.btn-sm` · `.btn-block` · `.btn-wa` | todo el sitio |
| Enlace subrayado | `.link-u` | “Ver todas”, limpiar filtros |
| Tarjeta base | `.card` (+ hover elevado en escritorio) | propiedades y posts |
| Tarjeta de propiedad | `.prop-card` › `.card-link`, `.pimg`, `.badge`, `.arrow.prev/.next`, `.dots`, `.pinfo`, `.ptitle`, `.pprice`, `.ploc`, `.more`, `.specrow`, `.specs`, `.spec`, `.kind` | `template-parts/property-card.php` |
| Mini tarjeta | `.mini-card` › `.pimg`, `.pbody` (+ `.pprice`, `.ptitle`, `.kind`, `.tag`) | `property-mini.php`, posts en home/ficha |
| Carrusel | `.arrow`, `.dots i.on` | tarjeta, galería, visor |
| Formularios | `.field`, `.lbl`, `.formcard` › `.t`, `.formmsg`, `.hp` (honeypot), `.cf-turnstile` | contacto, ficha, hojas |
| Hoja modal | `.sheet(.is-open)` › `.in`, `.hd`; `.sheet-info` | ficha (bottom sheet en móvil, modal centrado en escritorio) |
| Chips | `.chip`, `.chip.outline`, `.chip-f` › `.chip-l`, `.chip-x`; `.achips/.achip`; `.abadges/.abadge` | catálogo, ficha |
| Filtros | `.filters(.is-open)`, `.fpanel`, `.ftitle`, `.fdet` (details), `.fgrp`, `.fbody`, `.fopt(.sub/.zopt)`, `.pills`, `.range` | `filters.php` |
| Listado | `.countrow`, `.pagination(.current)`, `.empty` | catálogo, blog |
| Ficha | `.ftabs`, `.gallery .slide`, `.lbox`, `.mediabox`, `.pmap`, `.fsub`, `.fnav`, `.navl`, `.sharemenu`, `.fpricerow`, `.fcontent`, `.ftitle1`, `.fid`, `.stats/.stat`, `.desc(.open)`, `.readmore`, `.clist`, `.ftable .row`, `.fform`, `.bbar`, `.fextra` | `single-propiedad.php` |
| Home | `.bh-hero` › `__media`, `__shade`, `__inner`; `.bh-card` › `__eyebrow`, `__title`, `__text`, `__line`; `.bh-search` › `__row`, `__lbl(-t)`, `__field`, `__q`; `.ztile` | `front-page.php` |
| Blog (`.bhb-*`) | `.bhb-link`, `.bhb-btn(--ghost)`, `.bhb-pill`, `.bhb-meta`, `.bhb-date`, `.bhb-divider`, `.bhb-media`, `.bhb-card(--row/--row-m)` › `__title`, `__excerpt`, `.bhb-feature`, `.bhb-cta`, `.bhb-tabs/.bhb-tab`, `.bhb-sort`, `.bhb-pagination/.bhb-page(--gap)`, `.bhb-empty`, `.bhb-share` › `__menu`, `__item`, `.bhb-zonebox`, `.bhb-row`, `.bhb-props-cta`, `.bhb-zonebar`; `.bhb-rail .prop-card` (tarjeta nativa siempre vertical en el riel) | `home.php`, `archive.php`, `single.php`, `post-card.php`, `blog-cta.php` |
| Texto | `.tag`, `.article .entry` | home, ficha, `page.php` |
| Bloques de contenido | `.infocard`, `.ph` (placeholder de imagen) | páginas |

### Layout / rejillas (`layout.css`)
Header: `.site-header`, `.site-logo`, `.dnav`, `.navi`, `.dd/.ddp/.ddc/.ddcol/.ddsep`, `.dcontact`, `.mob-actions`, `.theme-btn`, `.mobile-menu(.is-open)/.mm-sub/.mm-h`. Footer: `.site-footer`, `.frow`, `.footer-logo`, `.socials`, `.flinks`, `.copy`. Rejillas: `.zgrid` (2 → 4 col), `.hscroll`/`.simgrid` (scroll horizontal → 3 col), `.plist`, `.catalog`, `.catlayout` (256px + 1fr), `.fwrap` (65/35), `.bgrid` (1 → 3 col), `.twocol`, `.article`, `.page-wrap`.
Blog (`.bhb-*`): base `.bhb` (reset de h1–h3, p y listas), `.bhb-wrap`, `.bhb-section(--surface/--canvas)` › `__head`, `__title`, `__desc`, `.bhb-rail` (riel → 3 col), `.bhb-grid` (1 → 3 col), `.bhb-all`, `.bhb-hero(__grid)`, `.bhb-trending`, `.bhb-crumbs`, `.bhb-page-title`, `.bhb-filterbar`, `.bhb-art-head(__grid)`, `.bhb-tags`, `.bhb-art-title/-lead/-meta`, `.bhb-cover`, `.bhb-art-layout(--solo)` (1fr + 340px), `.bhb-art-main`, `.bhb-art-foot`, `.bhb-prose` (contenido del editor + `.bhb-summary`, `.bhb-callout`).

### Impresión / PDF (ficha técnica)
`template-parts/ficha-print.php` (en el loop de `single-propiedad.php`) imprime `#bh-ficha.print-only`, oculto en pantalla. Datos de `bh_prop()` (metas `bh_*`), `bh_operation`, `bh_zone_parts()`, amenidades (etiquetas) y `bh_features` (lista); fotos: imagen destacada (`large`) primero y después `bh_images` de EasyBroker **servidas desde el propio dominio** (`inc/ficha.php`: `GET /wp-json/besthome/v1/ficha-foto/{id}/{n}` descarga la foto una vez, la reduce a ≤1024 px, JPEG calidad 82, en `uploads/bh-ficha/{id}/{md5}.jpg` y redirige; solo host `assets.easybroker.com` y propiedades publicadas; `before_delete_post` borra la carpeta al eliminar la propiedad). Motivo: Chrome solo inserta tal cual (DCTDecode) los JPEG del mismo origen; los de otro dominio sin CORS los recodifica sin pérdida (~1 MB por foto): la ficha pasó de 16 MB a 1.8 MB; mapa: mosaicos de OpenStreetMap con círculo de 500 m sin pin (`bh_lat` / `bh_lng`). Teléfono y correo van ofuscados (`data-c` / `data-l`). `assets/ficha/ficha-print.js` mueve la ficha a `<body>`, genera el QR (`qr.js`), carga las fotos solo al imprimir (cola de 3 en paralelo) y expone `BHFicha.print('brand'|'neutral'|'nobrand')`; botón público `[data-bh-print="brand"]` en `.sharemenu`; auto-impresión con `?print=brand|nobrand` (enlaces de wp-admin, `load` + 500 ms). El modo sin marca (`body.bh-print-neutral`) solo se aplica si `BH.noBrand` (usuario con `edit_posts`); para el público imprime con marca. Todo el diseño de impresión vive en `ficha-print.css` (Carta, `@media print`); el tema ya no tiene otro bloque `@media print`.

### Estados por clase (los alterna `main.js`)
`.is-open` (menú, filtros, sheet, sharemenu), `.on` (tabs, slides, dots, navl), `.open` (descripción), `.arrows-on` (táctil).

---

## Reseñas de Google
Calificación y reseñas de Google Business mostradas en el sitio. Verificado contra el código (plugin 0.3.4, tema 0.1.50).

**Plugin** `includes/google-places.php` (clase `BH_EB_Google_Places`, iniciada en el bootstrap):
- Constantes en `wp-config.php`: `BH_GOOGLE_PLACES_KEY` (llave de API restringida por IP del servidor y solo a «Places API (New)», **no** por HTTP referrer), `BH_GOOGLE_PLACE_ID` y, opcional, `BH_GOOGLE_MAX_STALE_DAYS` (def. 28; los términos de Google limitan el caché a ~30 días). Sin las dos primeras el módulo queda desactivado (`enabled()`).
- Cron `bh_google_sync_event` diario (`init` → `schedule()`, primera ejecución a +5 min; se desprograma si el módulo no está habilitado). Candado: transient `bh_google_sync_lock` (2 min).
- Llamada: `GET https://places.googleapis.com/v1/places/{place_id}?languageCode=es` con cabeceras `X-Goog-Api-Key` y `X-Goog-FieldMask: id,rating,userRatingCount,reviews`, timeout 10 s. Solo se guarda si responde 200 con `rating` numérico y `userRatingCount` ≥ 1; si falla, **no** se pisa el dato anterior.
- Opciones (sin autoload): `bh_google_reviews` = `place_id`, `rating` (1 decimal), `count`, `reviews[]` (`autor`, `url`, `maps`, `foto`, `rating`, `texto`, `cuando`, `fecha`; todo sanitizado) y `fetched_at`; `bh_google_status` = `ok`, `time`, `msg` (resultado de la última ejecución). Google devuelve como máximo 5 reseñas por llamada (las elige Google).
- Lector `bh_google_rating()`: devuelve `null` si no hay datos o tienen más de `BH_GOOGLE_MAX_STALE_DAYS` días (no se muestran datos caducados).
- Sync manual: `bh_google_sync_url()` → `admin-post.php?action=bh_google_sync` (`manage_options` + nonce `bh_google_sync`), o `wp eval 'print_r( BH_EB_Google_Places::sync() );'`.

**Tema** `inc/google-rating.php` (funciones `bh_gr_*`; HTML impreso desde PHP, sin widgets ni scripts de terceros; sprite SVG de estrella / «G» / flecha impreso una vez en `wp_footer`). Nunca se edita el texto de una reseña ni la calificación.
- `bh_gr_data()` envuelve `bh_google_rating()` y añade `url` (`https://search.google.com/local/reviews?placeid=…`).
- Reglas de visualización: `bh_gr_badge_ok()` (calificación ≥ `bh_gr_min('rating')` **y** total ≥ `bh_gr_min('total')`), `bh_gr_cards()` (reseñas con texto y ≥ `bh_gr_min('stars')` estrellas, las más recientes primero, tope por lugar) y `bh_gr_cards_ok()` (insignia válida y al menos `bh_gr_min('cards')` reseñas). Con `bh_gr_preview` activo se ignoran los mínimos (solo para revisar diseño).
- Dónde se imprime (cada lugar con interruptor propio, apagado por omisión):

| Lugar (`bh_gr_show_*`) | Función | Dónde se llama | Comportamiento |
|---|---|---|---|
| `hero` | `bh_render_gr_hero()` → `bh_gr_badge_html('hero')` | `front-page.php` | Insignia bajo el buscador |
| `home` | `bh_render_gr_home()` | `front-page.php` | Fila «Opiniones + Agenda tu visita» entre Destacadas y Del blog (`#resenas`). Agenda siempre; carrusel de hasta 6 reseñas (`data-bh-gcard`, JS en `main.js`) si hay reseñas suficientes; con calificación válida pero pocas reseñas, solo el resumen; sin calificación válida, nada |
| `about` | `bh_render_gr_about()` | `page-quienes-somos.php` | Bloque `.bh-greviews` con hasta 3 reseñas; no se imprime si no hay reseñas suficientes |
| `ficha` | `bh_gr_ficha_html()` | `single-propiedad.php` (dentro del formulario, antes del aviso de privacidad) | Línea tricolor + «Te atiende…» + calificación |
| `contacto` | `bh_gr_badge_html('contacto')` | `page-contacto.php` | Insignia junto al título del formulario |
| `footer` | `bh_gr_badge_html('footer')` | `footer.php` | Insignia opcional |

**Personalizador › Reseñas de Google** (`inc/customizer.php`, sección `bh_google_section`, prioridad 35):
- Interruptores `bh_gr_show_{hero,home,about,ficha,contacto,footer}` y `bh_gr_preview`.
- Mínimos (`bh_gr_min_*`, con su rango): `rating` 4 (1–5, paso 0,1), `total` 3 (1–1000), `cards` 3 (1–6), `stars` 4 (1–5).
- Textos: `bh_gr_fallback_text` (insignia sin calificación suficiente), `bh_gr_agenda_title` / `bh_gr_agenda_text` (tarjeta «Agenda tu visita»), `bh_gr_about_eyebrow` / `bh_gr_about_title`, `bh_gr_ficha_agency` (def. «Te atiende Best Home Querétaro»).

CSS en `components.css`: `.bh-gbadge*`, `.bh-gline*`, `.bh-gcard*`, `.bh-agenda*`, `.bh-greviews*`, `.bh-review*`.

---

## Notas para futuros cambios
- Cambios de CSS: no se hacen commit hasta que el usuario confirme la revisión visual.
- Tras modificar rewrites (`inc/catalog.php`) o el CPT: guardar Ajustes › Enlaces permanentes o reactivar el tema/plugin.
- Al subir assets, incrementar `BH_THEME_VERSION` (cache busting).
- Blog: para que `home.php` se use hay que tener una página asignada como “Página de entradas” en Ajustes › Lectura (`bh_blog_url()` cae a `/blog/` si no existe). La zona de un artículo se asigna en el editor (taxonomía `zona`).
- Personalizador (`inc/customizer.php`): **Footer y Redes** (`bh_social_{facebook,instagram,youtube,tiktok}`; vacío = sin ícono, sin ninguno no se imprime `.socials`), **Identidad del sitio** (logo nativo `custom_logo` + `bh_logo_dark`; el header imprime `.logo-light` y, si existe, `.logo-dark` con `.site-logo.has-dark`; sin logo oscuro el normal va sobre placa blanca), **Blog CTA** (`bh_blog_cta_title`, `bh_blog_cta_text`, usados por `template-parts/blog-cta.php`; predeterminados en `bh_blog_cta_defaults()`), **Portada** (imágenes y textos del hero, 4 zonas, banda de contacto `bh_home_cta_*`), **WhatsApp** (`bh_wa_generic_msg`, admite `{operacion}`; un mensaje ya guardado sin el marcador se respeta tal cual).
- `page-contacto.php`: el texto sale de `the_content()`; con el editor vacío muestra el texto de respaldo. `page-quienes-somos.php`: textos en Personalizador › Quiénes somos (`bh_about_title` H1, `bh_about_text`, `bh_about_card_{1..3}_title/_text`; tarjeta sin título ni texto = oculta); no usa el contenido del editor; la foto es la imagen destacada. El footer ya no tiene “Términos y condiciones”.
- WP-Cron: en producción reemplazar por cron real del servidor (ver README del plugin).

Open Graph / Twitter: `inc/seo.php` (`bh_og_image()` + hook `wp_head` prioridad 2).
