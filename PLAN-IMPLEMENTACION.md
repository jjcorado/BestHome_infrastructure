# Plan de implementación (2026-10-26)

Qué hay que cambiar en concreto, en qué orden y quién. Diseño de respaldo: [crm/REVISION-INTEGRAL.md](crm/REVISION-INTEGRAL.md). Revisado contra el código local del plugin (`rest-catalogo.php`, `cpt.php`, `leads.php`) el 2026-10-26.

## A. Lo que el sitio ya tiene (no se toca)
- Leads con `ref_code`, atribución first/last, `ga_client_id`, retención y endpoints para n8n (`includes/leads.php`). `lead_generado` intacto.
- API de catálogo con llave y `visibilidad=todas` (`includes/rest-catalogo.php`).
- Amenidades como catálogo fijo (10 términos en `BH_EB_CPT::amenities()`), que será el mismo catálogo de características del CRM.
- Zona jerárquica (`zona` con hijos), fechas de archivado (`bh_archived_at`), `bh_priv_price_log` (texto privado).

## B. Cambios en el plugin (`besthome-eb-sync`), sin tocar `sync.php` ni `api.php`
| # | Cambio | Por qué | Archivo |
|---|---|---|---|
| B1 | Parámetro **`eb_id`** exacto en `GET /propiedades` (busca `bh_eb_id`, visibilidad `todas` por defecto con ese parámetro) | Hoy `q` es la búsqueda de texto de WordPress (título y contenido) y **no encuentra el ID** | `rest-catalogo.php` |
| B2 | Para llamadas con llave: devolver el **precio real** aunque esté oculto en el sitio, más `precio_visible` | Hoy devuelve `null` si el precio está oculto, y la Visita y el Cierre necesitan el precio | `rest-catalogo.php` |
| B3 | **Bloquear papelera y borrado** de propiedades (`pre_trash_post` / `pre_delete_post`), salvo borradores automáticos | Regla: las propiedades nunca se borran; hoy se pueden mandar a la papelera | `cpt.php` |
| B4 | **Historial de precio por evento**: tabla `{prefix}bh_price_history` (post, `eb_id`, anterior, nuevo, moneda, fecha, origen sync/manual) escrita al cambiar `bh_price`; endpoint con llave `GET /precios?desde=` para la reconciliación; aviso por webhook a n8n cuando exista | Medir efecto de bajas de precio; el log actual es texto privado | archivo nuevo `includes/price-history.php` |
| B5 | Meta **`bh_condition`** (nueva \| usada \| desarrollo) con caja en el editor; la sync nunca la escribe; incluida en la API y en `bh_prop()` | No existe hoy y es dimensión clave de los reportes | `cpt.php`, `rest-catalogo.php`, tema |
| B6 | **Fraccionamiento**: usar el nivel hijo de `zona` (p. ej. Zibatá › Jade) en lugar de un campo nuevo, y devolverlo separado en la API (`zona`, `fraccionamiento`) | Evita otra taxonomía. **Requiere confirmar** que los términos de zona se usan así | `rest-catalogo.php` |
| B7 | Después: endpoint de suscripción al boletín (doble confirmación vía n8n) | Fase de correo | nuevo |
Validación: `php -l` en cada archivo y commit en `jjcorado/besthome-eb-sync`. Tras B5/B6, guardar Ajustes › Enlaces permanentes solo si se tocan rewrites (no debería).

**Estado 2026-10-28:** B1–B5 hechos en el plugin (commit 701c951 en `besthome-eb-sync`, `php -l` ok). B2: el endpoint ya exigía llave `X-BH-Key`, así que el precio real sale siempre. B3 muestra aviso admin. B4 deja la tabla `bh_price_history` (se crea al entrar a wp-admin), `GET /precios?desde=` y la acción `bh_price_changed`. B5 se devuelve como `condicion`. Falta probar en el sitio. B6 y B7 pendientes.

## C. Cambios en el tema
- Aviso de privacidad (texto): finalidades secundarias de marketing, herramientas de IA de terceros, Google Cloud/BigQuery, proveedor de correo, plazos de retención, **medio para ejercer ARCO** (correo dedicado). Subir `BH_PRIVACY_VERSION` (hoy `'1'`). Requiere revisión legal.
- Mostrar `bh_condition` en la ficha solo si se decide (opcional).
- Después: formulario de suscripción al boletín (pie de página o blog).

## D. EspoCRM (continuar la configuración; detalle en [crm/campos-espocrm.md](crm/campos-espocrm.md))
**Estado EspoCRM al 2026-10-28 (según Javier):** hecho: instalación limpia (MariaDB, límites de memoria, demonio), roles y equipos (Asesor, administradores, Flujo en «propio»), Oportunidad→Búsqueda y Reunión→Recorrido, Búsqueda con 8 etapas (probabilidades 5/10/20/35/60/100/0/0, auditada), entidad Cierre con Propiedad, Tipo, Etapa, Precio, Descripción y relación muchos-a-uno con Búsqueda. Dicho pero sin confirmar: regla de etapas según Tipo, Cierres en el menú, permisos del Asesor, diseño de Lista, campos extra del Cierre. Pendiente: usuarios reales y prueba con un asesor, los puntos 1–5 y 7–8 de abajo (campos de Contacto, Búsqueda, Recorrido, Visita, vistas guardadas), fórmulas Búsqueda/Cierre y usuario API para n8n. Ojo: los campos extra del Cierre (comisión, apartado, enganche, notaría…) deben alinearse con el diseño simple (`ingreso_inmobiliaria` + `compartida_otra_inmobiliaria`, `fecha_cierre`).

1. **Contacto**: `suscrito_boletin`, `no_contactar_marketing` (solo lectura para asesor), fecha/canal de baja, `bloqueo_legal`, `fin_bloqueo`, `origen_contacto`.
2. **Búsqueda**: `ref_code`, `ref_codes_adicionales`, `canal_entrada`, `eb_id_origen`, `fecha_entrada`, `fecha_primera_respuesta` (fórmula al salir de «Nuevo»), UTM e IDs de clic (solo administradores), `tipos_inmueble`, `presupuesto_min`, `motivo_perdida` (obligatorio al pasar a Perdido). Activar auditoría de `etapa` y del asignado.
3. **Recorrido** (Reunión): `ids_a_visitar`, `comentario_general`; relación uno a muchos con Visita.
4. **Visita** (entidad nueva): `id_propiedad`, `hora_visita`, `vista`/`motivo_no_vista`, `calificacion` (4 niveles), `motivo_rechazo` (lógica dinámica: visible y obligatorio si «no le gustó»), `comentario`, campos copiados en un panel plegado.
5. **Cierre**: `fecha_cierre` (escrituración/firma), `dias_ciclo` (fórmula), `ingreso_inmobiliaria`, `compartida_otra_inmobiliaria`, `precio_lista`, copias de la casa, `escritura_recogida`, `resena_solicitada`, bloqueo.
6. **Roles**: permisos por campo; asesores sin borrar ni exportar; usuario API para n8n y otro, más limitado, para el asistente MCP; equipo solo-administrador para bloqueo legal.
7. **Correo**: SMTP de Brevo, lista de destinatarios «Boletín», baja nativa.
8. Vistas guardadas: «Nuevos sin responder», «Mis búsquedas sin movimiento», «Mis tareas de hoy».

## E. Google (Javier, en consolas, sin código)
Proyecto de Google Cloud con facturación, alerta de presupuesto y cuota diaria; vincular **GA4 → BigQuery (diario)** y **exportación masiva de Search Console**. No rellenan histórico: hacerlo cuanto antes.

## F. n8n (cuando exista el dominio y el proxy)
1. Ajustes de rendimiento (sin historial de éxitos, concurrencia baja).
2. **Asistente MCP solo lectura**.
3. Alta de contacto del sitio (`BH_WEBHOOK_LEAD_URL`).
4. Completar Visita/Cierre (requiere B1 y B2).
5. Espejo a BigQuery por evento (requiere E y B4).
6. Correos de servicio y ARCO.

## Orden recomendado
**Esta semana:** E (Google), decisiones de B5/B6, B1–B3 en el plugin, y seguir con D1–D5 en EspoCRM. **Siguiente:** D6–D8, piloto con un asesor, B4. **Después:** F en orden, C con el asesor legal, B7.
