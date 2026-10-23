# Decisiones tomadas (no reabrir sin motivo)

- `lead_generado` es el **único** evento de conversión; se distingue con `lead_type`. Los eventos de "inicio" son para audiencias, no conversiones.
- WordPress es la fuente de verdad de los leads (outbox). El correo es solo alerta, no transporte (salvo fallo del INSERT).
- Ventas, precio de cierre y comisiones de cada operación viven en el CRM, no en el sitio.
- Visibilidad única: pública / privada / archivada. Las archivadas siguen abriendo con aviso y similares (noindex).
- Prefijos: propiedades `EB-` / `BH-`; leads `R-`.
- Datos internos (ubicación o modelo, dueño, comisión, notas) se mantienen en WordPress, no en las notas de EasyBroker.
- Rangos de precio del feed clasifican propiedades (uno por propiedad); la segmentación de personas por presupuesto se hace con audiencias.
- Meta: anuncios a público en México sin categoría especial; campañas a público en EE. UU./Canadá requieren categoría de vivienda (separarlas).
- **Infraestructura (2026-10-23):** CloudPanel con Docker «al lado» (WordPress nativo, sin Coolify/Dokploy). Puertos de contenedores solo a `127.0.0.1`. CloudPanel (8443) solo por túnel SSH. Dominios separados: `n8n.` privado tras Access, `api.` público solo `/webhook/`, `evo.` tras Access.
- **WhatsApp:** Evolution API en **solo lectura** (costo $0) para capturar historial al CRM; los asesores contestan a mano. Migrar a Cloud API oficial con *coexistence* (renta fija de un BSP, confirmar costos) solo cuando se necesiten plantillas o bot de primer contacto. GoWa queda como alternativa a probar con un número de prueba. n8n orquesta y escribe en EspoCRM por su API REST en cualquier caso.
- **CRM:** EspoCRM autoalojado (instalado en el VPS).
