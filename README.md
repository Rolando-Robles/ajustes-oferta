# Aviva-ofertas

## Scope

Aplicación web de una sola página para que un cliente de Aviva Crédito revise, ajuste o rechace una oferta de reestructura de crédito, con sincronización de estado en tiempo real contra HubSpot CRM (objeto Deal). El proyecto Firebase se llama `ajustes-oferta` (`.firebaserc`).

Arquitectura: frontend estático (Firebase Hosting) + backend serverless 
(Firebase Cloud Functions) que actúa como proxy autenticado hacia la API de HubSpot. El frontend nunca llama a HubSpot directamente; el token de HubSpot vive solo en el backend.

## Componentes

```
public/index.html    frontend: SPA de una sola página, sin build step
functions/index.js    backend: 4 Cloud Functions HTTP
firebase.json          config de hosting + functions
.firebaserc             project id: ajustes-oferta
```

No hay framework de frontend (React/Vue/etc). `index.html` contiene
HTML, CSS y JS inline en un solo archivo. No hay base de datos propia;
HubSpot es la única fuente de verdad de los datos del deal.

## Flujo funcional

1. El cliente recibe un link (WhatsApp o email) con query params:
   `?deal_id=<id>&utm_source=whatsapp|email`.
2. Al cargar, `index.html` llama a `GET /getDealData?deal_id=...`.
   Esa función consulta el deal en HubSpot (monto, periodos, pago por
   periodo, tasa semanal, nombre del contacto asociado, y si ya fue
   procesado vía la propiedad `ajuste_pantalla`).
3. Si `ajuste_pantalla` ya es `true`, se muestra la pantalla
   "ya procesado" y no se permite reprocesar.
4. Si no, se muestra la pantalla de oferta con tres caminos:
   - **Aceptar directo**: `POST /aceptarOferta` — cambia `dealstage`
     y marca `ajuste_pantalla = true`.
   - **Ajustar oferta**: el usuario mueve dos sliders (monto, plazos).
     `aplicarReglas()` (en el frontend) recalcula monto aprobado,
     plazos finales y pago semanal con reglas de negocio embebidas
     (topes de monto, extensión/reducción de plazos según dirección
     del cambio) y amortización francesa estándar. Al confirmar:
     `POST /ajustarOferta` — actualiza `amount`, `periodos`,
     `pago_por_periodo`, `dealstage`, y marca procesado.
   - **Rechazar**: `POST /rechazarOferta` — solo registra el
     `utm_source` de origen, sin cambiar el resto del deal.
5. Todas las funciones de escritura también guardan `utm_source`
   (mapeado a un valor legible: "WhatsApp Pantalla Ajuste" o "Email")
   para que HubSpot registre el canal de origen de la interacción.

## Backend (functions/index.js)

Cuatro Cloud Functions HTTP independientes (`onRequest`), cada una con CORS abierto (`origin: true` + headers manuales) porque se invocan desde el navegador del cliente sin autenticación de sesión:

| Función | Método | Propósito |
|---|---|---|
| `getDealData` | GET | Lee datos del deal + contacto asociado desde HubSpot |
| `aceptarOferta` | POST | Acepta la oferta original sin cambios |
| `ajustarOferta` | POST | Aplica monto/plazos ajustados por el cliente |
| `rechazarOferta` | POST | Registra rechazo y canal de origen |

Autenticación contra HubSpot: Bearer token leído de
`process.env.HUBSPOT_TOKEN` (variable de entorno de la función).

## Dependencias runtime

- `firebase-admin`, `firebase-functions` — runtime de Cloud Functions
- `axios` — cliente HTTP hacia la API de HubSpot
- `cors` — manejo de preflight/CORS



## Puntos a tener en cuenta

- Las reglas de negocio de reajuste (`aplicarReglas`) están implementadas en el cliente (JS del navegador), no en el backend. El backend confía en los valores finales que le manda el frontend para `ajustarOferta` sin recalcular ni validar límites del lado servidor.
- El `dealstage` de destino (`34528397`) está hardcodeado en las tres funciones de escritura.
