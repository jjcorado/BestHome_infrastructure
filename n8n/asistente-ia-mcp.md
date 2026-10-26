# Asistente de IA (Claude / ChatGPT) vía MCP en n8n

Estado: **diseño (2026-10-26); prioridad 1**. Meta de Javier: poder **preguntar y registrar** datos del CRM y del catálogo conversando con Claude (y después por WhatsApp). API de catálogo: [../sitio-web/API-CATALOGO.md](../sitio-web/API-CATALOGO.md). Plan general: [plan-automatizaciones.md](plan-automatizaciones.md).

## Cómo funciona
n8n expone un **servidor MCP** (nodo MCP Server Trigger) con herramientas que llaman a la API REST de EspoCRM y a la API de catálogo de WordPress. Claude se conecta a esa dirección como conector personalizado (en el celular y en el escritorio, según el plan; verificar disponibilidad en la cuenta). Sin Evolution ni WhatsApp: la conversación ocurre en la app de Claude. Un canal de WhatsApp para el mismo asistente es una fase posterior.

## Herramientas (empezar con pocas)
**Lectura (primero):** `buscar_contacto` (nombre, teléfono o correo), `ver_busqueda` (etapa, asesor, última actividad, recorridos y visitas, cierre), `mis_pendientes` (tareas de hoy y búsquedas sin movimiento), `buscar_propiedades` y `ver_propiedad` (catálogo; incluye aviso de precio sin verificar).
**Escritura (después, con confirmación):** `registrar_nota`, `crear_tarea`, `crear_recorrido` (con lista de IDs), `marcar_reaccion` de una visita, `cambiar_etapa`. Cada escritura primero devuelve una vista previa («voy a hacer X») y solo ejecuta cuando el usuario confirma.

## Seguridad y privacidad
- Dirección protegida (Cloudflare Access con token de servicio o llave en cabecera); no queda dentro de `/webhook/` público.
- **Usuario API de EspoCRM con rol limitado**: lectura de lo necesario y escritura solo en notas, tareas, recorridos, visitas y etapa; sin borrar ni exportar. Al inicio lo usan solo Javier y el gerente; luego, una llave por asesor para que cada quien vea lo suyo (los permisos viven en el CRM, no en las instrucciones del asistente).
- Nunca expone metas `bh_priv_*` (dueño, comisión de captación, notas) a asesores; solo a administradores si se decide.
- Los datos personales que se consulten pasan por el proveedor del asistente: usar solo cuentas de la empresa, y mencionarlo en el aviso de privacidad y en la política interna.
- Registro de cada llamada (quién, herramienta, fecha), sin guardar el contenido personal.

## Pasos
1. Crear el usuario API y probar las consultas de lectura con datos de prueba.
2. Montar el servidor MCP con las herramientas de lectura; probarlo desde Claude.
3. Agregar las de escritura con confirmación.
4. Llaves por asesor.
5. Canal de WhatsApp (con Evolution o la API oficial), más adelante.
