# Decisiones tomadas (no reabrir sin motivo)

- `lead_generado` es el **único** evento de conversión; se distingue con `lead_type`. Los eventos de "inicio" son para audiencias, no conversiones.
- WordPress es la fuente de verdad de los leads (outbox). El correo es solo alerta, no transporte (salvo fallo del INSERT).
- Ventas, precio de cierre y comisiones de cada operación viven en el CRM, no en el sitio.
- Visibilidad única: pública / privada / archivada. Las archivadas siguen abriendo con aviso y similares (noindex).
- Prefijos: propiedades `EB-` / `BH-`; leads `R-`.
- Datos internos (ubicación o modelo, dueño, comisión, notas) se mantienen en WordPress, no en las notas de EasyBroker.
- Rangos de precio del feed clasifican propiedades (uno por propiedad); la segmentación de personas por presupuesto se hace con audiencias.
- Meta: anuncios a público en México sin categoría especial; campañas a público en EE. UU./Canadá requieren categoría de vivienda (separarlas).
- Una propiedad nunca se borra: solo cambia de estado. Los `EB-` actuales son permanentes; las nuevas serán `BH-`. El inventario de referencia (visitas, cierres, reportes) es la base de WordPress, no EasyBroker.
- Recorridos: se registran dentro del CRM (Recorrido → Visita), reacción en 3 niveles y comentarios abiertos; los datos de la casa los completa n8n por ID.
- BigQuery guarda precio y comisión exactos y fechas exactas; la privacidad se apoya en no incluir identidad ni dirección y en el control de acceso. Al ejercer ARCO, se anonimiza la fila (conserva canal, precio, comisión, zona y mes). Cierres con operación se bloquean en el CRM, no se borran.
