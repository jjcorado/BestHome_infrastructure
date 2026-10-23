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
- [ ] Contenido del **Aviso de privacidad**: transferencia de datos a n8n y al CRM, cookies de atribución (`bh_attr`, GA, Meta), datos enviados a Google Ads y Meta (correo y teléfono con hash), periodo de retención de leads, cómo ejercer derechos ARCO.
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

---

## Independencia de EasyBroker (sin fecha)

- [ ] **Migrar fotos antes de cancelar EasyBroker** (hoy se sirven desde `assets.easybroker.com`; al cancelar desaparecen del sitio y del feed). Estructura: `uploads/propiedades/{ID}/`, adjuntos de WordPress, original máx. 2560 px.
- [ ] Generador de ID propio `BH-XX0000` (aleatorio, único, sin I ni O), guardado en `bh_eb_id`.
- [ ] Interfaz de captura de propiedades (wp-admin u otra), con `bh_source = 'propio'`.
- [ ] Precio editable en la tabla de Verificación de precios cuando el sitio sea la fuente.
- [ ] Banderas de publicación por portal (Inmuebles24, etc.).

Contrato de datos: [sitio-web/CONTRATO-DATOS.md](sitio-web/CONTRATO-DATOS.md).
