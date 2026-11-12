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
| SDK | `opentelemetry-sdk` Python | `@opentelemetry/sdk-web` npm |

---

## Dependencias npm

Desde la carpeta `frontend/`:

```bash
npm install \
  @opentelemetry/sdk-web \
  @opentelemetry/auto-instrumentations-web \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions
```

---

## Crear `src/otel.js`

```javascript
import { WebTracerProvider } from '@opentelemetry/sdk-web'
import { BatchSpanProcessor } from '@opentelemetry/sdk-web'
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

provider.register()

registerInstrumentations({
  instrumentations: [
    getWebAutoInstrumentations({
      // Instrumenta todos los fetch() y axios calls automáticamente
      '@opentelemetry/instrumentation-fetch': {
        enabled: true,
        propagateTraceHeaderCorsUrls: [/.*/],  // propagar headers a la API
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

El browser enviará los spans desde el dominio de la app (ej: `http://192.168.1.196:3000`) hacia SigNoz (`http://192.168.1.195:4318`). Si hay problemas de CORS, SigNoz necesita tener configurado el header `Access-Control-Allow-Origin`.

Verificar que el puerto esté abierto y con CORS activo:

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

Si no responde o da error de CORS, configurar en el OTel Collector de SigNoz (`otel-collector-config.yaml`):

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

### Atributos del span fetch

| Atributo | Ejemplo |
|---------|---------|
| `http.method` | `GET` |
| `http.url` | `http://192.168.1.196:3000/api/cards` |
| `http.status_code` | `200` |
| `http.response_content_length` | `1842` |

### Trazas distribuidas (backend + frontend juntos)

Cuando el frontend hace un `fetch` con `propagateTraceHeaderCorsUrls` configurado, los headers `traceparent` se inyectan en el request HTTP. Si el backend también está instrumentado, **SigNoz mostrará la traza completa** de principio a fin:

```
Browser: fetch GET /api/cards  (45ms)
  └── Backend: GET /api/cards  (32ms)
        └── MongoDB: find creditcard_db.cards  (12ms)
```

---

## Verificación en SigNoz

1. Abrir la app en el browser y navegar por varias páginas
2. Esperar ~30 segundos
3. **SigNoz → Services** → debe aparecer `sinercard-frontend`
4. **Traces** → filtrar por `service.name = sinercard-frontend`
5. Ver spans de `document.load` y los `fetch` a la API
