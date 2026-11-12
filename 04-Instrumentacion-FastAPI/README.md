# 🐍 Instrumentación — FastAPI + MongoDB (Python)

Guía de instrumentación de un backend **FastAPI + Motor (MongoDB async)** con OpenTelemetry SDK para Python, enviando trazas, métricas y logs a SigNoz.

> Basada en la implementación real del proyecto **SinerCard** (gestión de tarjetas de crédito).

---

## Stack instrumentado

| Componente | Versión | Instrumentor |
|------------|---------|-------------|
| FastAPI | 0.115+ | `opentelemetry-instrumentation-fastapi` |
| Motor/PyMongo | 3.6+ | `opentelemetry-instrumentation-pymongo` |
| Python logging | stdlib | `opentelemetry-instrumentation-logging` |
| Runtime | Python 3.12 | — |
| Imagen Docker | `python:3.12-slim` | requiere `setuptools` explícito |

---

## Dependencias

Agregar a `requirements.txt`:

```
setuptools==75.3.0
grpcio==1.68.0
opentelemetry-sdk==1.27.0
opentelemetry-api==1.27.0
opentelemetry-exporter-otlp-proto-grpc==1.27.0
opentelemetry-instrumentation-fastapi==0.48b0
opentelemetry-instrumentation-pymongo==0.48b0
opentelemetry-instrumentation-logging==0.48b0
```

> **¿Por qué `setuptools` y `grpcio` explícitos?**
> La imagen `python:3.12-slim` no incluye `pkg_resources` (parte de setuptools) ni las libs nativas de gRPC. Sin ellos los instrumentors fallan con `ModuleNotFoundError: No module named 'pkg_resources'`.

---

## Archivo `app/core/otel.py`

```python
import logging
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.pymongo import PymongoInstrumentor
from opentelemetry.instrumentation.logging import LoggingInstrumentor

# Sin http:// para gRPC — solo host:puerto
OTEL_ENDPOINT = "192.168.1.195:4317"
SERVICE_NAME  = "mi-backend"


def setup_telemetry():
    """
    Inicializar providers de OTel.
    DEBE llamarse ANTES de crear la app FastAPI.
    """
    resource = Resource.create({
        "service.name":           SERVICE_NAME,
        "service.version":        "1.0.0",
        "deployment.environment": "production",
    })

    # --- Trazas ---
    tracer_provider = TracerProvider(resource=resource)
    tracer_provider.add_span_processor(
        BatchSpanProcessor(
            OTLPSpanExporter(endpoint=OTEL_ENDPOINT, insecure=True)
        )
    )
    trace.set_tracer_provider(tracer_provider)

    # --- Métricas ---
    metric_reader = PeriodicExportingMetricReader(
        OTLPMetricExporter(endpoint=OTEL_ENDPOINT, insecure=True),
        export_interval_millis=15_000,   # enviar cada 15 segundos
    )
    meter_provider = MeterProvider(resource=resource, metric_readers=[metric_reader])
    metrics.set_meter_provider(meter_provider)

    # --- Logs ---
    # Inyecta trace_id y span_id en cada línea de log de Python
    LoggingInstrumentor().instrument(set_logging_format=True)
    logging.basicConfig(level=logging.INFO)

    # --- MongoDB ---
    # Instrumentar antes de que Motor se conecte
    PymongoInstrumentor().instrument()


def instrument_app(app):
    """
    Instrumentar FastAPI.
    DEBE llamarse DESPUÉS de crear la app pero ANTES de que uvicorn arranque.
    """
    FastAPIInstrumentor.instrument_app(app)
```

---

## Orden correcto en `main.py`

⚠️ El orden importa. Seguirlo mal causa `RuntimeError: Cannot add middleware after an application has started`.

```python
from app.core.otel import setup_telemetry, instrument_app

# 1️⃣ Inicializar OTel ANTES de crear la app
setup_telemetry()

# 2️⃣ Crear la app
app = FastAPI(...)

# 3️⃣ Agregar middlewares propios
app.add_middleware(CORSMiddleware, ...)

# 4️⃣ Registrar routers
app.include_router(...)

# 5️⃣ Instrumentar FastAPI (agrega su middleware interno)
instrument_app(app)

# ✅ Ahora uvicorn puede arrancar
```

---

## Variables de entorno en `docker-compose.yml`

```yaml
services:
  backend:
    environment:
      OTEL_SERVICE_NAME: mi-backend
      OTEL_EXPORTER_OTLP_ENDPOINT: http://192.168.1.195:4317
      OTEL_EXPORTER_OTLP_INSECURE: "true"
      OTEL_PYTHON_LOG_CORRELATION: "true"
```

> El SDK lee estas variables automáticamente como fallback. Si cambias de servidor SigNoz, solo editas el compose.

---

## ¿Qué se ve en SigNoz?

### Trazas automáticas

Cada request HTTP genera automáticamente un span con spans hijos para cada query MongoDB:

```
GET /api/cards  (45ms)
  └── mongodb.find  creditcard_db.cards  (12ms)
  └── mongodb.find  creditcard_db.clients  (8ms)
```

### Atributos del span HTTP

| Atributo | Ejemplo |
|---------|---------|
| `http.method` | `GET` |
| `http.route` | `/api/cards/{id}` |
| `http.status_code` | `200` |
| `http.url` | `http://localhost:8000/api/cards/abc` |
| `net.peer.ip` | `172.18.0.5` |

### Atributos del span MongoDB

| Atributo | Ejemplo |
|---------|---------|
| `db.system` | `mongodb` |
| `db.name` | `creditcard_db` |
| `db.mongodb.collection` | `cards` |
| `db.operation` | `find` |

---

## Métricas de negocio personalizadas (opcional)

```python
from opentelemetry import metrics

meter = metrics.get_meter("sinercard")

# Contador de transacciones
tx_counter = meter.create_counter(
    "transactions.processed",
    description="Total de transacciones procesadas"
)

# Histogram de montos
tx_amount = meter.create_histogram(
    "transactions.amount_usd",
    description="Monto de transacciones en USD"
)

# Usar dentro de un route:
@router.post("/transactions")
async def create_transaction(tx: TransactionCreate):
    # ... lógica ...
    tx_counter.add(1, {
        "status": "approved",
        "card_type": card.card_type,
        "network": card.network
    })
    tx_amount.record(tx.amount, {
        "merchant_category": tx.merchant_category
    })
```

Estas métricas aparecen en SigNoz → **Metrics** con el prefijo `transactions.*`.

---

## Verificación rápida

```bash
# 1. Levantar el backend
docker compose up --build backend

# 2. Generar tráfico
curl http://localhost:8000/api/cards
curl http://localhost:8000/api/clients

# 3. En SigNoz (esperar ~30s)
# Services → debe aparecer "mi-backend"
# Click → Traces → ver spans de GET /api/cards con hijos MongoDB
# Logs → filtrar service.name=mi-backend → ver trace_id en cada línea
```
