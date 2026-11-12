# 🔧 Troubleshooting — SigNoz + OpenTelemetry

Catálogo de errores reales encontrados durante la implementación, con causa raíz y solución verificada.

---

## Índice de errores

1. [`ModuleNotFoundError: No module named 'pkg_resources'`](#1-modulenotfounderror-no-module-named-pkg_resources)
2. [`RuntimeError: Cannot add middleware after an application has started`](#2-runtimeerror-cannot-add-middleware-after-an-application-has-started)
3. [SigNoz no recibe datos (sin error en logs del backend)](#3-signoz-no-recibe-datos-sin-error-en-logs)
4. [Dashboard de la app muestra todo en 0 (datos vacíos)](#4-dashboard-muestra-todo-en-0)
5. [Backend 502 Bad Gateway desde el frontend](#5-backend-502-bad-gateway)
6. [Error de conexión gRPC al OTel Collector](#6-error-de-conexión-grpc)
7. [Imagen Docker cacheada no toma nuevas dependencias](#7-imagen-docker-cacheada)
8. [Seed de MongoDB no corre en nuevo volumen](#8-seed-de-mongodb-no-corre)

---

## 1. `ModuleNotFoundError: No module named 'pkg_resources'`

**Entorno:** Python 3.12 + `python:3.12-slim` en Docker

**Error completo:**
```
File "/usr/local/lib/python3.12/site-packages/opentelemetry/instrumentation/dependencies.py", line 4, in <module>
    from pkg_resources import (
ModuleNotFoundError: No module named 'pkg_resources'
```

**Causa:**
`python:3.12-slim` no incluye `setuptools`, que contiene `pkg_resources`. Los instrumentors de OTel lo necesitan para verificar versiones de dependencias.

**Solución:**
Agregar a `requirements.txt`:
```
setuptools==75.3.0
grpcio==1.68.0
```

Luego reconstruir sin caché:
```bash
docker compose build --no-cache backend
```

---

## 2. `RuntimeError: Cannot add middleware after an application has started`

**Error completo:**
```
File "/usr/local/lib/python3.12/site-packages/starlette/applications.py", line 131, in add_middleware
    raise RuntimeError("Cannot add middleware after an application has started")
RuntimeError: Cannot add middleware after an application has started
```

**Causa:**
`FastAPIInstrumentor.instrument_app(app)` se llamó dentro del `lifespan` de FastAPI. Para ese momento uvicorn ya inició y no acepta nuevos middlewares.

**Solución — orden correcto en `main.py`:**

```python
# ❌ INCORRECTO — dentro del lifespan
@asynccontextmanager
async def lifespan(app):
    setup_telemetry(app)   # <-- aquí falla
    await connect_db()
    yield

# ✅ CORRECTO — a nivel de módulo
from app.core.otel import setup_telemetry, instrument_app

setup_telemetry()          # 1. antes de crear la app

app = FastAPI(...)         # 2. crear app
app.add_middleware(...)    # 3. middlewares propios
instrument_app(app)        # 4. instrumentar FastAPI

@asynccontextmanager
async def lifespan(app):   # 5. lifespan solo para DB
    await connect_db()
    yield
```

**Regla:** `FastAPIInstrumentor.instrument_app()` debe ejecutarse **antes de que uvicorn arranque**, no durante el lifespan.

---

## 3. SigNoz no recibe datos (sin error en logs)

**Síntoma:** El backend arranca sin errores, pero en SigNoz → Services no aparece el servicio.

**Causa 1:** El endpoint gRPC tiene el prefijo `http://`

```python
# ❌ Incorrecto para gRPC
OTEL_ENDPOINT = "http://192.168.1.195:4317"

# ✅ Correcto para gRPC — solo host:puerto
OTEL_ENDPOINT = "192.168.1.195:4317"
```

**Causa 2:** Firewall bloqueando puerto 4317 en el servidor SigNoz

```bash
# Verificar conectividad desde el servidor del backend
curl -v http://192.168.1.195:4318/v1/traces \
  -X POST -H "Content-Type: application/json" -d '{}'
# Si no responde: firewall bloqueado

# Abrir puertos en el servidor SigNoz (Linux)
sudo firewall-cmd --permanent --add-port=4317/tcp
sudo firewall-cmd --permanent --add-port=4318/tcp
sudo firewall-cmd --reload
```

**Causa 3:** El SDK descarta spans silenciosamente si el collector no está disponible
OTel no crashea cuando no puede conectar — simplemente descarta. Generar tráfico y esperar 30-60 segundos antes de concluir que no funciona.

---

## 4. Dashboard muestra todo en 0

**Síntoma:** La app React muestra "0 tarjetas", "0 clientes" después de un rebuild.

**Causa:** El volumen de MongoDB tiene datos de una versión anterior. El script de seed `init.js` solo corre cuando el volumen está **vacío**.

**Solución:**
```bash
# Bajar contenedores Y borrar el volumen de datos
docker compose down -v

# Volver a levantar (corre el seed desde cero)
docker compose up
```

> ⚠️ `-v` borra los datos permanentemente. Solo usar en desarrollo.

**Verificar que el seed corrió:**
```bash
docker compose logs mongo | grep "✅"
# Debe mostrar:
# ✅ Clientes insertados: 5
# ✅ Tarjetas insertadas: 8
# ✅ Transacciones insertadas: XX
```

---

## 5. Backend 502 Bad Gateway

**Log de nginx:**
```
connect() failed (111: Connection refused) while connecting to upstream
upstream: "http://172.18.0.4:8000/api/cards/"
```

**Causa:** El contenedor del backend está caído (crasheó durante el startup).

**Diagnóstico:**
```bash
docker compose logs backend --tail=30
```

Buscar el error real que causó el crash (generalmente uno de los errores anteriores).

**Solución:** Resolver el error del backend y hacer rebuild:
```bash
docker compose up --build backend
```

---

## 6. Error de conexión gRPC

**Error en logs del backend:**
```
grpc._channel._InactiveRpcError: StatusCode.UNAVAILABLE
failed to connect to all addresses
```

**Causa:** El servidor SigNoz no está corriendo, la IP es incorrecta, o el puerto 4317 está bloqueado.

**Diagnóstico:**
```bash
# Desde el servidor del backend
nc -zv 192.168.1.195 4317
# "Connection to 192.168.1.195 4317 port [tcp] succeeded!" = OK

# O con telnet
telnet 192.168.1.195 4317
```

> Nota: OTel no crashea por este error — solo loguea y descarta los spans. El backend sigue funcionando.

---

## 7. Imagen Docker cacheada

**Síntoma:** Se agrega un paquete a `requirements.txt` pero el build no lo instala.

**Causa:** Docker usa la capa cacheada del `pip install` porque el hash de `requirements.txt` no cambió en la capa anterior.

**Solución:**
```bash
docker compose build --no-cache backend
```

**Cuándo usar `--no-cache`:**
- Al agregar/cambiar dependencias de Python o npm
- Cuando sospechas que hay una versión incorrecta de un paquete
- Después de cambiar la imagen base

---

## 8. Seed de MongoDB no corre

**Síntoma:** Mongo arranca pero no ejecuta `init.js`.

**Causa:** El volumen `mongo_data` ya existe y tiene datos. MongoDB solo ejecuta los scripts de `/docker-entrypoint-initdb.d/` en el **primer arranque** con volumen vacío.

**Solución:**
```bash
docker compose down -v && docker compose up
```

**Cómo confirmar que el seed corrió:**
```bash
docker compose logs mongo 2>&1 | grep -E "(✅|init process complete|error)"
```

---

## Comandos de diagnóstico rápido

```bash
# Ver estado de todos los contenedores
docker compose ps

# Logs en tiempo real de todos los servicios
docker compose logs -f

# Logs solo del backend (últimas 50 líneas)
docker compose logs backend --tail=50

# Entrar al contenedor del backend para depurar
docker compose exec backend bash

# Verificar variables de entorno en el contenedor
docker compose exec backend env | grep OTEL

# Reiniciar solo un servicio sin rebuild
docker compose restart backend

# Rebuild y reiniciar solo el backend
docker compose up --build backend
```
