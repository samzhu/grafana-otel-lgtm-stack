# Grafana LGTM (Loki, Grafana, Tempo, Metrics) Observability Platform

This project provides a complete local observability platform, using Docker Compose to integrate the main components of the Grafana ecosystem to achieve End-to-End Observability.

## Architecture Overview

This environment includes the following core components:

- **OpenTelemetry Collector** - Receives OTLP data and routes to various storage backends
- **Prometheus** - Stores and queries metrics
- **Loki** - Stores and queries logs
- **Tempo** - Stores and queries traces
- **Grafana** - Unified visualization and query interface

The overall data flow is: Application → OpenTelemetry Collector → Dedicated storage backend (Prometheus/Loki/Tempo) → Grafana unified query and presentation

![LGTM Architecture](https://grafana.com/media/blog/otel-lgtm/architecture.png)
*Figure: OpenTelemetry Collector receives OTLP telemetry data (traces/metrics/logs) sent from applications, and forwards metrics to Prometheus, traces to Tempo, and logs to Loki; Grafana is pre-configured with these three data sources for observability queries.*

## Quick Start

### Starting the Environment

```bash
# Start all services
docker compose up -d

# Check service status
docker compose ps
```

### Accessing Services

After the environment starts, you can access the services via the following URLs:

- **Grafana**: http://localhost:3000
  - Default username: admin
  - Default password: admin
- **Prometheus**: http://localhost:9090
- **Loki**: http://localhost:3100
- **Tempo**: http://localhost:3200
- **OpenTelemetry Collector**: http://localhost:4318 (OTLP HTTP)

### Using Services in Grafana

After logging into Grafana, you can:

1. View default dashboards displaying metrics collected from Prometheus
2. Query log data from Loki in the Explore page
3. Query trace data from Tempo in the Explore page

## Configuration Files

### Docker Compose Structure (`docker-compose.yml`)

```yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
    command: ["--config", "/etc/otel-collector-config.yaml"]
    ports:
      - "4318:4318"                   # OTLP HTTP
      - "9464:9464"                   # Prometheus metrics
    depends_on:
      - prometheus
      - loki
      - tempo

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"

  loki:
    image: grafana/loki:latest
    container_name: loki
    user: "0:0"
    volumes:
      - ./loki-local-config.yaml:/etc/loki/local-config.yaml:ro
      - loki-data:/tmp/loki
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"

  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    user: "0:0"
    volumes:
      - ./tempo-local.yaml:/etc/tempo/tempo-local.yaml:ro
      - tempo-data:/tmp/tempo
    command: ["-config.file=/etc/tempo/tempo-local.yaml"]
    ports:
      - "3200:3200"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin 
      - GF_SECURITY_ADMIN_PASSWORD=admin 
    volumes:
      - ./grafana/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml:ro
      - ./grafana/dashboards.yml:/etc/grafana/provisioning/dashboards/dashboards.yml:ro
      - ./grafana/dashboards/:/var/lib/grafana/dashboards:ro
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - loki
      - tempo

volumes:
  loki-data: {}
  tempo-data: {}
```

### OpenTelemetry Collector Configuration (`otel-collector-config.yaml`)

OpenTelemetry Collector is responsible for receiving, processing, and routing telemetry data:

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch: {}

exporters:
  prometheus:
    endpoint: "0.0.0.0:9464"
    resource_to_telemetry_conversion:
      enabled: true

  otlphttp/loki:
    endpoint: "http://loki:3100/loki/api/v1/push"

  otlp/tempo:
    endpoint: "tempo:4317"
    insecure: true

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/loki]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo]
```

### Prometheus Configuration (`prometheus.yml`)

Prometheus is configured to scrape metrics from the OpenTelemetry Collector:

```yaml
global:
  scrape_interval: 5s
scrape_configs:
  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:9464']
```

### Loki Configuration (`loki-local-config.yaml`)

Loki is used for storing and querying log data, with settings suitable for local development:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2022-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v12
      index:
        prefix: index_
        period: 24h

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  allow_structured_metadata: true
```

### Tempo Configuration (`tempo-local.yaml`)

Tempo stores and queries trace data:

```yaml
server:
  http_listen_port: 3200
  grpc_listen_port: 4317

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
        http:

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1

compactor:
  compaction:
    block_retention: 1h

storage:
  trace:
    backend: local
    wal:
      path: /tmp/tempo/wal
    local:
      path: /tmp/tempo/blocks
```

### Grafana Data Source Configuration (`grafana/datasources.yml`)

Grafana is pre-configured with three data sources:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: false

  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    isDefault: false
```

### Grafana Dashboard Configuration (`grafana/dashboards.yml`)

Grafana dashboard provider configuration for automatically loading dashboard definition files:

```yaml
apiVersion: 1
providers:
  - name: 'Local Dashboards'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 5
    options:
      path: /var/lib/grafana/dashboards
```

This configuration will automatically load JSON format dashboard definition files from the `/var/lib/grafana/dashboards` directory in the container. In the Docker Compose configuration, we mount the host's `./grafana/dashboards/` directory to the container's path to automatically deploy dashboard definition files to Grafana.

### Default Dashboard Definition (`grafana/dashboards/springboot-observability.json`)

System default includes a simple demo dashboard for displaying data from Prometheus and Loki:

```json
{
    "uid": "springboot-observability",
    "title": "Spring Boot OTEL Demo Dashboard",
    "time": {
        "from": "now-5m",
        "to": "now"
    },
    "panels": [
        {
            "type": "timeseries",
            "title": "JVM Heap Memory Used",
            "datasource": "Prometheus",
            "targets": [
                {
                    "expr": "process_runtime_jvm_memory_usage{service_name=\"springboot-otel-demo\", type=\"heap\"}"
                }
            ],
            "fieldConfig": {
                "defaults": {
                    "unit": "bytes"
                }
            },
            "gridPos": {
                "x": 0,
                "y": 0,
                "w": 12,
                "h": 8
            }
        },
        {
            "type": "logs",
            "title": "Application Logs (springboot-otel-demo)",
            "datasource": "Loki",
            "targets": [
                {
                    "expr": "{service_name=\"springboot-otel-demo\"}"
                }
            ],
            "gridPos": {
                "x": 0,
                "y": 8,
                "w": 24,
                "h": 8
            }
        }
    ]
}
```

This dashboard includes two panels:
1. JVM Heap Memory Usage Time Series Chart - Get Java application memory metrics from Prometheus
2. Application Logs Panel - Get logs from Loki for the `springboot-otel-demo` service

When you integrate your application, you can modify this dashboard or create a new dashboard to monitor your application.

## Application Integration

You can integrate your application into this observability platform in the following ways:

1. Add OpenTelemetry support to your application to output OTLP format telemetry data
2. Configure your application to send OTLP data to the Collector's HTTP endpoint (`http://localhost:4318`)

### Integration Example

For Java applications, you can use the OpenTelemetry Java Agent:

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.exporter.otlp.endpoint=http://localhost:4318 \
     -Dotel.resource.attributes=service.name=your-service-name \
     -jar your-application.jar
```

For Node.js applications:

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { OTLPMetricExporter } = require('@opentelemetry/exporter-metrics-otlp-http');
const { OTLPLogExporter } = require('@opentelemetry/exporter-logs-otlp-http');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://localhost:4318/v1/traces'
  }),
  metricExporter: new OTLPMetricExporter({
    url: 'http://localhost:4318/v1/metrics'
  }),
  logExporter: new OTLPLogExporter({
    url: 'http://localhost:4318/v1/logs'
  }),
  serviceName: 'your-service-name'
});

sdk.start();
```

## Management Operations

### Reset Environment

To completely reset the environment (including deleting all data):

```bash
docker compose down -v
docker compose up -d
```

### Start Specific Services

If only specific services are needed:

```bash
# For example, only start Grafana, Prometheus, and Loki
docker compose up -d grafana prometheus loki
```

## Advanced Usage

### Query Techniques

- **Prometheus (PromQL)**: `rate(http_requests_total{service="your-service"}[1m])`
- **Loki (LogQL)**: `{service_name="your-service"} |= "error"`
- **Tempo**: Query by TraceID or use TraceQL

### Data Persistence

Currently configured to use Docker volume to store data:
- `loki-data`: Stores Loki log data
- `tempo-data`: Stores Tempo trace data

These volumes will retain data when containers restart, but will be deleted when executing `docker compose down -v`.

## Troubleshooting

If you encounter issues, you can check the logs of each service:

```bash
# Check logs of a specific service
docker compose logs grafana
docker compose logs otel-collector

# Real-time view of logs for all services
docker compose logs -f
```

## Reference

- [Grafana LGTM Introduction](https://grafana.com/blog/2024/03/13/an-opentelemetry-backend-in-a-docker-image-introducing-grafana/otel-lgtm/)
- [Grafana Documentation](https://grafana.com/docs/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Tempo Documentation](https://grafana.com/docs/tempo/latest/)
