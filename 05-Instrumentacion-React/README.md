# ⚛️ Instrumentación — React Frontend (Web)

Guía de instrumentación de un frontend **React + Vite** con el OTel Web SDK, enviando trazas de navegación y llamadas HTTP a SigNoz.

> El frontend corre en el **browser**, por lo que usa HTTP (puerto `4318`) en lugar de gRPC (puerto `4317`).

---

## Diferencias clave vs backend

| Aspecto | Backend (Python) | Frontend (Browser) |
|---------|-----------------|-------------------|
| Protocolo | gRPC — puerto `4317` | HTTP — puerto `4318` |
| Endpoint | `192.168.1.195:4317` | `http://192.168.1.195:4318/v1/traces` |
| Lo que instrumenta | Requests HTTP entrantes, queries DB | `fetch`, XHR, carga de página |
| SDK | `opentelemetry-sdk` Python | `@opentelemetry/sdk-trace-web` npm |

---

## Dependencias npm

Desde la carpeta `frontend/`:

```bash
npm install \
  @opentelemetry/sdk-trace-web \
  @opentelemetry/context-zone \
  @opentelemetry/auto-instrumentations-web \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions
```

> ⚠️ El paquete se llama `@opentelemetry/sdk-trace-web`, **no** `@opentelemetry/sdk-web` (ese no existe y da error 404).
> `@opentelemetry/context-zone` es necesario para propagar el contexto de trazas en operaciones async del browser.

---

## Crear `src/otel.js`

```javascript
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web'
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-web'
import { ZoneContextManager } from '@opentelemetry/context-zone'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { Resource } from '@opentelemetry/resources'
import { SEMRESATTRS_SERVICE_NAME } from '@opentelemetry/semantic-conventions'
import { getWebAutoInstrumentations } from '@opentelemetry/auto-instrumentations-web'
import { registerInstrumentations } from '@opentelemetry/instrumentation'

const provider = new WebTracerProvider({
  resource: new Resource({
    [SEMRESATTRS_SERVICE_NAME]: 'sinercard-frontend',
  }),
})

provider.addSpanProcessor(
  new BatchSpanProcessor(
    new OTLPTraceExporter({
      // Puerto 4318 HTTP — browsers no soportan gRPC nativo
      url: 'http://192.168.1.195:4318/v1/traces',
    })
  )
)

// ZoneContextManager mantiene el contexto de traza entre callbacks async
provider.register({
  contextManager: new ZoneContextManager(),
})

registerInstrumentations({
  instrumentations: [
    getWebAutoInstrumentations({
      // Instrumenta todos los fetch() automáticamente
      '@opentelemetry/instrumentation-fetch': {
        enabled: true,
        propagateTraceHeaderCorsUrls: [/.*/],  // propaga traceparent header a la API
      },
      // Instrumenta XMLHttpRequest (axios lo usa internamente)
      '@opentelemetry/instrumentation-xml-http-request': {
        enabled: true,
        propagateTraceHeaderCorsUrls: [/.*/],
      },
      // Instrumenta la carga inicial de la página
      '@opentelemetry/instrumentation-document-load': {
        enabled: true,
      },
    }),
  ],
})
```

---

## Activar en `src/main.jsx`

Agregar como **primera línea** antes de cualquier import de React:

```jsx
import './otel.js'   // ← PRIMERA LÍNEA — antes de React y todo lo demás

import React from 'react'
import ReactDOM from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import App from './App'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
)
```

---

## Rebuild del frontend

```bash
# Si usas Docker
docker compose up --build frontend

# Si corres en local
npm run build
npm run dev
```

---

## Verificar CORS en SigNoz

El browser enviará los spans desde el dominio de la app (ej: `http://192.168.1.196:3000`) hacia SigNoz (`http://192.168.1.195:4318`). Verificar que CORS esté activo:

```bash
curl -v -X OPTIONS http://192.168.1.195:4318/v1/traces \
  -H "Origin: http://192.168.1.196:3000" \
  -H "Access-Control-Request-Method: POST"
```

La respuesta debe incluir:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: POST
```

Si hay error de CORS, agregar en el OTel Collector de SigNoz (`otel-collector-config.yaml`):

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins:
            - "http://*"
            - "https://*"
```

---

## ¿Qué se ve en SigNoz?

### Trazas automáticas del browser

```
document.load  (850ms)
  └── fetch GET /api/cards  (45ms)
  └── fetch GET /api/clients  (38ms)
  └── fetch GET /api/cards/stats  (22ms)
```

### Trazas distribuidas (browser → backend → MongoDB)

Con `propagateTraceHeaderCorsUrls` configurado, el header `traceparent` viaja en cada `fetch`. SigNoz une la traza completa:

```
Browser: fetch GET /api/cards  (45ms)
  └── Backend: GET /api/cards  (32ms)
        └── MongoDB: find creditcard_db.cards  (12ms)
```

---

## Verificación en SigNoz

1. Navegar por la app en el browser
2. Esperar ~30 segundos
3. **SigNoz → Services** → debe aparecer `sinercard-frontend`
4. **Traces** → filtrar por `service.name = sinercard-frontend`
5. Ver spans de `document.load` y los `fetch` a la API
