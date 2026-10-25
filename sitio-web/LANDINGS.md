# Landings de campaña

Páginas de WordPress sin header ni footer, con HTML y CSS propios, para dirigir tráfico de campañas. Implementado en el tema (`inc/landing.php`, `template-landing.php`).

## Crear una landing
1. Páginas › Añadir nueva. El slug es el **nombre de la landing** (`juriquilla-casas`): va en el origen del lead y en el dataLayer.
2. Atributos de página › Plantilla **Landing**.
3. Metabox **Landing: código**: pega el HTML y el CSS. Opciones: indexable (por defecto `noindex` y fuera del sitemap), pie con Aviso de privacidad (por defecto sí) y modo claro forzado (por defecto sí).
4. Solo usuarios con `unfiltered_html` (administradores) pueden guardar el código.

## Reglas para quien diseñe el código (dárselas a Claude Design tal cual)
**HTML**
- Una sola `<section>`/`<div>` raíz por bloque; sin `<html>`, `<head>`, `<body>`, header ni footer. Un único `<h1>`.
- Clases con prefijo `lp-` (`lp-hero`, `lp-boton`). No uses `.btn`, `.card`, `.field`, `.container`, `.section` ni otras del tema.
- Sin jQuery ni librerías; si necesitas JS, plano y en un `<script>` al final del HTML. Imágenes con `alt`, `width`/`height` y `loading="lazy"` (menos la del hero).
- `{{landing}}` se sustituye por el nombre de la landing.
- Prohibido el número de WhatsApp y los enlaces `wa.me`. Teléfono y correo directos, tampoco: usa solo los botones de abajo.

**CSS**
- Se limita solo a la landing (el sistema prefija `.bh-lp`); `:root`, `html` y `body` equivalen al contenedor. `@keyframes`, `@font-face` y `@media` funcionan.
- Colores y fuentes propios son válidos en la landing. Móvil primero; breakpoint único de escritorio: `@media (min-width:1024px)`.
- No dependas del modo oscuro (se fuerza modo claro). No uses `position:fixed` con `z-index` > 90 (el modal de WhatsApp usa 100).

**Contacto (única vía; es lo que cuenta como conversión)**
- WhatsApp: `<button type="button" data-open-sheet="wa-gen" data-wa-origin="lp_{{landing}}">Escríbenos por WhatsApp</button>`. Abre el paso intermedio, registra el lead y dispara `lead_generado` (`lead_type: whatsapp`).
- Formulario estándar: `[bh_formulario titulo="..." boton="..." mensaje="si"]`.
- Formulario propio: `<form data-lead>` con `name` (obligatorio), `phone` y/o `email` (al menos uno), `message` opcional, `<div class="formmsg" role="status"></div>` y el shortcode `[bh_campos_ocultos]` dentro (honeypot, utm, `origin=lp_<nombre>`, Turnstile). Botón `type="submit"`. Al enviar con éxito se dispara `lead_generado` (`lead_type: contacto`, `lead_id = ref_code`).
- Aviso: incluye `<p class="privnote">` con el enlace al Aviso de privacidad o usa el pie de la landing.
- Campos extra (presupuesto, recámaras, fecha de visita, etc.): inputs con nombre `x_<clave>` (`x_presupuesto`, `x_recamaras`). Máx. 20, clave `a-z0-9_`, 300 caracteres. Llegan al CRM en `payload.extra.<clave>`. Los campos sin prefijo `x_` se ignoran.
- No crees otros eventos de conversión. Eventos propios de interacción son libres si no se llaman `lead_generado`.

**Casas**
- `[bh_casas zona="juriquilla" operacion="venta" tipo="casa" n="6" ids="" enlace="no" cta="Quiero ver más" orden="destacadas"]`
- Por defecto sin enlace a la ficha. `enlace="si"` enlaza a la ficha. `cta` agrega un botón de WhatsApp. `ids` fija propiedades concretas (ids de WordPress). Privadas y archivadas nunca salen.
- Marcado generado: `.lp-casas > .lp-casa > .lp-casa__img | .lp-casa__body (.lp-casa__price, __title, __loc, __specs)`. Rediseñalas con tu CSS: `.lp-casa{...}`.

## Analítica y conversiones (GTM)
- GTM se carga como en el resto del sitio (Personalizador › Analítica y GTM); no pegues otro contenedor ni `gtag` en la landing.
- En cada landing el `dataLayer` recibe `{page_type: "landing", landing_name: "<slug>"}` antes de que cargue GTM. Úsalos para disparadores y segmentos.
- `lead_generado` (única conversión) ahora trae `origin` (`lp_<nombre>` en landings; `header`, etc. en el sitio). Los UTM y click ids se guardan igual que en el sitio (sessionStorage, `attribution_first`/`last`, `_fbc`, `_fbp`, `_ga`, `landing_page`).
- Pruebas A/B: dos landings con distinto slug; compara por `landing_name`/`origin` en GA4 y en el CRM.

## Qué recibe el CRM
Igual que un formulario normal (ver `LEADS.md`): `name`, `phone`, `email`, `message`, atribución, `journey.landing_page`, `origin` (`lp_<nombre>`), más `extra` (objeto con los `x_*`). WhatsApp: lead local con `origin` y `ref_code`, confirmado por n8n.

## SEO
Por defecto `noindex, nofollow` y excluida de `wp-sitemap`. Indexable solo si se marca en la metabox. El SEO vive en `inc/seo.php`.

## Plugin
`includes/leads.php` acepta `x_*` y los guarda en `payload.extra`.
