# 📚 Base de Conocimientos — SigNoz + OpenTelemetry

Documentación técnica para la implementación de observabilidad completa con **SigNoz** y **OpenTelemetry**: métricas de infraestructura, trazas de aplicación y logs correlacionados.

---

## 🗂️ Contenido

| # | Sección | Descripción |
|---|---------|-------------|
| 1 | [Instalación de SigNoz en Docker](./01-Instalacion-SigNoz/README.md) | Despliegue de SigNoz con Docker Compose, configuración inicial y acceso a la UI |
| 2 | [Instalación del Agente OpenTelemetry](./02-Instalacion-OpenTelemetry/README.md) | Instalación del OTel Collector en Windows, Linux y AIX |
| 3 | [Métricas de Infraestructura](./03-Metricas-Infraestructura/README.md) | Configuración del hostmetrics receiver para CPU, memoria, disco y red |
| 4 | [Instrumentación — FastAPI + Python](./04-Instrumentacion-FastAPI/README.md) | Auto-instrumentación de FastAPI y MongoDB con OTel SDK para Python |
| 5 | [Instrumentación — React Frontend](./05-Instrumentacion-React/README.md) | Instrumentación del frontend React con OTel Web SDK |
| 6 | [Troubleshooting](./06-Troubleshooting/README.md) | Errores frecuentes, diagnóstico y soluciones |

---

## 🏗️ Arquitectura general

```
┌─────────────────────────────────────────────────────────────────┐
│                         APLICACIÓN                              │
│                                                                 │
│   React (browser)   FastAPI (servidor)   Host OS (sistema)     │
│        │                   │                    │               │
│   OTel Web SDK        OTel Python SDK     OTel Collector        │
│        │                   │                    │               │
│        └──── OTLP/HTTP ────┴──── OTLP/gRPC ────┘               │
│                             │                                   │
└─────────────────────────────┼───────────────────────────────────┘
                              ▼
                   ┌─────────────────┐
                   │   OTel Collector │  (incluido en SigNoz)
                   │  :4317 (gRPC)   │
                   │  :4318 (HTTP)   │
                   └────────┬────────┘
                            ▼
                   ┌─────────────────┐
                   │     SigNoz      │
                   │   :3301 (UI)    │
                   │  ClickHouse DB  │
                   └─────────────────┘
```

## 🔌 Puertos importantes

| Puerto | Protocolo | Uso |
|--------|-----------|-----|
| `3301` | HTTP | SigNoz UI |
| `4317` | gRPC | OTLP — backends, agentes (no browsers) |
| `4318` | HTTP | OTLP — browsers, clientes sin soporte gRPC |
| `8888` | HTTP | Métricas internas del Collector |

---

## ⚡ Quickstart

```bash
# 1. Clonar SigNoz y levantar
git clone https://github.com/SigNoz/signoz.git
cd signoz/deploy
docker compose -f docker/clickhouse-setup/docker-compose.yaml up -d

# 2. Abrir UI
# http://localhost:3301

# 3. Enviar primer dato de prueba
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[]}'
# Respuesta 200 = collector activo
```

---

## 📋 Requisitos previos

- Docker Engine 20.10+
- Docker Compose v2+
- 4 GB RAM mínimo para SigNoz
- Puertos `3301`, `4317`, `4318` disponibles

---

*Documentación generada en base a implementaciones reales. Última actualización: Noviembre 2026.*
