# 📊 Métricas de Infraestructura con SigNoz

Configuración del OTel Collector para recolectar y enviar métricas de infraestructura (CPU, memoria, disco, red, procesos) hacia SigNoz.

---

## ¿Qué métricas se recolectan?

| Categoría | Métricas clave | Receiver |
|-----------|---------------|---------|
| **CPU** | utilización por core, idle, user, system | `hostmetrics/cpu` |
| **Memoria** | used, free, cached, utilización % | `hostmetrics/memory` |
| **Disco** | operaciones de lectura/escritura, latencia | `hostmetrics/disk` |
| **Filesystem** | espacio usado/libre por partición | `hostmetrics/filesystem` |
| **Red** | bytes enviados/recibidos, errores, paquetes | `hostmetrics/network` |
| **Carga del sistema** | load average 1m, 5m, 15m | `hostmetrics/load` |
| **Procesos** | cantidad de procesos, estados | `hostmetrics/processes` |

---

## Configuración completa del Collector

Archivo `/etc/otelcol/config.yaml`:

```yaml
receivers:
  hostmetrics:
    collection_interval: 30s   # frecuencia de recolección
    scrapers:

      cpu:
        metrics:
          system.cpu.time:
            enabled: true
          system.cpu.utilization:
            enabled: true      # porcentaje de uso (0.0 - 1.0)

      memory:
        metrics:
          system.memory.usage:
            enabled: true
          system.memory.utilization:
            enabled: true

      disk:
        metrics:
          system.disk.io:
            enabled: true
          system.disk.io_time:
            enabled: true
          system.disk.operations:
            enabled: true

      filesystem:
        metrics:
          system.filesystem.usage:
            enabled: true
          system.filesystem.utilization:
            enabled: true
        exclude_mount_points:
          mount_points: ["/dev", "/sys", "/proc", "/run"]
          match_type: strict
        exclude_fs_types:
          fs_types: ["autofs", "binfmt_misc", "cgroup", "configfs",
                     "debugfs", "devpts", "devtmpfs", "fusectl",
                     "hugetlbfs", "mqueue", "nsfs", "overlay",
                     "proc", "procfs", "pstore", "rpc_pipefs",
                     "securityfs", "selinuxfs", "squashfs",
                     "sysfs", "tracefs"]
          match_type: strict

      network:
        metrics:
          system.network.io:
            enabled: true
          system.network.errors:
            enabled: true
          system.network.packets:
            enabled: true
          system.network.connections:
            enabled: true

      load:
        cpu_average: false     # true = normaliza por cantidad de cores

      processes: {}

      process:
        mute_process_name_error: true
        mute_process_exe_error: true
        mute_process_io_error: true
        metrics:
          process.cpu.time:
            enabled: true
          process.memory.physical_usage:
            enabled: true
          process.disk.io:
            enabled: true

processors:
  batch:
    timeout: 10s
    send_batch_size: 1000
    send_batch_max_size: 2000

  # Enriquece métricas con atributos del host
  resourcedetection:
    detectors: [system, env, docker]
    system:
      hostname_sources: [os]
      resource_attributes:
        host.name:
          enabled: true
        os.type:
          enabled: true
        host.arch:
          enabled: true

  # Filtra métricas específicas (opcional)
  # filter:
  #   metrics:
  #     exclude:
  #       match_type: strict
  #       metric_names: ["system.cpu.time"]  # excluir si no se necesita

exporters:
  otlp:
    endpoint: "192.168.1.195:4317"   # IP de tu servidor SigNoz
    tls:
      insecure: true
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
    sending_queue:
      enabled: true
      num_consumers: 4
      queue_size: 100

  # Opcional: ver métricas en los logs del collector
  logging:
    verbosity: basic

service:
  telemetry:
    logs:
      level: info

  pipelines:
    metrics:
      receivers: [hostmetrics]
      processors: [resourcedetection, batch]
      exporters: [otlp]
```

---

## Verificación en SigNoz

### 1. Esperar la primera recolección (~30 segundos)

### 2. En la UI de SigNoz

- Ve a **Infrastructure** (menú lateral)
- Debe aparecer el hostname del servidor con métricas en tiempo real

### 3. En el explorador de métricas

Ve a **Metrics** y busca estas métricas por nombre:

```
system.cpu.utilization
system.memory.utilization
system.filesystem.utilization
system.network.io
system.disk.io
system.load.average.1m
```

---

## Dashboards recomendados

### Importar dashboard de infraestructura prebuilt

SigNoz incluye dashboards predefinidos. Para importarlos:

1. **SigNoz UI** → **Dashboards** → **+ New Dashboard** → **Import JSON**
2. Usar los dashboards de la comunidad:
   ```
   https://github.com/SigNoz/dashboards
   ```

### Dashboard básico manual

Métricas útiles para crear paneles:

| Panel | Métrica | Función |
|-------|---------|---------|
| CPU % | `system.cpu.utilization` | avg por host |
| RAM usada | `system.memory.usage{state="used"}` | sum |
| Disco libre % | `system.filesystem.utilization` | avg |
| Red in/out | `system.network.io` | rate |
| Load avg | `system.load.average.1m` | avg |

---

## Alertas de infraestructura

Ejemplo de alerta cuando CPU supera 80%:

1. **SigNoz UI** → **Alerts** → **+ New Alert**
2. Configurar:
   ```
   Metric: system.cpu.utilization
   Condition: avg > 0.80
   Duration: 5 minutes
   Severity: Warning
   ```

---

## Solución de problemas comunes

| Síntoma | Causa probable | Solución |
|---------|---------------|----------|
| No aparecen métricas en SigNoz | Collector no conecta a SigNoz | Verificar IP/puerto y firewall |
| Métricas aparecen pero sin hostname | `resourcedetection` no configurado | Agregar processor `resourcedetection` |
| Métricas de procesos vacías | Permisos insuficientes | Ejecutar el collector como root/admin |
| Error `connection refused` al iniciar | Puerto 4317 bloqueado | Verificar firewall en servidor SigNoz |
| Collector crashea al iniciar | Sintaxis en `config.yaml` | Validar con `otelcol validate --config config.yaml` |
