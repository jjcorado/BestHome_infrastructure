# Configuración de EspoCRM

Estado al 2026-10-28 (EspoCRM 10.0.11 en el VPS, solo por túnel SSH, backup diario a las 3:10). Infraestructura: [../servidor/ESTADO-ACTUAL.md](../servidor/ESTADO-ACTUAL.md).

## Hecho
- Instalación limpia con MariaDB, límites de memoria y demonio.
- Roles y equipos: Asesor y administradores; Flujo en «propio»; entidades que no se usan, desactivadas.
- Oportunidad renombrada a **Búsqueda**; Reunión a **Recorrido**.
- Búsqueda con 8 etapas (Nuevo por defecto, auditada). Probabilidades: 5 / 10 / 20 / 35 / 60 / 100 / 0 / 0.
- Entidad **Cierre** (Base Plus, con Flujo): Propiedad, Tipo (Venta/Renta), Etapa, Precio, Descripción y relación muchos-a-uno con Búsqueda. Diseño de Detalle armado.

## Dicho pero por confirmar
- Regla de opciones de Etapa según Tipo, Cierres en el menú, permisos del rol Asesor, diseño de Lista.
- Campos extra del Cierre (comisión, apartado, enganche, fechas, notaría, motivo de caída, datos de renta) y sus reglas.

## Pendiente
- Usuarios reales (asesores, gerente, cuenta del dueño) y prueba con un asesor.
- Campos de Recorrido, vistas guardadas, fórmulas de sincronía Búsqueda/Cierre, usuario API para n8n.
- Campos de atribución del lead (ver [README.md](README.md)).
- Después: flujos de n8n, dominio y Cloudflare, backups fuera del servidor.
