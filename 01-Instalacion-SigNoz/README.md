# 🚀 Instalación de SigNoz en Docker

SigNoz es una plataforma de observabilidad open-source que unifica trazas, métricas y logs en una sola UI. Utiliza ClickHouse como base de datos columnar para almacenamiento eficiente.

---

## Requisitos previos

| Componente | Versión mínima |
|------------|---------------|
| Docker Engine | 20.10+ |
| Docker Compose | v2+ |
| RAM | 4 GB mínimo (8 GB recomendado) |
| CPU | 2 cores mínimo |
| Disco | 20 GB libres |

Puertos que deben estar libres: `3301`, `4317`, `4318`, `8888`, `9000`

---

## Instalación

### 1. Clonar el repositorio oficial

```bash
git clone -b main https://github.com/SigNoz/signoz.git
cd signoz/deploy
```

### 2. Levantar con Docker Compose

```bash
docker compose -f docker/clickhouse-setup/docker-compose.yaml up -d
```

Esto levanta los siguientes contenedores:

| Contenedor | Función |
|------------|---------|
| `signoz-frontend` | UI web en puerto 3301 |
| `signoz-otel-collector` | Recibe datos OTLP (4317/4318) |
| `signoz-query-service` | API backend de SigNoz |
| `clickhouse` | Base de datos de series de tiempo |
| `zookeeper` | Coordinación de ClickHouse |
| `signoz-alertmanager` | Gestión de alertas |

### 3. Verificar que todo esté corriendo

```bash
docker compose -f docker/clickhouse-setup/docker-compose.yaml ps
```

Todos los servicios deben mostrar estado `Up`.

### 4. Acceder a la UI

Abre en el navegador:
```
http://localhost:3301
```

En el primer acceso se solicita crear una cuenta de administrador local.

---

## Verificar que el Collector recibe datos

```bash
# Prueba rápida — debe responder HTTP 200
curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{"resourceSpans":[]}'
```

Respuesta `200` = Collector activo y listo para recibir telemetría.

---

## Acceso desde otra máquina en la red

Si SigNoz corre en el servidor `192.168.1.195`:

| Servicio | URL |
|---------|-----|
| UI | `http://192.168.1.195:3301` |
| OTLP gRPC | `192.168.1.195:4317` |
| OTLP HTTP | `http://192.168.1.195:4318` |

Asegúrate de que el firewall permita esos puertos:

```bash
# En el servidor SigNoz (Linux con firewalld)
sudo firewall-cmd --permanent --add-port=3301/tcp
sudo firewall-cmd --permanent --add-port=4317/tcp
sudo firewall-cmd --permanent --add-port=4318/tcp
sudo firewall-cmd --reload

# Con ufw
sudo ufw allow 3301/tcp
sudo ufw allow 4317/tcp
sudo ufw allow 4318/tcp
```

---

## Detener y reiniciar

```bash
# Detener (conserva datos)
docker compose -f docker/clickhouse-setup/docker-compose.yaml down

# Detener y borrar todos los datos
docker compose -f docker/clickhouse-setup/docker-compose.yaml down -v

# Reiniciar
docker compose -f docker/clickhouse-setup/docker-compose.yaml up -d
```

---

## Actualizar SigNoz

```bash
cd signoz/deploy
git pull
docker compose -f docker/clickhouse-setup/docker-compose.yaml pull
docker compose -f docker/clickhouse-setup/docker-compose.yaml up -d
```

---

## Estructura de la UI

| Sección | Contenido |
|---------|-----------|
| **Services** | Lista de servicios instrumentados con OTel, latencia p50/p99, error rate |
| **Traces** | Búsqueda y detalle de trazas individuales, vista de spans anidados |
| **Metrics** | Explorador de métricas, builder de dashboards |
| **Logs** | Búsqueda full-text de logs, correlación con trazas |
| **Alerts** | Configuración de alertas por umbral o anomalía |
| **Dashboards** | Paneles personalizados con gráficas |
