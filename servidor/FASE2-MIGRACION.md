# Fase 2: servidor y migración (checklist)

Pila decidida: Ubuntu 24.04 + CloudPanel (Nginx, PHP-FPM, MySQL) + n8n en Docker.

Arquitectura de referencia: [ARQUITECTURA-VPS.md](ARQUITECTURA-VPS.md). GTM: [../sitio-web/GTM.md](../sitio-web/GTM.md).

- [ ] Contratar KVM 2. SSH solo con llave, fail2ban, firewall: 22, y 80/443 solo desde rangos de Cloudflare más la IP pública del VPS.
- [ ] `/etc/hosts`: el dominio del sitio y `n8n.besthomeqro.com` resuelven a 127.0.0.1 (tráfico interno sin salir por Cloudflare).
- [ ] Instalar CloudPanel y crear el sitio WordPress; activar Redis Object Cache para que los transients por IP (`bh_v_*`, `bh_lead_*`, `bh_attr_*`) no inflen `wp_options`.
- [ ] Migrar archivos y base de datos; `wp search-replace` del dominio `.local` al final (respetando serializados).
- [ ] `BH_WEBHOOK_LEAD_URL` puede quedar vacía mientras se levanta n8n: los leads esperan en `pending` (`last_error='config'`) sin caducar y se envían solos al configurarla.
- [ ] `wp-config.php`: `WP_ENVIRONMENT_TYPE = 'production'`, `DISABLE_WP_CRON = true`, `BH_BEHIND_CF = true`, claves de Turnstile, EasyBroker y SMTP (`BH_SMTP_*`), `BH_WEBHOOK_LEAD_URL` (webhook temporal o Make hasta tener n8n), `BH_PRIVACY_VERSION`. Confirmar que **no** exista `BH_EB_STAGING`.
- [ ] Verificar que PHP tenga GD o Imagick (ficha PDF).
- [ ] Excluir `/wp-json/besthome/*` de la caché de página de Nginx.
- [ ] Cron del sistema con WP-CLI cada 3 min, como el usuario del sitio.
- [ ] Guardar enlaces permanentes; desmarcar "Disuadir a los motores de búsqueda"; sync completa manual.
- [ ] Vaciar la tabla de leads de prueba: `TRUNCATE wp_bh_leads` (y `wp_bh_price_checks` si tiene pruebas).
- [ ] SMTP autenticado (587) con el correo del dominio; SPF, DKIM y DMARC en el DNS.
- [ ] Respaldos automáticos cifrados a almacenamiento externo (base de datos con leads = datos personales).
- [ ] Probar: lead de contacto, lead de ficha, WhatsApp desde ficha/header/home/blog, PDF, filtros, búsqueda.
