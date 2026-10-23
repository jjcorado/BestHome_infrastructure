# Decisiones tomadas (no reabrir sin motivo)

- `lead_generado` es el **único** evento de conversión; se distingue con `lead_type`. Los eventos de "inicio" son para audiencias, no conversiones.
- WordPress es la fuente de verdad de los leads (outbox). El correo es solo alerta, no transporte (salvo fallo del INSERT).
- Ventas, precio de cierre y comisiones de cada operación viven en el CRM, no en el sitio.
- Visibilidad única: pública / privada / archivada. Las archivadas siguen abriendo con aviso y similares (noindex).
- Prefijos: propiedades `EB-` / `BH-`; leads `R-`.
- Datos internos (ubicación o modelo, dueño, comisión, notas) se mantienen en WordPress, no en las notas de EasyBroker.
- Rangos de precio del feed clasifican propiedades (uno por propiedad); la segmentación de personas por presupuesto se hace con audiencias.
- Meta: anuncios a público en México sin categoría especial; campañas a público en EE. UU./Canadá requieren categoría de vivienda (separarlas).
