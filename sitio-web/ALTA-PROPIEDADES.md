# Alta y edición de propiedades sin EasyBroker (diseño)

Estado: **diseño, sin implementar** (2026-11-05). Decisión de origen: Best Home pasa a ser la fuente de datos y fotos de las propiedades. Referencia de interfaz: capturas del editor de EasyBroker; no se copia tal cual, se conservan las mejoras propias. Actualización 2026-11-05: el alta es en **2 pasos** (no 4) y la lista tiene **búsqueda con filtros avanzados** (sección 1.1).

Lo que ya está definido y no se repite aquí: [CONTRATO-DATOS.md](CONTRATO-DATOS.md) (campos, IDs, estados, reglas de la sync, carpeta de fotos), [PLUGIN.md](PLUGIN.md) (CPT, metas, taxonomías) y [ARQUITECTURA.md](ARQUITECTURA.md) (catálogo y filtros). Este documento define **pantallas, fotos, migración, características y fases**.

Reglas que rigen todo el diseño: prefijos `bh_`; vanilla JS, sin jQuery ni constructores; metas `bh_priv_*` nunca públicas; no se toca `includes/sync.php` ni `includes/api.php` sin orden expresa (abajo solo se **proponen** cambios); una propiedad nunca se borra (solo cambia de estado); y, por la decisión del CRM simple (2026-10-24), la mayoría de los asesores captura desde el celular: **móvil primero**, pocos campos obligatorios.

---

## 1. Pantallas del admin

Todo vive dentro del plugin (`BH_EB_*`), en el menú **Propiedades**, para no depender del tema. Un solo archivo JS plano y un CSS propio del admin (sin build).

### 1.1 Lista de propiedades (mejorar la actual)
La lista nativa ya tiene columnas, orden y acciones. Cambios:
- Botón principal **«Nueva propiedad»** que abre el asistente (no el editor de bloques).
- **Buscador con panel «Filtros»** (botón que abre un panel lateral; en móvil, pantalla completa) y chips de filtros activos con «Limpiar». Reutiliza la lógica del catálogo público, pero **sin restricciones de visibilidad**: ve todas las propiedades (incluidas privadas, archivadas y borradores) y **todas** las características, no solo las de `bh_filtro = 1`. Filtros:
  - **Estado** (Borrador / Publicada / Privada / Archivada), origen (EasyBroker / Propia), operación y tipo.
  - **Ubicación, selección múltiple**: zonas padre y colonias (`zona`, `tax_query` con `IN`; al marcar un padre se pueden marcar todas sus colonias).
  - **Características: todas** las del catálogo (sección 4), agrupadas por categoría, con buscador de texto; modo **todas** (AND) o **cualquiera** (OR).
  - **Precio** (mín./máx., por moneda), **construcción** y **terreno** (m²), **recámaras**, **baños**, **estacionamientos** (mínimo, «3 o más»), condición, destacada/lujo.
  - **Etiquetas internas** (ver abajo) y **fecha** (creada, modificada, precio verificado, sin foto suficiente, sin verificar precio hace > N días).
  - **Búsqueda de texto interna**: un solo cuadro que busca en ID (`bh_eb_id`, `bh_internal_id`), título, **dueño o constructor** (`bh_priv_owner_name`, teléfono), **dirección y «ubicación o modelo»** (`bh_priv_address`, calle/colonia privadas), **desarrollo/constructor** (campo `bh_priv_developer`, nuevo, privado, con autocompletar de los ya usados) y **notas internas** (`bh_priv_notes`). Esta búsqueda solo corre en wp-admin con `edit_posts`; nunca se expone en el catálogo público, REST pública, feeds ni webhooks.
  - **Etiquetas internas**: meta `bh_priv_tags` (lista; autocompletar) o término de una taxonomía privada `bh_etiqueta` (`public => false`, solo admin); recomendación: taxonomía privada porque permite filtrar, contar y renombrar. Sirve para desarrollos, constructores, «oportunidad», «pedir actualización de precio», etc.
  - Las búsquedas se pueden **guardar** como vistas (por usuario, `user_meta bh_saved_views`) y se pueden **exportar a CSV** (sin teléfonos del dueño salvo administrador).
  - Implementación: `pre_get_posts` + `posts_clauses` en `edit-propiedad` (mismo patrón que la búsqueda por ID que ya existe), parámetros GET para que la URL sea compartible y funcione el botón «atrás»; JS plano para el panel; sin jQuery.
- Columna **Completitud** (círculo verde/ámbar/rojo): verde = cumple lo necesario para publicar; ámbar = faltan recomendados (fotos < 10, sin mapa, sin zona); rojo = falta algo obligatorio. Tocarla abre el paso que falta.
- Acción por fila **Editar** → asistente; **Duplicar** (copia datos y características, **sin** fotos ni ID; crea borrador con ID `BH-` nuevo); **Archivar** (ya existe vía visibilidad).
- En móvil, la lista pasa a tarjetas (foto, título, precio, estado, ID). El editor nativo de WordPress queda como respaldo («Edición avanzada») mientras se estabiliza.

No se replican de EasyBroker: Historial, Interesados, Listas, Portales conectados, «Sin asignar» (asesor), Crear reporte. Leads, visitas e interesados viven en el **CRM** ([crm/](../crm/)); el asesor por propiedad se decide con el CRM (ver preguntas).

### 1.2 Asistente de alta/edición en 2 pasos
Por decisión de Javier (2026-11-05) son **2 pasos**: **1 Información** (todo lo que describe la propiedad: datos, precio, ubicación, detalles, características) y **2 Multimedia y publicación** (fotos, video, tour 360, documentos y estado). Las secciones internas de EasyBroker (Lo esencial, Detalles, Multimedia, Publicación) pasan a ser **bloques plegables** dentro de esos dos pasos, para no perder el orden mental ni alargar la pantalla en el celular. Barra de progreso arriba; botones fijos abajo **Guardar y continuar** / **Guardar** / **Cancelar**. El paso 1 se puede guardar por **bloque** (cada bloque tiene su «Guardar» y hay un «Guardar y continuar» al final). Cada guardado escribe por REST (`besthome/v1/admin/propiedad/{id}`, nonce + `edit_posts`) sin recargar y deja la propiedad en **borrador** hasta el paso 4. Se puede salir y volver sin perder nada; los dos pasos son navegables en cualquier orden una vez creado el borrador.

El ID `BH-XX0000` se asigna **en el primer guardado** (así el slug `titulo-bh-xx0000`, las fotos y el texto alternativo nacen estables; el slug **no** cambia si luego se edita el título). Edición de una propiedad `EB-` ya migrada usa el mismo asistente.

#### Paso 1 — Información (bloques A «Lo esencial», B «Detalles» y C «Características»)
| Campo | Meta / destino | Oblig. | Validación |
|---|---|---|---|
| Tipo de propiedad | término `tipo_propiedad` | Sí | casa · departamento · terreno (ver pregunta sobre más tipos) |
| Operación | término `operacion` + `bh_operation` | Sí | venta · renta (una sola por propiedad; ver pregunta) |
| Título del anuncio | `post_title` | Sí | 20–110 caracteres; contador; sugerencia automática «{Tipo} en {operación} en {colonia}, {ciudad}» editable |
| Descripción | `post_content` | Rec. (avisa) | mín. 150 caracteres para publicar sin advertencia; sin teléfonos, correos ni enlaces (alerta, no bloqueo) |
| Precio | `bh_price` | Sí | número > 0; total (no por m²) |
| Moneda | `bh_currency` | Sí | MXN (def.) · USD |
| Mostrar precio | `bh_show_price` | Sí | Sí/No; «No» = «Consultar precio» |
| Mantenimiento mensual | `bh_maint_amount`, `bh_maint_currency` | No | número ≥ 0 |
| Mantenimiento incluido | `bh_maint_included` | No | No indicado · Sí · No (aplica sobre todo a renta) |
| Condición | `bh_condition` | No | nueva · usada · desarrollo (ya existe) |
| Colaboración y exclusividad *(sección plegada, interna)* | `bh_priv_exclusive` (no indicado/sí/no), `bh_priv_comm_shared` (sí/no), `bh_priv_comm_pct` (p. ej. 50), `bh_priv_comm_conditions` (texto) | No | **siempre privadas**; se suman a `bh_priv_commission` (comisión pactada) |
| Ubicación en el anuncio | `bh_loc_precision` | Sí | **Exacta** o **Aproximada** (def. aproximada) |
| Dirección | `bh_priv_street`, `bh_priv_number`, `bh_priv_interior`, `bh_priv_cross`, `bh_priv_zip` | No | privadas; `bh_priv_address` («ubicación o modelo») se conserva |
| Estado / Municipio / Colonia | `bh_location_name` («Colonia, Municipio, Estado») + término `zona` (padre › colonia) | Sí | listas dependientes; el padre sale de `zone_parents()`; colonia nueva = se crea el término (solo administrador) |
| Mapa | `bh_lat`, `bh_lng` (públicas) | Rec. | ver 1.3 |

Para terreno, el bloque B oculta recámaras/baños/estacionamientos; `bh_lot` pasa a obligatorio.

**Bloque B — Detalles** (todo opcional, con aviso «cada campo vacío es una búsqueda menos»)
Recámaras, baños, medios baños, estacionamientos (`bh_bedrooms`, `bh_bathrooms`, `bh_half_bathrooms`, `bh_parking`; enteros ≥ 0, medios baños por 1); construcción `bh_construction` y terreno `bh_lot` (m², > 0); campos nuevos: superficie no cubierta `bh_uncovered`, fondo `bh_lot_depth`, frente `bh_lot_front`, año de construcción `bh_year_built`, pisos del edificio `bh_floors` (ya existe), disposición (`bh_layout`: interior/exterior/frente…), orientación `bh_orientation`; **clave interna** `bh_internal_id` (ya existe, búsqueda) y **código de la llave** `bh_priv_key_code` (privado). Etiqueta destacada `_bh_badge` y «Destacada / Lujo» se quedan en el metabox lateral actual (o se mueven aquí, mismo contenido).
**Bloque C — Características** (sección 4), con casillas agrupadas por categoría y buscador de texto.

Sugerencia de lógica (no obligatoria): si `bh_construction` > `bh_lot` en casa de una planta, avisar; si el precio por m² queda fuera del rango de la zona, mostrar el aviso de `bh_price_review` que ya existe.

#### Paso 2 — Multimedia y publicación
**Bloque D — Multimedia.** Fotos (sección 2), **video de YouTube** (`bh_video_url`, valida que sea youtube.com/youtu.be/shorts y extrae el ID), **tour virtual** (`bh_tour_url`, https obligatorio; solo dominios permitidos: Matterport, Kuula, 3DVista, YouTube) y **documentos públicos** (planos, brochure, lista de precios): PDF ≤ 10 MB, hasta 5, meta `bh_docs` = `[{att_id, titulo}]`, se muestran como «Descargas» en la ficha. Lo que no deba ser público (contratos, escrituras) va en `bh_priv_onedrive` / adjuntos privados, nunca aquí. Documentos públicos es opcional y puede entrar en una fase posterior.

**Bloque E — Publicación.** Resumen con checklist (obligatorios en rojo, recomendados en ámbar), **vista previa** (abre la ficha pública con `?preview` en pestaña nueva) y selector de estado:
- **Borrador** (`post_status = draft`) · **Publicada** (`publish` + `bh_visibility = publica`) · **Privada** (`publish` + `privada`; abre por URL con `noindex`, útil para compartir con un cliente) · **Archivada** (`archivada`; vendida o retirada, pide motivo opcional).
- **No se puede publicar** sin: tipo, operación, título, precio (o «consultar»), moneda, zona/ubicación y **≥ 1 foto**. Con < 10 fotos o sin descripción se permite, con aviso.
- Bloque **interno** (no público): descripción privada = `bh_priv_notes` (el equivalente a «Descripción privada» de EB, con dueño, modelo, comisión, historial de precio), dueño `bh_priv_owner_name` / `bh_priv_owner_phone`, comisión `bh_priv_commission`, OneDrive `bh_priv_onedrive`, fecha de verificación de precio.
- **Etiquetas** de EB (casa, Juriquilla, Home Office…): **no** se replican como tal; ya existen términos de zona y características, y las etiquetas libres duplicarían filtros. Si se quieren notas de búsqueda internas, se usan en `bh_priv_notes`. (Pregunta abierta.)
- Mover de estado registra en el log de la propiedad (usuario y fecha), como ya hace `bh_archived_by`.

### 1.3 Ubicación y mapa (exacta vs aproximada)
- El sitio ya usa **Leaflet + OpenStreetMap** con círculo de 500 m sin pin ([ARQUITECTURA.md](ARQUITECTURA.md)); el admin reutiliza lo mismo (sin llaves ni costo).
- El asistente intenta ubicar con la dirección (Nominatim/OSM, solo desde el admin, ≤ 1 consulta/s con `User-Agent` propio); si falla, el asesor **arrastra el pin**. En móvil, botón «Usar mi ubicación actual» (útil cuando el asesor está en la casa).
- **Coordenadas privadas vs públicas** (cambio de privacidad): hoy `bh_lat`/`bh_lng` son públicas. Propuesta: la coordenada real se guarda en `bh_priv_lat`/`bh_priv_lng` (privadas). Las públicas `bh_lat`/`bh_lng` son iguales en modo **Exacta** y, en modo **Aproximada**, se desplazan de forma determinista (semilla = ID, 150–300 m) y la ficha dibuja el círculo de 500 m (ya existe). Así el feed, JSON-LD y REST no revelan la casa exacta. En las `EB-` migradas, las coordenadas actuales se conservan como públicas.
- Colonia/CP no cuadran con el pin (>2 km del centro de la colonia): aviso, no bloqueo.

---

## 2. Fotos

### 2.1 Dónde se guardan
- **Hoy**: URLs de `assets.easybroker.com` en `bh_images`; se pierden al cancelar EB. **Después**: **adjuntos de WordPress** (`post_parent` = la propiedad) en `uploads/propiedades/{ID}/` (filtro `upload_dir`), como ya define [CONTRATO-DATOS.md](CONTRATO-DATOS.md#fotos-pendiente-de-implementar).
- **Disco del VPS primero** (Hostinger KVM 2, ~100 GB NVMe): 150 propiedades × 25 fotos × ~0.5 MB ≈ 2 GB. Cloudflare (proxy + caché de `/uploads/`) sirve las imágenes sin costo extra. **CDN/almacenamiento externo (Cloudflare R2) es opcional y posterior**: como las fotos son adjuntos de WordPress, mover `/uploads/` a R2 con un plugin de offload no cambia la base de datos ni el código del tema. No se necesita para arrancar.
- **Backups**: `uploads/propiedades/` entra en el snapshot de Hostinger y en el respaldo de CloudPanel ([servidor/](../servidor/)); sin foto original perdida no hay manera de recuperarla de EB después de cancelarlo, así que esto es requisito **antes** de cancelar.

### 2.2 Estructura de datos
- `bh_gallery` (array de IDs de adjunto en orden). La **primera** es la portada y se copia a `_thumbnail_id` (imagen destacada: og:image, feed, tarjetas).
- Por foto (meta del adjunto): `bh_foto_desc` = «¿Qué hay en esta foto?» (p. ej. «Fachada», «Cocina equipada»); `bh_foto_src` = URL de origen (solo migradas, para idempotencia); `bh_foto_hash` = sha1 (evita duplicados).
- `bh_prop()['images']` devuelve primero `bh_gallery` (URLs `large`) y, si está vacía, cae a `bh_images` de EasyBroker → **compatible durante la migración** y sin tocar la sync. Hay que actualizar `bh_prop()`, `single-propiedad.php`, `property-card`, feed `g:image_link`, `ficha-print.php` y el JSON-LD para que lean de `bh_prop()` (cambio de tema, no de sync).

### 2.3 Límites y formato
- **Máximo 40 fotos por propiedad** (igual que EB); **recomendado ≥ 10** (aviso, no bloqueo); mínimo 1 para publicar. 
- Cada archivo: JPG, PNG, WebP o HEIC (iPhone) hasta **15 MB** al subir.
- **El navegador reduce antes de subir** (canvas, vanilla JS): lado largo máx. **2560 px**, WebP calidad ~82 (JPEG si el navegador no soporta WebP), corrige orientación EXIF. Ahorra datos en el celular y evita límites de `upload_max_filesize`. HEIC que el navegador no pueda decodificar se sube tal cual y lo convierte el servidor (si Imagick lo soporta; si no, error claro «usa JPG»).
- **Servidor**: genera solo `bh-card` (800×600 recorte), `large` (1600) y `bh-thumb` (400) en WebP; se desactivan los demás tamaños intermedios para esta carpeta. 
- Subida en paralelo (3 a la vez) con barra por foto, reintento si falla y cola que sobrevive a recargar.

### 2.4 Interfaz (paso 2, bloque D)
- Tarjetas como en EB: miniatura, «¿Qué hay en esta foto?», **abrir** (ampliar), **girar** (↻ 90°, en navegador antes de guardar o regenerando en servidor), **borrar** (🗑 con confirmación) y asa de **arrastrar** (⠿).
- **Ordenar**: arrastrar y soltar (HTML5 drag, o Pointer Events para táctil) **y** botones «subir/bajar» y «Hacer portada» (arrastrar en celular es poco confiable; los botones son la vía principal en móvil).
- **Portada** marcada con insignia; «Quitar todas» pide confirmación escribiendo el ID.
- **Subir varias a la vez** desde cámara o galería (`<input type=file multiple accept="image/*">`).
- Sugerencias de descripción (botones rápidos de un toque): Fachada, Sala, Comedor, Cocina, Recámara principal, Baño, Jardín, Terraza, Alberca, Estacionamiento, Plano, Vista, Amenidades del fraccionamiento; editable.

### 2.5 Texto alternativo (alt) automático
Regla (se genera en el render, no se guarda a mano):

```
{descripción de la foto} - {Tipo} en {operación} en {colonia, municipio} - {ID}
ej.: «Fachada - Casa en venta en Grand Juriquilla, Querétaro - EB-XF4820»
```
- Sin descripción: `Foto {n} - {Tipo} en {operación} en {colonia, municipio} - {ID}` (o solo la parte de la propiedad en la portada).
- Operación: «en venta» / «en renta». Colonia y municipio salen de `bh_location_name` (sin estado). Máx. ~125 caracteres (se recorta la ubicación primero).
- Se calcula con una función `bh_foto_alt($att_id, $post_id)` usada por `wp_get_attachment_image_attributes` y las plantillas; así **cambia sola** si se corrige la zona, el tipo o la operación. Además se escribe en `_wp_attachment_image_alt` al guardar (para REST/feed/biblioteca).
- `title` de la foto = descripción; `caption` vacío. Las fotos decorativas no existen: todas llevan alt.
- El ID va al final porque el alt también lo ve quien busque la clave; no se incluye teléfono ni precio.

---

## 3. Migración de EasyBroker al servidor

Premisa: **los datos ya están en WordPress** (la sync escribe todas las metas, términos, notas y `bh_images`). Lo que falta migrar son las **fotos** y el **cambio de dueño de los datos** (`bh_source`).

### 3.1 Fotos (script idempotente)
Comando WP-CLI `wp bh migrar-fotos` (o botón en Ajustes › EasyBroker Sync para correrlo sin consola; mismo código):
1. Recorre propiedades con `bh_source = easybroker` (o sin meta) y `bh_images` no vacío; opción `--id=EB-XF4820`, `--limite=N`, `--dry-run`.
2. Por cada URL: si ya existe un adjunto con `bh_foto_src` = esa URL (o igual `bh_foto_hash`), **se salta**; si no, `download_url()` + `media_handle_sideload()` hacia `uploads/propiedades/{ID}/`, con reducción a 2560 px/WebP en servidor, y guarda `bh_foto_src`, `bh_foto_hash`, descripción vacía (o el `title` de la foto si la API de EB lo devuelve; verificar con una propiedad).
3. Conserva el **orden** de `bh_images`; escribe `bh_gallery` solo cuando **todas** las fotos de esa propiedad bajaron bien; la 1.ª va a `_thumbnail_id`. Mientras tanto la ficha sigue sirviendo `bh_images`.
4. Lotes con presupuesto de tiempo (como la sync, ~25 s) y reanudable con cron único; una propiedad por pasada si hace falta; pausa corta entre descargas y respeto a 429.
5. Registro (`bh_mig_log`, mismo estilo que `bh_eb_log`): propiedad, fotos esperadas / bajadas / saltadas / fallidas y bytes. Verificación final `wp bh verificar-fotos`: cuenta, archivo existe y > 0 bytes, ancho mínimo, ninguna URL de `assets.easybroker.com` queda en el HTML de la ficha, el feed ni el JSON-LD.
6. Es seguro volver a correrlo: solo completa lo que falta. Si EB agrega una foto después, otra corrida la trae (mientras la propiedad siga en `easybroker`).
- La sync sigue pisando `bh_images`, pero no toca `bh_gallery`, adjuntos ni `_thumbnail_id` (ya listados como «nunca toca» la imagen destacada); **no hace falta cambiar `sync.php`** para esta etapa.

### 3.2 «Tomar el control» (cambio de `bh_source`)
- Acción por propiedad (y masiva con filtro) **«Pasar a Best Home»**: exige fotos migradas y verificadas, cambia `bh_source` a `propio`, **conserva `bh_eb_id`** (el `EB-…` es permanente: slug, feed, QR, PDFs, leads) y no asigna un ID `BH-` nuevo. `bh_status` y `bh_updated_at` quedan inertes. Guarda `bh_migrated_at` y `bh_migrated_from = EB-…`.
- Desde ese momento la sync **no crea, modifica ni archiva** la propiedad (regla ya vigente para `propio`), así que cambios hechos en el sitio no se pisan. El riesgo está en el sentido inverso: si aún se edita algo en EB después del cambio, **no llegará** al sitio; por eso el cambio se hace por lotes y se avisa al equipo.
- Orden: 1) migrar y verificar fotos de todas; 2) piloto con 3–5 propiedades (incluida una archivada y una privada); 3) pasar el resto; 4) desactivar el cron de sync (`bh_eb_incremental_sync`, `bh_eb_full_sync`) y la clave API; 5) cancelar EB. Antes del paso 4, la sync **sigue necesaria** para propiedades nuevas que se capturen en EB.
- Reversible hasta cancelar EB (volver a `easybroker` re-activa la sync; se perderían las ediciones hechas en el sitio, así que se avisa).
- **Cambio a proponer (sin hacerlo)**: tras cortar EB, retirar `sync.php`/`api.php` del arranque o dejarlas desactivadas por constante `BH_EB_DISABLED`; es una decisión para después, con orden expresa.
- Los nuevos IDs `BH-XX0000` se generan en el primer guardado: 2 letras + 4 dígitos, sin `I` ni `O`, únicos contra `bh_eb_id` **y** `bh_internal_id`; función `bh_new_property_id()` con reintento y bloqueo (transient) para evitar colisiones por doble clic.
- Verificación de precios: la tabla pasa a **editar el precio** directamente (ya en [PENDIENTES.md](../PENDIENTES.md)) cuando la fuente sea el sitio, con historial (`bh_price_history`, ya existe).

### 3.3 Qué cambia en el resto del sistema
Feed de anuncios, sitemap, JSON-LD, PDF y leads siguen igual (usan `bh_eb_id`). REST de catálogo (`/besthome/v1/…`) y n8n: sin cambios. Si se hiciera un feed hacia portales, quedaría para «Banderas por portal» (futuro).

---

## 4. Catálogo de características editable

### 4.1 Modelo
Hoy: 10 términos de `amenidad` (los usados en filtros), el resto de características de EB solo en `bh_features`/`bh_features_full` (texto). Propuesta: **todas las características son términos de `amenidad`**, con metadatos de término (`wp_termmeta`), y una pantalla propia **Propiedades › Características** que se ve como tabla:

| Columna | Meta de término | Notas |
|---|---|---|
| Nombre | nombre del término | «Alberca», «Cuarto de servicio»… |
| Slug | slug | **no se cambia** en las 10 existentes (URLs `/amenidad/…` y `amenidad[]` ya indexadas) |
| Categoría | `bh_cat` | Amenidades · Exterior · General · Recreación · Políticas (lista editable en una opción `bh_feat_cats`) |
| **Usar en filtros** | `bh_filtro` = 1/0 | casilla; solo con 1 aparece como filtro/chip |
| Orden | `bh_orden` | dentro de su categoría |
| Icono | `bh_icono` (opcional) | texto/emoji o nombre de ícono del tema |
| Activa | `bh_activa` | desactivar = ya no aparece en el alta, pero conserva su historial |
| Propiedades | conteo | cuántas la tienen |

- Botón **«Agregar característica»** (nombre + categoría + ¿usar en filtros?) y **fusionar** dos (p. ej. «Roof garden»/«Roof Garden»). Borrar solo si no se usa.
- Se **siembra** con el catálogo de EB de las capturas (~100: Amenidades, Exterior, General, Políticas, Recreación) con `bh_filtro = 0`, y con `bh_filtro = 1` las 10 actuales (una-planta, recamara-en-planta-baja, roof-garden, estudio-home-office, sala-de-tv, cuarto-de-juegos, cuarto-de-servicio, alberca-privada, alberca-en-fraccionamiento, terraza). El mapeo de las de EB reutiliza el de `BH_EB_Sync` (solo lectura); lo demás se crea por nombre normalizado desde `bh_features_full`.
- **Por qué taxonomía y no tabla SQL propia**: `pre_get_posts` ya arma `tax_query` con `amenidad[]`, existen URLs, chips y conteos; una taxonomía con termmeta es «una tabla en la base de datos» editable, sin reescribir catálogo ni rutas. Una tabla aparte exigiría reescribir filtros y consultas.
- Coincide con el catálogo del CRM (las 10 amenidades de [crm/](../crm/)): el CRM sigue usando solo las `bh_filtro = 1` hasta que se decida otra cosa.

### 4.2 Relación con filtros y ficha
- `filters.php` y `bh_active_filters()` (chips) leen **solo** términos con `bh_filtro = 1`, agrupados por `bh_cat` y ordenados por `bh_orden` (resultado en transient `bh_filter_terms`, se invalida al editar un término o la pantalla de características). Una característica `bh_filtro = 0` **no** genera chip ni opción, pero **sí** se muestra en la ficha.
- **Admin**: la lista de propiedades (1.1) filtra por **todas** las características (`bh_filtro` 0 o 1); `bh_filtro` solo decide lo que ve el público.
- Ficha: sección «Características» agrupada por categoría (como la vista de EB: Exterior, General, Recreación…), con los íconos de `bh_icono` si hay; hoy sale de `bh_features`. Después de la migración `bh_features`/`bh_features_full` se conservan sin editar (respaldo) y la ficha lee de los términos.
- **Alta**: casillas agrupadas por categoría con buscador y «+ Agregar nueva» (solo administrador; el asesor ve las existentes). Convertir una característica en filtro es una casilla en la tabla: aparece en el catálogo sin tocar código.
- Si una característica activa en filtros tiene muy pocas propiedades (< 3), la tabla lo avisa («el filtro devolverá casi nada»).
- Políticas (mascotas, fumar) pueden tener `bh_filtro = 1` pero se recomienda que solo aplique a renta (`bh_cat = Políticas` filtra por operación).
- `bh_lock_terms` deja de ser necesario cuando la propiedad es `propio`.
- Cambios en `sync.php` para escribir en el nuevo catálogo: **solo propuesta**; mientras tanto la sync sigue asignando las 10 y guardando `bh_features`, y el script de siembra completa el resto una vez.

---

## 5. Fases, riesgos y orden recomendado

| Fase | Contenido | Depende de |
|---|---|---|
| **0 Respaldo y decisiones** | Respuestas a las preguntas de abajo; snapshot del VPS/sitio; confirmar espacio en disco. | — |
| **1 Fotos (lo urgente)** | `upload_dir` + tamaños WebP, `bh_gallery`, `bh_prop()['images']` con respaldo a `bh_images`, alt automático, script `migrar-fotos` + verificación. Se puede correr **sin interfaz nueva** y reduce el riesgo más grande: perder fotos al cancelar EB. | 0 |
| **2 Catálogo de características** | Pantalla Propiedades › Características, termmeta, siembra, filtros/chips y ficha leyendo de ahí. Independiente de la fase 3. | 0 |
| **3 Asistente de 2 pasos** | Pasos 1–2 y 4 con generador `BH-`, guardado por REST, validaciones, completitud; luego paso 3 con subida, orden, portada, alt. Lista mejorada. | 1, 2 |
| **4 Ubicación privada/aproximada y mapa** | `bh_priv_lat/lng`, desplazamiento determinista, mapa del admin. | 3 |
| **5 Corte de EasyBroker** | «Pasar a Best Home» por lotes (piloto → todas), apagar cron de sync y API, cancelar EB. | 1, 3 y 2 semanas de uso real |
| **6 Extras** | Documentos públicos, duplicar propiedad, banderas por portal, formulario ligero para asesores en móvil (solo lo esencial y fotos), precio editable en Verificación. | 3 |

Riesgos y mitigación:
- **Perder fotos al cancelar EB** → fase 1 antes que todo; verificación automática; no cancelar hasta 0 URLs de EB en el HTML y un respaldo confirmado.
- **Disco y rendimiento** → reducción a 2560 px/WebP y solo 3 tamaños; vigilar espacio; R2 como salida.
- **Subida en celular** (conexión mala, HEIC, 40 fotos) → reducción en el navegador, cola con reintento, guardar por paso.
- **Pisar datos** (sync sobre una propiedad ya propia, o editar en EB tras el corte) → corte por lotes, avisar al equipo, la sync ya excluye `propio`.
- **Filtros demasiados** → `bh_filtro = 0` por defecto; solo las 10 actuales.
- **Privacidad** → dirección, coordenada exacta, comisión, exclusividad, dueño y llave solo en `bh_priv_*`; probar que REST, feed, JSON-LD y n8n no los exponen (misma prueba que hoy).
- **SEO** → no cambian URLs ni slugs ni `EB-…`; las fotos nuevas conservan alt con ID y ubicación. Tras tocar rewrites o el CPT, **guardar Ajustes › Enlaces permanentes**.
- **Carga para el asesor** (CRM simple) → solo 7 obligatorios en el alta: tipo, operación, título, precio, moneda, zona y 1 foto.
- Al cerrar cada fase: actualizar `CONTRATO-DATOS.md`, `PLUGIN.md`, `ARQUITECTURA.md`, `CAMBIOS.md` y subir `BH_THEME_VERSION` si se toca CSS/JS del tema.

## 6. Preguntas abiertas para Javier
1. **Asesor por propiedad**: ¿cada propiedad tiene asesor responsable (como «Sin asignar» de EB)? ¿Vive en el CRM o en el sitio?
2. **Tipos de propiedad**: hoy solo casa, departamento y terreno. ¿Agregamos local, oficina, bodega, rancho?
3. **Una propiedad en venta y renta a la vez**: EB lo permite en una sola ficha; nosotros usamos una operación por propiedad (para duplicar una ficha con otro ID). ¿Te sirve así?
4. **Renta temporal**: ¿se descarta?
5. **Etiquetas libres de EB** (Home Office, Fracc con alberca): ¿las reemplazamos por características/etiqueta destacada, o las quieres como etiquetas internas de búsqueda?
6. **Quién captura y desde dónde**: ¿solo tú y el equipo en computadora, o los asesores por celular? Si es celular, la fase 3 prioriza un formulario ligero (solo el paso 1 reducido y las fotos).
7. **Límite de fotos**: ¿40 como EB, o menos (p. ej. 30) para cuidar disco y velocidad?
8. **Ubicación pública**: ¿todas las propiedades «Aproximadas» por defecto (como ahora, círculo de 500 m)? ¿Alguna «Exacta» (desarrollos)?
9. **Documentos públicos** (planos/brochure): ¿los quieres en la ficha ahora o después?
10. **Corte con EasyBroker**: ¿hay fecha de renovación o cancelación? Eso fija el calendario de la fase 1 y 5. ¿Siguen publicando en portales (Inmuebles24, etc.) por EB? Si sí, hay que resolver ese envío antes de cancelar.
11. **Texto alternativo**: ¿la regla `{descripción} - {tipo} en {operación} en {colonia, municipio} - {ID}` está bien, o prefieres sin guion y con ID antes?
12. **Desarrollo / constructor**: ¿lo capturamos como campo privado aparte (`bh_priv_developer`) para buscar por él, o basta con el dueño y las etiquetas internas?
13. **Etiquetas internas**: ¿taxonomía privada (recomendada, se renombran y cuentan) o texto libre por propiedad?
