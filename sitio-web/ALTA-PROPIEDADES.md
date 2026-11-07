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
| Tipo de propiedad | término `tipo_propiedad` | Sí | casa · departamento · terreno; **lista editable** (Propiedades › Tipos de propiedad: agregar, renombrar, ordenar) |
| Operación | término `operacion` + `bh_operation` | Sí | venta · renta, **una por propiedad**; si se vende y renta, se **duplica** la propiedad (acción «Duplicar»). Sin renta temporal |
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
- **Etiquetas** de EB (casa, Juriquilla, Home Office…): **decidido: son las características**; no hay etiquetas libres; ya existen términos de zona y características, y las etiquetas libres duplicarían filtros. Si se quieren notas de búsqueda internas, se usan en `bh_priv_notes`. (Pregunta abierta.)
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
- **Sin límite duro**: aviso amarillo desde 50 fotos («ya son muchas, ¿seguro?») y tope técnico de 100 por propiedad; **recomendado ≥ 10** (aviso, no bloqueo); mínimo 1 para publicar. (EasyBroker permitía ~50.) 
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

## 6. Decisiones de Javier (2026-11-05)
- **Asesor por propiedad**: opcional; por defecto **sin asesor**. Se puede asignar (campo `bh_priv_asesor` = usuario WP, privado) para saber de quién es la casa en el tema de comisión. Filtro «Asesor» en la lista. No se publica teléfono por asesor.
- **Tipos de propiedad**: casa, departamento y terreno (no se separa casa en condominio); como **tabla editable**.
- **Venta y renta**: se duplica la propiedad. **Renta temporal**: no se usa.
- **Etiquetas = características**: todo se maneja por características, en una tabla con **ID propio**, **categoría** (Amenidades, Exterior, General, Recreación, Políticas…) y editable (renombrar, agregar, agrupar) sin tocar código (sección 4). El nombre se muestra desde la tabla; el ID/slug no cambia al renombrar.
- **Captura**: Javier da de alta casi todo, en computadora. Móvil: sobre todo **editar**; el formulario de alta móvil queda de baja prioridad.
- **Asesores**: **solo ver y buscar** (no editan nada), con login, viendo datos exactos (sección 7).
- **Fotos**: límite abierto con avisos (sección 2.3).
- **Ubicación**: pública **aproximada por defecto**, con opción **exacta** por propiedad (exclusivas, terrenos). En el panel (admin y asesores) siempre se ve la **exacta**.
- **Documentos**: no hay brochures ni documentos públicos. **Planos** = imágenes, pero **separadas de las fotos**: galería propia `bh_planos` (normalmente 2–3), con **pestaña «Planos»** en la ficha (junto a Fotos/Video/Recorrido 3D/Mapa) y su propio bloque en el paso 2. El bloque «Documentos públicos» (PDF) se descarta.
- **Texto alternativo**: la regla acordada se queda. En planos: `Plano {n} - {Tipo} en {operación} en {colonia, municipio} - {ID}`.
- **Desarrollo / constructor**: campo privado `bh_priv_developer`, **buscable**, con autocompletar.
- **Portales**: no se publica en portales; solo el sitio.
- **Corte con EasyBroker**: la renovación es 2026-11-06 y **no se renueva**. Migración de fotos y datos en los **3 días siguientes**.

## 7. Vista de asesores (solo lectura, con login)
Recomendación: **página privada en el sitio** (`/asesores/`), no el wp-admin, para que funcione bien en celular y sin exponer el panel.
- Rol nuevo `bh_asesor` (capacidad propia `bh_ver_propiedades`; sin `edit_posts`). Entra por `wp-login.php`; al entrar se redirige a `/asesores/` y se les oculta la barra y el wp-admin. Solo el administrador crea usuarios y edita (propiedades, características, etiquetas, tipos).
- La página es `noindex, nofollow`, sin caché (`DONOTCACHEPAGE`, `Cache-Control: private`), excluida del sitemap y con regla de Cloudflare para no cachearla. Sin sesión → login; sin rol → 403.
- Misma búsqueda y panel de filtros que la lista del admin (sección 1.1; todas las características; búsqueda por dueño, dirección, desarrollo y etiquetas) pero de **solo lectura**.
- Campos que ven (más que el público): ubicación exacta con mapa, dirección, desarrollo/constructor, dueño, comisión, notas internas, asesor asignado, clave de llave. **No ven** teléfono del dueño si Javier decide ocultarlo (opción por campo en Ajustes; pendiente confirmar).
- Acciones: copiar enlace público, PDF con/sin marca, compartir por WhatsApp (el enlace de la ficha). Sin botones de edición.
- Seguridad: cada consulta verifica el rol en el servidor (REST `besthome/v1/asesores/…`, nonce + `current_user_can('bh_ver_propiedades')`); los campos `bh_priv_*` solo salen por esta ruta y nunca por la REST pública, feeds ni webhooks. Registrar accesos (usuario, fecha) para auditoría.

## 8. Calendario del corte (renovación 2026-11-06)
- **Hoy (día 0)**: script de fotos listo (`includes/migrar-fotos.php`, ver 3.1). Correr «Propiedades › Migrar fotos» o `wp bh fotos migrar`; después `wp bh fotos verificar`. Mientras EB esté activo las URLs de `assets.easybroker.com` siguen vivas.
- **Día 1–2**: revisar fotos en la ficha, el feed y el PDF; respaldo del sitio con `uploads/propiedades/`; probar con 3–5 propiedades «Pasar a Best Home».
- **Día 3**: pasar todas a `propio`, apagar la sync y la clave API; confirmar que no quedan URLs de EB.
- **Riesgo**: no se sabe cuánto tiempo conservan las URLs de EB tras vencer la cuenta → bajar las fotos **antes** de que venza, aunque el resto del plan tome más días.
- **Al mover el sitio al VPS** (Fase 2): `bh_images` guarda URLs absolutas del sitio actual; hay que hacer `wp search-replace` del dominio (incluye arrays serializados) y copiar `uploads/propiedades/`.

## 9. Roles y permisos (decisión 2026-11-06)
**Actualización 2026-11-06**: el administrador de WordPress es solo Javier. El **gerente** tiene un rol propio (`bh_gerente`) con vista sencilla de propiedades y más permisos que el asesor, **sin** menús de configuración de WordPress. **La tabla de permisos es configurable**: pantalla **Ajustes › Roles de Best Home** con una matriz de casillas (filas = permisos, columnas = roles), y botón «Crear rol» (copia de uno existente) para roles nuevos. Los roles base son administrador, gerente y asesor. El login es el de **WordPress** (usuario y contraseña, con límite de intentos y Turnstile que ya existe en el sitio): los permisos por rol viven ahí. **Cloudflare Access** (código por correo, gratis hasta 50 usuarios) se puede poner **encima** solo de `/wp-admin` y `/wp-login.php` para administrador y gerente cuando el sitio esté en el VPS (Fase 2); para los asesores no se recomienda, porque sería un segundo login en cada celular. Empezar solo con el login de WordPress.

| Capacidad | Administrador (solo Javier) | Gerente (`bh_gerente`, el «intermedio») | Asesor (`bh_asesor`) |
|---|---|---|---|
| Buscar y ver propiedades (`/asesores/`, filtros, todas las características) | Sí | Sí | Sí |
| Ver ubicación exacta, desarrollo/constructor | Sí | Sí | Sí |
| Ver dueño, teléfono del dueño, comisión y notas internas (`bh_priv_*`) | Sí | Sí | **No** (solo la comisión de las propiedades que tiene asignadas) |
| Copiar enlace, PDF, compartir | Sí | Sí | Sí |
| Ver leads (`bh_view_leads`) | Sí | Sí | No |
| Editar datos y fotos de propiedades | Sí | Sí (sin cambiar precio ni estado; configurable) | No |
| Publicar, archivar, cambiar visibilidad | Sí | No | No |
| Tabla de características, tipos, etiquetas | Sí | No | No |
| Importar desde EasyBroker, pasar a propia | Sí | No | No |
| Tema, plugins, ajustes, usuarios | Sí | No | No |

- Capacidades propias (`bh_ver_propiedades`, `bh_ver_privados`, `bh_editar_propiedades`, `bh_publicar`, `bh_gestionar_catalogos`, `bh_importar_eb`); los roles personalizados **no** tienen `edit_posts` ni `upload_files` de WordPress, así que no pueden entrar a lo demás del wp-admin. Los permisos finos (por ejemplo, que el intermedio sí pueda cambiar el precio) se activan con casillas en una pantalla **Ajustes › Roles de Best Home**, sin tocar código.
- Activar verificación en dos pasos para el administrador y confirmar `DISALLOW_FILE_EDIT` (editor de archivos desactivado) en `wp-config.php`.
- Todos los roles con acceso a `/asesores/` (asesor y gerente) pueden **mandar fichas**: PDF con marca y sin marca, copiar enlace y compartir por WhatsApp; son permisos de la matriz (`bh_enviar_fichas`).
- Los asesores entran por `wp-login.php` y se redirigen a `/asesores/` (sección 7); cada consulta valida la capacidad en el servidor.

## 10. Importar desde EasyBroker por ID (uso esporádico)
Contexto: Javier mantendrá un plan **más bajo** de EasyBroker (≈10 propiedades activas) donde queda el historial. Si una propiedad antigua vuelve a venderse, se activa en EasyBroker y se **copia aquí** sin capturarla de nuevo. La sync continua (cron) deja de usarse; el plugin pasa a ser una **herramienta de importación manual**.

- Pantalla **Propiedades › Importar de EasyBroker**: cuadro para uno o varios IDs (`EB-XF4820`) y botón «Importar». Solo administrador (`bh_importar_eb`). La clave API sigue configurada (`BH_EB_API_KEY`), pero **los crons `bh_eb_incremental_sync` y `bh_eb_full_sync` se apagan**.
- Qué hace, reutilizando sin modificar `BH_EB_API::property()`, `BH_EB_Sync::map_detail()` y `BH_EB_Sync::upsert()`:
  1. Consulta el detalle en EasyBroker (descripción, precio, operación, tipo, ubicación, características, video, tour y fotos).
  2. **Si la propiedad ya existe en el sitio** (archivada o privada): muestra qué cambiaría (precio, descripción, fotos nuevas) y deja elegir **«Actualizar todo»** o **«Solo lo que falta»**; después la pone en **Pública** (`bh_visibility = publica`, `bh_archived_*` limpio). Como `upsert()` omite las propiedades `propio`, el importador cambia `bh_source` a `easybroker` solo durante la copia y la regresa a `propio` al terminar.
  3. **Si no existe**: la crea con su mismo `EB-…` (permanente), zona y características mapeadas.
  4. Descarga las fotos con el mismo código de migración (`BH_EB_Fotos::migrar()`), sin duplicar, y deja `bh_images` con URLs locales y `bh_gallery` en orden.
  5. Deja `bh_source = propio`, así queda como cualquier otra propiedad propia y se edita en el sitio. Registra usuario, fecha y resultado en un log.
- Notas: la API de EasyBroker puede no devolver propiedades **no publicadas**; por eso el flujo es «activar en EB → importar → editar aquí». Hay que **probarlo** con una propiedad archivada. Con el plan bajo, EasyBroker solo se usa para ese paso.
- Cambios a `sync.php`/`api.php`: **ninguno**; solo se reutilizan sus funciones públicas.

## 11. Planos separados de las fotos
- Misma interfaz de visor, **pestaña propia «Planos»** en la ficha (Fotos · Video · Recorrido 3D · **Planos** · Mapa); solo aparece si la propiedad tiene planos.
- Datos: `bh_planos` (IDs de adjunto en orden), independiente de `bh_gallery`. Cada adjunto lleva `bh_foto_tipo` = `foto` | `plano`.
- Alta/edición (paso 2): dos bloques con el mismo control de subir, arrastrar y descripción: **Fotos** y **Planos**. Cada foto tiene el botón **«Mover a planos»** (y cada plano «Mover a fotos»), para pasar a planos las que hoy están mezcladas en el carrusel sin volver a subirlas.
- Los planos **no** entran en `bh_images` (feed de anuncios, JSON-LD, tarjetas, PDF de ficha como galería); la ficha PDF puede imprimirlos en una página aparte. Alt: `Plano {n} - {Tipo} en {operación} en {colonia, municipio} - {ID}`.
- Migración: la migración copia todas las fotos de EB como fotos; después Javier mueve a planos las que correspondan (suelen ser 2 o 3 por propiedad). Se puede ayudar con una vista «Revisar planos» que muestra la última foto de cada propiedad (los planos suelen ir al final) para moverlas rápido.
- Cambio de tema pendiente: `bh_prop()['planos']`, pestaña y visor en `single-propiedad.php`, CSS y JS (vanilla). Subir `BH_THEME_VERSION`.

## 12. Carpetas y tamaños de imagen
- **`uploads/propiedades/{ID}/`** (lo que ya hace el script): el archivo original más sus versiones; todo junto, no hay que moverlo. Cada propiedad ≈ original (≤ 2560 px) + `-800x600` (tarjeta del catálogo) + `-1024` o `-1600` (`large`) + `-150x150` (miniatura del admin). Son pocos tamaños a propósito (el script desactiva el resto), porque WordPress por defecto genera 5 a 8 versiones de cada foto y llena el disco.
- **Fotos y planos en la misma carpeta**: no se separan físicamente. La verdad es el dato `bh_foto_tipo` (y `bh_gallery` / `bh_planos`); mover archivos de carpeta rompería URLs sin ganar nada. Los planos nuevos se nombran `{id}-plano-01.jpg` para reconocerlos al verlos en el explorador.
- **`uploads/bh-ficha/`**: es el caché de la ficha PDF; las fotos de casas que ves ahí son copias que se bajaron para imprimir (cuando las fotos estaban en EasyBroker). Es seguro borrarla; se regenera sola. Con las fotos ya locales, la ficha puede leer directamente de `uploads/propiedades/` y esa carpeta deja de crecer. Pendiente de confirmar al tocar `ficha-print.php`.
- Lo que ya se migró **no hay que mover**: ya está donde debe. Los tamaños extra que WordPress hubiera creado antes de este script (si los hay) se pueden regenerar/borrar con un comando de limpieza (`wp bh fotos limpiar-tamanos`, por hacer) tras el corte.
- Orden futuro opcional: conversión a WebP y CDN/R2 (sección 2).

## 13. URLs viejas de EasyBroker (corregido 2026-11-06, con ejemplos reales)
Dos dominios distintos:
- **Sitio de Javier alojado en EasyBroker**: `https://www.besthomeqro.com/property/{slug}` (ej. `.../property/casa-en-venta-en-grand-juriquilla-queretaro-3-recamaras-y-jardin-de-40m2`; a veces el slug termina en un UUID). **Estas son las URLs de los anuncios** y **sí se pueden redirigir**: cuando el dominio apunte al VPS, llegarán al WordPress nuevo. El slug no lleva el ID `EB-…` y viene del título **que tenía** la propiedad (puede haber cambiado).
- `https://www.easybroker.com/listings/{slug}`: es lo que devuelve la API (`public_url`); no se pueden redirigir (dominio de EasyBroker). Se usa como pista: se asume que el slug de `/property/{slug}` es el mismo que el último tramo de `public_url` (**supuesto por verificar** con las URLs de ejemplo).
- **Regla de redirección 301** (`includes/redirecciones.php`), solo cuando la petición sería 404 y conservando `utm_*`/`gclid`: (a) ruta guardada; (b) para `/property/{slug}` y `/listings/{slug}`: slug = último tramo de `bh_eb_public_url` guardada, o slug igual al del título de la propiedad nueva; (c) mapa manual (`/ruta => EB-XF4820` o `=> /venta/casas/`); (d) ID `EB-XX0000` en la ruta; (e) prefijos de EasyBroker (`property`, `listings`…) → `/venta/` (o `/renta/`). Lo que no se pueda mapear cae en el catálogo. La pantalla registra las URLs que llegan al 404 y tiene un probador.
- **CSV** «URL anterior (besthomeqro.com/property/…), URL de EasyBroker, URL nueva, visibilidad» para reemplazar los enlaces de Google Ads (descarga en Propiedades › Redirecciones).
- La captura (`bh_eb_public_url`) se hace una vez mientras la API devuelva las propiedades (ya corrida por Javier: ~230 propiedades revisadas). Tras cambiar el dominio, **probar las 5 URLs de ejemplo** y las que más tráfico tengan (Google Ads, Search Console).
- Pendiente de confirmar: si las propiedades vendidas/archivadas conservan URL en la API y si la regla de slug acierta en las 9 URLs de ejemplo; los que no acierten se agregan al mapa manual.

## 14. Orden de implementación (acordado: «empezar los cambios»)
1. **Hecho/entregado**: script de fotos (`migrar-fotos.php`) — Javier ya lo usó; herramienta de redirecciones (`redirecciones.php`) — por correr.
2. **Pasar todas a propias** (`bh_source = propio`) cuando las fotos estén verificadas; apagar los crons de sync (el importador manual queda).
3. **Roles y permisos** (sección 9) y página `/asesores/` de solo lectura con búsqueda y fichas — es lo que más usa el equipo.
4. **Lista de propiedades con filtros** (sección 1.1) para admin y gerente.
5. **Catálogo de características editable** (sección 4) y tipos de propiedad.
6. **Importador de EasyBroker por ID** (sección 10) — antes de reducir el plan de EasyBroker.
7. **Asistente de 2 pasos** con fotos y planos (secciones 1.2, 2 y 11).
8. Ubicación aproximada/exacta (1.3).

## 14a. Menú (decisión 2026-11-07)
En Propiedades quedan: lo diario (lista, Añadir, Leads, Verificación de precios), **Equipo** (pestañas Usuarios y Roles y permisos) y **Migración de EasyBroker** (pestañas Fotos, URLs y redirecciones, Pasar a propias; luego Importar por ID). Archivo `includes/menu.php`; las pantallas son las mismas, solo agrupadas. Al terminar el corte se puede quitar la entrada de migración.

## 14b. Página `/asesores/` (hecha, v1 sin probar)
`includes/asesores.php`: página privada (login, `noindex`, sin caché) con búsqueda por texto, estado, operación, tipo, **zonas múltiples**, **todas las características**, precio, recámaras, baños y m²; tarjetas con acciones (ver ficha, copiar enlace, PDF con/sin marca, WhatsApp, ubicación). Datos internos (dirección, desarrollo, dueño, teléfono, comisión, notas) solo con `bh_ver_privados`; acciones de ficha solo con `bh_enviar_fichas`. Quien tiene rol del equipo sin `edit_posts` entra aquí tras el login y no ve el wp-admin. Pendiente: asesor asignado, editar desde aquí, filtros por todas las características de la tabla nueva, y ocultar la comisión ajena al asesor.

## 15. Usuarios del equipo (hecho como herramienta, 2026-11-06)
`includes/roles.php` en el plugin (sin probar en Local aún):
- **Propiedades › Usuarios del equipo**: solo el administrador crea usuarios (nombre, correo, rol); se envía un correo con enlace para que cada persona **fije su propia contraseña** (vence en 3 días); acciones: reenviar enlace, cambiar rol, desactivar/activar (cierra sus sesiones). Se ve el último acceso. El registro abierto de WordPress queda apagado, y la pantalla no puede dar ni quitar el rol de administrador.
- **Propiedades › Roles y permisos**: matriz de casillas (permiso × rol) y «Crear rol» copiando otro. Roles base: **Gerente** y **Asesor**; Javier es el administrador. Permisos: ver propiedades, ver privados, mandar fichas, ver leads, editar, cambiar precio, publicar, catálogos, importar de EasyBroker.
- Si el correo no llega (Local no envía correo), la pantalla ofrece **«Ver enlace»** (válido 15 min) para mandarlo por WhatsApp. En el servidor sí hay que configurar el SMTP.
- Requiere que el **correo (SMTP) del sitio funcione** (viene en `bh-security.php`); probar el envío con un usuario de prueba.
- Los permisos ya quedan registrados pero **todavía no controlan** la lista, la ficha PDF ni `/asesores/`: eso se aplica al construir la página de asesores (paso 3 del orden).

## 16. Salida a producción (estado: todo en local, EasyBroker aún activo)
Se prueba todo en local; el cambio al servidor se hace al final, con el dominio apuntando al VPS ([servidor/FASE2-MIGRACION.md](../servidor/FASE2-MIGRACION.md)). Lista de cosas propias de esta migración:
1. Capturar las URLs de EasyBroker y descargar el CSV (mientras la API responda).
2. Fotos migradas y verificadas; `uploads/propiedades/` incluido en la copia al VPS.
3. Cambiar el dominio en la base: `wp search-replace` (URLs de `bh_images`, `bh_images_eb`, contenido) de `besthome.local` al dominio real; después guardar Ajustes › Enlaces permanentes.
4. Pasar todas las propiedades a `bh_source = propio`; apagar los crons de la sync (queda el importador manual).
5. Crear usuarios del equipo (el correo debe salir por el SMTP real) y revisar la matriz de permisos.
6. Cambiar los enlaces de anuncios con el CSV (Google Ads) y verificar las redirecciones que lleguen al 404.
7. Verificar que ninguna foto ni URL apunte a `assets.easybroker.com`, y el feed de anuncios.

## 17. Preguntas que quedan
1. ¿Quién es el gerente y qué no debe ver, además de la configuración? (Por defecto ve datos privados pero no cambia precio ni publica.)
2. ¿Pasamos ahora todas las propiedades a «propias» o esperamos a salir a producción?
3. ¿Sigo con el paso 3 (página `/asesores/` con búsqueda y fichas, que aplica los permisos)?
