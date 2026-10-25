# Estado real del VPS (2026-10-23)

Lo que **ya está hecho** en el servidor. La arquitectura objetivo está en [ARQUITECTURA-VPS.md](ARQUITECTURA-VPS.md); lo que falta, en [../PENDIENTES.md](../PENDIENTES.md). No se guardan contraseñas ni llaves aquí: viven en `~/n8n/.env`, `~/espocrm/.env` y el gestor de contraseñas de Javier.

## Servidor base
- Hostinger VPS KVM 2, Ubuntu 24.04, CloudPanel (Nginx, PHP-FPM, MySQL/Percona). IP `2.25.72.182`.
- Kernel actualizado (6.8.0-147), sin reinicio pendiente.
- **UFW** activo, entrada denegada por defecto: abiertos 22, 80 y 443 (443/udp incluido). El 8443 de CloudPanel **está cerrado**; se entra solo por túnel SSH:
  `ssh -L 8443:127.0.0.1:8443 javier@2.25.72.182` y luego `https://localhost:8443` (aceptar la advertencia del certificado).
- **Fail2Ban** vigilando SSH. `root` sin acceso SSH; acceso solo con llave. Usuario con sudo: `javier`.
- Hostinger: snapshot y respaldos semanales (están en la misma infraestructura; falta copia externa).
- CloudPanel protegido con contraseña propia. Sitio de prueba en `besthomeqro.2.25.72.182.nip.io`; el sitio definitivo será una **migración nueva** desde el local con el dominio real (no se invierte esfuerzo en el de prueba).
- Memoria tras instalar Docker: ~5.8 GB disponibles y 2 GB de swap; no hizo falta crear swap.

## Docker
- Docker y Compose instalados desde el repositorio oficial. Logs de contenedores limitados (`json-file`, 10 MB × 3). Todos los puertos publicados van solo a `127.0.0.1` (Docker se salta UFW).
- Se entra a n8n y EspoCRM por túnel SSH mientras no exista el dominio real.

| Servicio | Versión (fija) | Carpeta | Puerto | Límites | Backup |
|---|---|---|---|---|---|
| n8n | 2.44.6 | `~/n8n` | `127.0.0.1:5678` | 1 GB / 1 CPU | diario 3:00 (`backup.sh`: `pg_dump` + volumen) |
| Postgres de n8n | 16 | `~/n8n` | interno | 512 MB | incluido arriba |
| EspoCRM (web) | 10.0.11 | `~/espocrm` | `127.0.0.1:8081` (el 8080 lo usa CloudPanel) | 768 MB | diario 3:10 |
| EspoCRM daemon | 10.0.11 | `~/espocrm` | interno | 256 MB | — |
| MariaDB del CRM | 11.4 | `~/espocrm` | interno | 512 MB | incluido arriba |

- Backups locales en `/home/javier/backups/{n8n,espocrm}`, 7 días de retención, por cron en `/etc/cron.d/`. **Falta la copia fuera del VPS.**
- n8n: la clave `N8N_ENCRYPTION_KEY` se guarda en el gestor de contraseñas (si se pierde no se pueden leer las credenciales guardadas).
- EspoCRM 10: se montan solo los volúmenes `data`, `custom` y `client-custom`. **No funciona montar todo `/var/www/html`** en la v10: la instalación queda a medias y el daemon se reinicia sin escribir log, porque su script sale si el CRM no está instalado.
- Para cambiar de versión: respaldar, cambiar la etiqueta en `docker-compose.yml`, `docker compose up -d` y probar.

## Observaciones
- MySQL (3306), FTP (21) y correo escuchan en `0.0.0.0` pero UFW los bloquea desde fuera. Ver pendientes: dejar MySQL solo en `127.0.0.1` y apagar FTP.
