# Evolution API (WhatsApp)

Estado: **por instalar** (al final, después del CRM) (Docker, red interna `bh_net`, sin exponerla al público). Detalle de red, recursos y orden de despliegue: [../servidor/ARQUITECTURA-VPS.md](../servidor/ARQUITECTURA-VPS.md).

- Conecta WhatsApp de forma **no oficial**: hay riesgo de bloqueo del número. Usar un número dedicado, o la API oficial de WhatsApp Business (Cloud API) para lo crítico.
- Evolution no necesita estar público: solo sale a WhatsApp y manda webhooks a n8n por la red interna.
- Uso previsto: ligar el chat con su referencia `R-XXXXXX` ([../n8n/whatsapp-ref.md](../n8n/whatsapp-ref.md)).
- Pendiente de decidir: Evolution vs Cloud API oficial; coexistencia con la app de WhatsApp Business en el mismo número.

## Decisiones (2026-10-23)
- Uso inicial: **solo lectura** de mensajes (sin envíos ni difusiones). Baja el riesgo de baneo y de picos de CPU. Evolution v2 usa Baileys (WebSocket), no navegadores; el pico ocurre al emparejar o sincronizar historial.
- Límites: `cpus: '0.5'` (0.75 si el arranque es lento), `cpu_shares` bajo, puertos solo a `127.0.0.1`; Manager en `evo.` tras Cloudflare Access.
- Evitar pérdida de mensajes: confirmar que guarde mensajes en su Postgres, que el webhook de n8n responda de inmediato y guarde el JSON crudo antes de procesar, y una tarea de n8n que rellene huecos contra Evolution.
- Alternativa: GoWa (Go + whatsmeow; más ligera, pero sin base de mensajes propia y con menos mantenedores). Solo con número de prueba y en paralelo.
- Futuro: Cloud API oficial con *coexistence* vía BSP (renta fija; flujo extra `smb_message_echoes` para lo escrito desde el celular).
