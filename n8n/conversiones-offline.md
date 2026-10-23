# Conversiones offline (Google Ads y Meta)

Lo de más valor a mediano plazo. Estado: **pendiente**. Depende del CRM ([../crm/ELECCION-CRM.md](../crm/ELECCION-CRM.md)).

- [ ] Al avanzar una oportunidad en el CRM (cita, visita, venta): n8n envía la conversión a Google Ads (con `gclid`/`gbraid`/`wbraid`) y a Meta Conversions API (con `_fbc`/`_fbp`), usando `lead_id` como ID de evento.
