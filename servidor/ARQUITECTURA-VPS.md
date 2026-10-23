# VPS Best Home: Cloudflare, Docker y CloudPanel (recomendación)

Estado real hoy: [ESTADO-ACTUAL.md](ESTADO-ACTUAL.md). Checklist de migración: [FASE2-MIGRACION.md](FASE2-MIGRACION.md). Evolution API: [../evolution-api/README.md](../evolution-api/README.md).

Base: Hostinger VPS KVM 2 (2 vCPU, 8 GB), Ubuntu 24.04, CloudPanel con WordPress, y Docker a instalar con n8n, CRM (EspoCRM es el candidato; en `prompt-base-besthome.md` aún figura "sin elegir") y Evolution API. Las cifras de RAM son estimaciones, no mediciones.

## 1. Cloudflare: proxy vs túnel

Son dos cosas distintas y a menudo se confunden.

- **Proxy (nube naranja):** el DNS apunta a Cloudflare y este reenvía el tráfico a tu VPS. Da WAF, reglas anti-bots, rate limiting, CDN para estáticos, Turnstile y ocultar la IP. Es lo que realmente bloquea bots y tráfico basura.
- **Túnel (`cloudflared`):** un servicio en el VPS que abre una conexión *saliente* hacia Cloudflare. Con eso no necesitas abrir los puertos 80/443 al mundo; todo entra por el túnel. Es una forma de entrega, no una protección extra por sí misma.
- **Cloudflare Access (Zero Trust, gratis hasta 50 usuarios):** pide login (correo/Google) antes de llegar a una URL. Se usa sobre el túnel o el proxy.

**Qué conviene aquí**
- WordPress público: proxy naranja + firewall que solo acepte 80/443 desde IPs de Cloudflare. No necesitas túnel y evitas un punto de falla extra.
- Paneles internos (n8n editor, CRM admin, Evolution manager, CloudPanel :8443): túnel + Access, o al menos Access sobre el proxy. Aquí sí vale la pena.
- **Webhooks:** Evolution, EasyBroker, Meta y Google Ads deben poder llamar a n8n sin login. Excluye las rutas `/webhook/*` de Access y de Bot Fight Mode (o protégelas con token/firma). Si activas Bot Fight en esos hostnames, se romperán.

**Convivencia con CloudPanel y el cache**
- CloudPanel (Nginx) sigue en 80/443; con proxy usa certificado Origin de Cloudflare o Let's Encrypt, SSL modo *Full (strict)*.
- IP real: configura Nginx con `real_ip_header CF-Connecting-IP` y las IPs de Cloudflare. **Importante:** el plugin/tema usa rate limit por IP (p. ej. 5 s en `/attr` y límites de leads). Sin esto, todos los visitantes parecerán tener la IP de Cloudflare y el límite los bloqueará a todos juntos.
- Cache: Cloudflare no cachea HTML por defecto; deja eso al cache de CloudPanel (FastCGI/Varnish) + Redis Object Cache. En Cloudflare cachea solo estáticos (`/wp-content`, `/assets`). Nunca caches `/wp-admin`, `/wp-json/besthome/*` (leads, atribución con cookie `bh_attr`) ni respuestas con `Set-Cookie`.
- Reglas útiles gratis: challenge a `/wp-login.php`, bloquear `xmlrpc.php`, rate limit a `/wp-json/besthome/v1/lead`, Turnstile en formularios. Googlebot y bots verificados pasan.

## 2. Arquitectura recomendada

```
Internet -> Cloudflare (proxy/WAF) -> VPS :443 (solo IPs Cloudflare)
                                       CloudPanel Nginx
                                        |- besthomeqro.com      -> PHP-FPM (WordPress)
                                        |- n8n.besthomeqro.com   -> reverse proxy 127.0.0.1:5678
                                        |- crm.besthomeqro.com   -> reverse proxy 127.0.0.1:8080
                                        '- evo.besthomeqro.com   -> (idealmente NO público)
Docker (red interna bh_net): n8n, postgres, redis, espocrm, mariadb-crm, evolution-api
```

- **Un solo reverse proxy:** usa el que ya trae CloudPanel (tipo de sitio *Reverse Proxy*) para cada subdominio. No instales Traefik/Caddy aparte; competirían por el 80/443.
- **Puertos de Docker solo a localhost:** `ports: "127.0.0.1:5678:5678"`. Docker salta UFW al publicar puertos, así que `0.0.0.0:puerto` quedaría expuesto aunque UFW diga que no.
- **Red interna:** una red `bh_net` para todos. n8n habla con Evolution y el CRM por nombre de contenedor (`http://evolution:8080`), sin salir a internet. Evolution no necesita estar público: solo sale a WhatsApp y manda webhooks a n8n por la red interna. Si no lo expones, tampoco hay panel que atacar.
- **Bases de datos:** n8n con Postgres (no SQLite). Evolution con su Postgres y Redis (puede compartir el Postgres con instancias separadas). EspoCRM con su propio MariaDB en contenedor; no mezcles con el MySQL de WordPress.
- **Websockets:** el vhost de n8n necesita `Upgrade`/`Connection` y timeouts largos (se edita en el vhost de CloudPanel). Variables de n8n: `N8N_HOST`, `WEBHOOK_URL=https://n8n.besthomeqro.com/`, `N8N_PROXY_HOPS`, `N8N_ENCRYPTION_KEY` (guárdala fuera del servidor), poda de ejecuciones (`EXECUTIONS_DATA_PRUNE=true`).
- **UFW:** permitir 22 (solo tu IP si es fija, y SSH con llave, sin contraseña), 80/443 solo desde rangos de Cloudflare, 8443 de CloudPanel solo tu IP o detrás de Access; resto denegado. Lista de IPs de Cloudflare actualizada con un cron.
- **fail2ban:** jail de `sshd` y de Nginx (login de WP, 4xx repetidos). Con Cloudflare delante, que lea la IP real, o banearás a Cloudflare.
- **Backups (lo más descuidado):** snapshot semanal de Hostinger + backup diario de CloudPanel (sitio y MySQL) + `pg_dump`/`mysqldump` nocturnos de los contenedores y copia de volúmenes, enviados fuera del VPS (Cloudflare R2/Backblaze con `restic` o `rclone`). Prueba una restauración al menos una vez.
- **Docker:** imágenes con versión fija (no `latest`), rotación de logs (`max-size: 10m`), `restart: unless-stopped`, archivo `.env` fuera del repositorio.
- **Aviso sobre Evolution API:** conecta WhatsApp de forma no oficial; hay riesgo de bloqueo del número. Usa un número dedicado o considera la API oficial de WhatsApp Business para lo crítico.

## 3. Reparto de recursos: ¿alcanzan 8 GB?

Sí, con holgura moderada. Estimación en reposo:

| Componente | RAM aprox. |
|---|---|
| Ubuntu + CloudPanel + Nginx + fail2ban | 0.7 GB |
| WordPress: PHP-FPM, MySQL, Redis | 1.5–2.5 GB |
| n8n | 0.4–1 GB (picos con flujos pesados) |
| Postgres compartido | 0.3–0.5 GB |
| EspoCRM + MariaDB | 0.6–1 GB |
| Evolution API + Redis | 0.4–0.8 GB |
| **Total** | **~4–6.5 GB** |

- El cuello de botella son los **2 núcleos**, no la RAM. Los picos de n8n o de un crawl pueden quitarle CPU a PHP.
- Pon `mem_limit` y `cpus` a los contenedores (p. ej. n8n 1 GB / 1 CPU, CRM 1 GB, Evolution 768 MB) y baja la prioridad con `cpu_shares`; WordPress debe ganar siempre.
- Ajusta PHP-FPM (`pm.max_children` ~8–10 según el peso real de cada worker) y `innodb_buffer_pool_size` ~512 MB–1 GB.
- Agrega 2–4 GB de swap como colchón. Si el swap se usa de forma constante, sube a KVM 4 (4 vCPU / 16 GB) antes de empezar a optimizar a ciegas.
- Mide con `docker stats` y `htop` durante una semana antes de decidir.

## 4. CloudPanel vs alternativa amigable con Docker

La latencia que te preocupaba solo aparecería si **WordPress corriera dentro de Docker**. Aquí WordPress sigue nativo en CloudPanel y Docker queda al lado, así que **no agrega latencia al sitio**.

- **Seguir con CloudPanel + Docker lateral (recomendado):** CloudPanel pesa ~150 MB, trae Nginx afinado para WordPress, FastCGI cache, Redis, backups y SSL. Cero migración. Lo único manual es el vhost de cada contenedor.
- **Coolify/Dokploy (panel orientado a Docker):** más cómodos para contenedores, pero toman el 80/443 con Traefik, consumen ~0.5–1 GB extra, y tendrías que correr WordPress en contenedor y afinar tú el cache. El costo de rendimiento de Docker en sí es pequeño (~1–3 % CPU), el costo real es la afinación y el riesgo de migrar.
- Si luego quieres una interfaz para los contenedores, agrega **Portainer o Dockge** (ligeros) sin cambiar de panel.

Conclusión: no cambies de panel. La prioridad es velocidad de WordPress y esta combinación la protege.

## 5. Qué cambiar o mejorar, en orden

1. Confirmar el estado real: qué corre hoy (`cloudflared` o solo DNS con proxy), si el sitio ya está detrás de la nube naranja, y cuánta RAM/swap usa.
2. Cloudflare: SSL *Full (strict)*, `real_ip` en Nginx, reglas de cache (solo estáticos), WAF/rate limit/Turnstile. Verificar que los rate limits del plugin vean la IP real.
3. Firewall: UFW (22, 80/443 solo Cloudflare, 8443 restringido), SSH solo con llave, fail2ban.
4. Backups fuera del VPS y una restauración de prueba **antes** de montar Docker.
5. Instalar Docker + Compose, red `bh_net`, límites de memoria, rotación de logs.
6. Levantar n8n (con Postgres) en `n8n.besthomeqro.com` vía reverse proxy de CloudPanel + Access en el editor, `/webhook/*` abierto.
7. Levantar Evolution API solo en red interna (o con IP/Access restringido); probar con un número dedicado.
8. Elegir y levantar el CRM (EspoCRM) al final, cuando el flujo n8n ↔ leads ya funcione (endpoints `ref`/atribución de la Fase 3).
9. Monitoreo ligero (UptimeRobot + alerta de disco/RAM) y revisión a las 2 semanas para decidir si hace falta KVM 4.

## 6. Acuerdos finales (2026-10-23)
- **Dominios separados:** `n8n.` 100 % tras Cloudflare Access (solo Javier). `api.` público, apunta al mismo n8n pero Nginx solo deja pasar `/webhook/`, con rate limit en el WAF y un secreto en cada webhook. Sustituye la idea de una excepción de Access para `/webhook/*`.
- **Evolution Manager** en `evo.` tras Access con API key; n8n le habla por la red interna de Docker.
- **CPU:** Evolution con `cpus: '0.5'` y `cpu_shares` bajo; WordPress debe ganar siempre.
- **Tras instalar Docker**, revisar que el sitio siga abriendo (Docker modifica iptables).
- **El túnel de Cloudflare no es obligatorio:** con el proxy (nube naranja) hay WAF y protección. Cloudflare exige el dominio real (no sirve `nip.io`).
- Orden: snapshot, Cloudflare proxy, `real_ip` en Nginx, UFW/fail2ban/SSH con llave, backups externos, Docker con límites, reverse proxies en CloudPanel, Access en los subdominios admin.
