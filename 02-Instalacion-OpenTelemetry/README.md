# 📦 Instalación del Agente OpenTelemetry Collector

El **OpenTelemetry Collector** es el agente que recibe, procesa y exporta telemetría (métricas, trazas, logs) hacia SigNoz. Se instala en cada servidor que desees monitorear.

---

## Índice

- [Windows](#windows)
- [Linux](#linux)
- [AIX](#aix)
- [Configuración base del Collector](#configuración-base)
- [Verificación](#verificación)

---

## Windows

### Descarga

```powershell
# Crear directorio de instalación
New-Item -ItemType Directory -Path "C:\otelcol" -Force

# Descargar el binario (versión 0.111.0)
Invoke-WebRequest `
  -Uri "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.111.0/otelcol-contrib_0.111.0_windows_amd64.tar.gz" `
  -OutFile "C:\otelcol\otelcol.tar.gz"

# Extraer
tar -xzf C:\otelcol\otelcol.tar.gz -C C:\otelcol\
```

### Crear archivo de configuración

Crear `C:\otelcol\config.yaml` (ver sección [Configuración base](#configuración-base))

### Ejecutar manualmente (prueba)

```powershell
C:\otelcol\otelcol-contrib.exe --config C:\otelcol\config.yaml
```

### Instalar como servicio de Windows

```powershell
# Instalar como servicio con NSSM
# 1. Descargar NSSM desde https://nssm.cc/download
# 2. Registrar el servicio
nssm install OTelCollector "C:\otelcol\otelcol-contrib.exe"
nssm set OTelCollector AppParameters "--config C:\otelcol\config.yaml"
nssm set OTelCollector AppDirectory "C:\otelcol"
nssm set OTelCollector DisplayName "OpenTelemetry Collector"
nssm set OTelCollector Description "OTel Collector — envia metricas a SigNoz"
nssm set OTelCollector Start SERVICE_AUTO_START

# Iniciar servicio
nssm start OTelCollector

# Verificar estado
Get-Service -Name OTelCollector
```

### Firewall Windows

```powershell
# Permitir puertos si el collector expone endpoints locales
New-NetFirewallRule -DisplayName "OTel Collector gRPC" -Direction Inbound -Protocol TCP -LocalPort 4317 -Action Allow
New-NetFirewallRule -DisplayName "OTel Collector HTTP" -Direction Inbound -Protocol TCP -LocalPort 4318 -Action Allow
```

---

## Linux

### Descarga e instalación

```bash
# Crear directorio
sudo mkdir -p /opt/otelcol

# Descargar binario (ajustar arquitectura: amd64 o arm64)
curl -Lo /tmp/otelcol.tar.gz \
  https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.111.0/otelcol-contrib_0.111.0_linux_amd64.tar.gz

# Extraer e instalar
sudo tar -xzf /tmp/otelcol.tar.gz -C /opt/otelcol/
sudo chmod +x /opt/otelcol/otelcol-contrib

# Crear enlace simbólico
sudo ln -s /opt/otelcol/otelcol-contrib /usr/local/bin/otelcol
```

### Crear archivo de configuración

```bash
sudo mkdir -p /etc/otelcol
sudo nano /etc/otelcol/config.yaml
# Pegar contenido de la sección Configuración base
```

### Ejecutar manualmente (prueba)

```bash
otelcol --config /etc/otelcol/config.yaml
```

### Instalar como servicio systemd

Crear el archivo `/etc/systemd/system/otelcol.service`:

```ini
[Unit]
Description=OpenTelemetry Collector
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/otelcol --config /etc/otelcol/config.yaml
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
# Habilitar e iniciar el servicio
sudo systemctl daemon-reload
sudo systemctl enable otelcol
sudo systemctl start otelcol

# Verificar estado
sudo systemctl status otelcol

# Ver logs
sudo journalctl -u otelcol -f
```

### Verificar conectividad con SigNoz

```bash
# Probar que el collector puede alcanzar SigNoz (reemplazar IP)
curl -v http://192.168.1.195:4318/v1/metrics \
  -H "Content-Type: application/json" \
  -d '{"resourceMetrics":[]}'
# Respuesta 200 = OK
```

---

## AIX

> ⚠️ OpenTelemetry Collector **no tiene binario oficial para AIX** (PowerPC/ppc64). Se requiere compilar desde fuente o usar una alternativa.

### Opción A — Compilar desde fuente (requiere Go 1.21+)

```bash
# 1. Instalar Go en AIX (desde IBM o compilado)
# https://www.ibm.com/support/pages/aix-toolbox-open-source-software-downloads-alpha

# 2. Clonar el builder de OTel Collector
go install go.opentelemetry.io/collector/cmd/builder@v0.111.0

# 3. Crear manifest.yaml del builder
cat > /tmp/otelcol-builder.yaml << 'EOF'
dist:
  name: otelcol-aix
  description: OTel Collector for AIX
  output_path: /opt/otelcol

exporters:
  - gomod: go.opentelemetry.io/collector/exporter/otlpexporter v0.111.0

receivers:
  - gomod: go.opentelemetry.io/collector/receiver/hostmetricsreceiver v0.111.0
  - gomod: go.opentelemetry.io/collector/receiver/otlpreceiver v0.111.0

processors:
  - gomod: go.opentelemetry.io/collector/processor/batchprocessor v0.111.0
EOF

# 4. Compilar
builder --config /tmp/otelcol-builder.yaml
```

### Opción B — Telegraf como alternativa (binario disponible para AIX)

Telegraf puede enviar métricas en formato OTLP a SigNoz y tiene binario para AIX/ppc64:

```bash
# Descargar Telegraf para AIX desde InfluxData
# https://repos.influxdata.com/

# Configurar output hacia SigNoz
cat > /etc/telegraf/telegraf.conf << 'EOF'
[[outputs.opentelemetry]]
  endpoint = "192.168.1.195:4317"
  insecure_skip_verify = true

[[inputs.cpu]]
  percpu = true
  totalcpu = true

[[inputs.mem]]

[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs"]

[[inputs.net]]
EOF

# Iniciar
startsrc -s telegraf
```

---

## Configuración base

Archivo `config.yaml` para el OTel Collector. **Reemplaza `192.168.1.195` con la IP de tu servidor SigNoz.**

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  # Métricas del host (CPU, memoria, disco, red)
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu:
        metrics:
          system.cpu.utilization:
            enabled: true
      memory:
        metrics:
          system.memory.utilization:
            enabled: true
      disk: {}
      filesystem:
        exclude_mount_points:
          mount_points: ["/dev", "/sys", "/proc"]
          match_type: strict
      network: {}
      load: {}
      processes: {}

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000

  resourcedetection:
    detectors: [system, env]
    system:
      hostname_sources: [os]

exporters:
  otlp:
    endpoint: "192.168.1.195:4317"
    tls:
      insecure: true

  logging:
    verbosity: basic

service:
  pipelines:
    metrics:
      receivers: [hostmetrics, otlp]
      processors: [resourcedetection, batch]
      exporters: [otlp]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

---

## Verificación

Después de iniciar el collector, verificar en SigNoz:

1. **SigNoz UI** → `http://192.168.1.195:3301`
2. **Infrastructure** → debe aparecer el hostname del servidor con métricas de CPU, memoria y disco en ~1 minuto
3. **Metrics** → buscar `system.cpu.utilization` o `system.memory.usage`

Si no aparece, revisar logs del collector:
```bash
# Linux
journalctl -u otelcol -n 50

# Windows
Get-EventLog -LogName Application -Source OTelCollector -Newest 20
```
