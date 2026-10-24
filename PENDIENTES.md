# PENDIENTES — Best Home Querétaro

Documento vivo; describe lo que **falta**. Lo que ya existe está en [sitio-web/ARQUITECTURA.md](sitio-web/ARQUITECTURA.md). Decisiones: [DECISIONES.md](DECISIONES.md). Ideas sin fecha: [FUTURO.md](FUTURO.md).

Última actualización de contenido: 2026-10-20 (migrado a este repo el 2026-10-23).

---

## Estado general

- **Fase 1 (local): terminada.** Seguridad (IP, SMTP, plugin protegido), outbox de leads `wp_bh_leads` con reintentos, atribución first/last, WhatsApp con `Ref: R-XXXXXX`, diseño (hero, modales, header, banda de contacto), aviso de privacidad enlazado, eventos de dataLayer, remarketing (view_item + feed alineados), precio por m², verificación de precios, notas internas, visibilidad (pública / privada / archivada), orden manual del catálogo (`bh_sort_date`).
- **Fase 2 (servidor y migración): pendiente.**
- **Fase 3 (n8n, CRM, conversiones offline): pendiente.**

---

## Antes de migrar (no es código)

- [ ] Revisar las 3 propiedades marcadas como archivadas (IDs 20, 21, 22) y ponerlas en "Pública" si fueron pruebas.
- [ ] Contenido del **Aviso de privacidad**: transferencia de datos a n8n y al CRM, cookies de atribución (`bh_attr`, GA, Meta), datos enviados a Google Ads y Meta (correo y teléfono con hash), periodo de retención de leads, cómo ejercer derechos ARCO. **Retención ya implementada en el código (2026-10-23):** WhatsApp sin confirmar 30 días; nombre, teléfono, correo y mensaje de formularios sincronizados 30 días tras `synced_at` (90 días si `failed`); el resto (referencia, atribución, propiedad, `crm_id`) se conserva sin datos personales. Al redactar el aviso, declarar esos plazos y subir `BH_PRIVACY_VERSION` en `leads.php`/`wp-config` (detalle en [sitio-web/LEADS.md](sitio-web/LEADS.md#retención-cron-diario-bh_eb_purge_leads)).
- [ ] Imágenes del hero: escritorio (~1920×1080) y móvil (~1080×1600), < 250 KB cada una.
- [ ] Personalizador: título y texto de apoyo del hero; mensaje de WhatsApp con `{operacion}`; textos de la banda de contacto.
- [ ] Cambiar la contraseña de administrador por una propia (el día de la migración).
- [ ] Definir `BH_EB_EDIT_URL` (URL de edición de EasyBroker con `%s` en lugar del ID).

---

## Fase 2 — Servidor y migración
Checklist completo en [servidor/FASE2-MIGRACION.md](servidor/FASE2-MIGRACION.md); arquitectura en [servidor/ARQUITECTURA-VPS.md](servidor/ARQUITECTURA-VPS.md); GTM en [sitio-web/GTM.md](sitio-web/GTM.md).

## Fase 3 — CRM, n8n y conversiones
- Elegir CRM: [crm/ELECCION-CRM.md](crm/ELECCION-CRM.md)
- Automatizaciones n8n: [n8n/README.md](n8n/README.md)
- Evolution API: [evolution-api/README.md](evolution-api/README.md)
- [ ] **Flujos n8n de leads** (el sitio ya está listo; falta n8n): formulario → [n8n/flujo-leads.md](n8n/flujo-leads.md); WhatsApp por `Ref:` → [n8n/whatsapp-ref.md](n8n/whatsapp-ref.md). Hasta que exista `BH_WEBHOOK_LEAD_URL`, los formularios esperan en `pending` (no caducan) y los de WhatsApp en `local` (se borran a los 30 días: no hace falta configurar nada para no perderlos, salvo que n8n tarde más de ese plazo).
- [ ] **Analítica (BigQuery + Looker Studio)**, diseño en [n8n/bigquery-looker.md](n8n/bigquery-looker.md) y campos en [crm/campos-analitica.md](crm/campos-analitica.md). **Ahora:** proyecto Google Cloud con facturación + alerta de presupuesto; exportación diaria GA4 y exportación masiva de Search Console (no rellenan histórico); crear en EspoCRM los campos/entidades de analítica (incl. «Propiedad mostrada») antes de capturar clientes; confirmar en el sitio términos `amenidad`, fraccionamiento y condición. **Después:** conector Google Ads, flujo n8n CRM → BigQuery, tablero Looker. Añadir al aviso de privacidad el análisis con datos seudonimizados.
- [ ] **Privacidad de contactos** (revisar con asesor legal): contactos heredados de EasyBroker/CRMs anteriores (enviar aviso de privacidad en el primer contacto, marcar `origen = heredado`, depurar inactivos); si Best Home es actividad vulnerable por intermediar inmuebles (ley antilavado), las obligaciones de identificación y conservación (5 años) limitan la cancelación de datos de cierres; procedimiento ARCO, baja de marketing y bloqueo legal ya diseñados en [crm/privacidad-arco.md](crm/privacidad-arco.md): falta crear en EspoCRM `no_contactar_marketing`, `bloqueo_legal`, equipo solo-administrador y permisos sin borrado para asesores, y el flujo n8n de anonimización. **Antes de cualquier correo automático**: la baja de marketing debe estar activa.
- [ ] **Recorridos** ([crm/recorridos.md](crm/recorridos.md)): decidido: todo dentro de EspoCRM (entidades Recorrido y Visita, reacción de 3 niveles, comentarios abiertos). Falta crearlas, el webhook y el flujo n8n que completa los datos de la casa por ID desde la API de catálogo (confirmar que `q` busque `bh_eb_id` o agregar filtro `eb_id`). Confirmar que el plugin impide el borrado definitivo de propiedades.

---

## Independencia de EasyBroker (sin fecha)

- [ ] **Migrar fotos antes de cancelar EasyBroker** (hoy se sirven desde `assets.easybroker.com`; al cancelar desaparecen del sitio y del feed). Estructura: `uploads/propiedades/{ID}/`, adjuntos de WordPress, original máx. 2560 px.
- [ ] Generador de ID propio `BH-XX0000` (aleatorio, único, sin I ni O), guardado en `bh_eb_id`.
- [ ] Interfaz de captura de propiedades (wp-admin u otra), con `bh_source = 'propio'`.
- [ ] Precio editable en la tabla de Verificación de precios cuando el sitio sea la fuente.
- [ ] Banderas de publicación por portal (Inmuebles24, etc.).

Contrato de datos: [sitio-web/CONTRATO-DATOS.md](sitio-web/CONTRATO-DATOS.md).
