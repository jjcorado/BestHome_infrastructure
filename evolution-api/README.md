# Evolution API (WhatsApp)

Estado: **por instalar** (Docker, red interna `bh_net`, sin exponerla al público). Detalle de red, recursos y orden de despliegue: [../servidor/ARQUITECTURA-VPS.md](../servidor/ARQUITECTURA-VPS.md).

- Conecta WhatsApp de forma **no oficial**: hay riesgo de bloqueo del número. Usar un número dedicado, o la API oficial de WhatsApp Business (Cloud API) para lo crítico.
- Evolution no necesita estar público: solo sale a WhatsApp y manda webhooks a n8n por la red interna.
- Uso previsto: ligar el chat con su referencia `R-XXXXXX` ([../n8n/whatsapp-ref.md](../n8n/whatsapp-ref.md)).
- Pendiente de decidir: Evolution vs Cloud API oficial; coexistencia con la app de WhatsApp Business en el mismo número.
