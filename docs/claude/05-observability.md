# Observability

Brought up as its own compose profile, never in the default one. The whole stack is
about 1 GB — roughly one Java service's worth.

| Component | Purpose | Resident |
|---|---|---|
| Prometheus | Metrics scraping and storage, 7-day retention | ~250 MB |
| Grafana | Dashboards for metrics and logs in one UI | ~150 MB |
| Loki | Log aggregation, single-binary mode, filesystem backend | ~150 MB |
| Alloy | Reads container logs from the Docker socket, ships to Loki | ~80 MB |
| Kafbat UI | Topic and consumer-group inspection, lives in the `kafka` profile | ~350 MB |

Tempo is deferred to Phase 4. Traces only become interesting once a request crosses Go
to Java to Python, and it is another ~200 MB.

## Why Loki rather than Elasticsearch and Kibana

ELK is about 2.2 GB resident. On this machine that is `ledger-core`,
`exposure-aggregator` and Redpanda that could not then run at the same time. Loki is
~150 MB.

The fit is also better. Vela's services log structured JSON with a `correlation_id`,
which is exactly the shape Loki handles well: filter by label to pick the service, then
parse the JSON to pull out the correlation ID. Full-text search over unstructured log
bodies at volume — the thing Elasticsearch is genuinely better at — is not something
this project will ever do.

And Grafana is already in the stack for metrics, so Loki is a datasource in the same UI.
One console for metrics and logs, and later traces. Kibana would be a second, separate
one.

What is given up, honestly: no full-text search over log bodies, no aggregations over
log fields, and a broad time range with no label filter is slow. None of these bite at
laptop volume with structured logs.

Runner-up: VictoriaLogs — lighter still and it does do full text, but the Grafana
integration is a plugin rather than native and the ecosystem is thinner.

## Kafbat UI is a debugging tool, not observability

It is how you see what is on a topic and where a consumer group's lag sits. That is
useful from Phase 1, long before dashboards are, so it belongs in the `kafka` profile.

It is a JVM application and defaults to a quarter of host RAM. **Cap the heap.**

## Order to bring things up

1. **Now** — Kafbat UI, in the existing `kafka` profile. Used constantly from Phase 1:
   checking what actually landed on `fx.rates`, watching consumer lag.
2. **Phase 1, once `ledger-core` and `fx-ingester` exist** — Loki and Alloy. Logs matter
   as soon as there are two services in different languages to correlate across, and
   this is what forces the structured JSON logging the conventions already promise.
3. **Phase 6** — Prometheus and Grafana properly, once services expose meaningful
   `/metrics` beyond process defaults. Earlier, the dashboards would show JVM heap and
   nothing else.

## Simplifications, labelled

- **Anonymous admin access to Grafana.** Fine on localhost, would be a fireable offence
  anywhere else.
- **Single-binary Loki, no object storage, `inmemory` ring.** Correct for one machine,
  not how Loki is run in production.
- **Static Prometheus targets.** Replaced by `kubernetes_sd_configs` in Phase 2's k8s
  manifests.
- **No alerting rules.** Alerts with nobody on call are noise. Worth adding in Phase 6,
  when the chaos testing needs a signal to watch.

---

# Configuration

These belong at `observability/` in the repo root when Phase 6 arrives. They are kept
here as the reference version.

## `compose.obs.yaml`

Kept separate from `compose.yaml` so the profile cannot leak into a default `up`. Run
with:

```
docker compose -f compose.yaml -f compose.obs.yaml --profile obs up
```

```yaml
name: vela

services:
  prometheus:
    image: prom/prometheus:v3.1.0
    container_name: vela-prometheus
    profiles: ["obs"]
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=7d
      - --storage.tsdb.retention.size=2GB
      - --web.enable-lifecycle
    volumes:
      - ./observability/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    mem_limit: 512m
    restart: unless-stopped

  loki:
    image: grafana/loki:3.3.2
    container_name: vela-loki
    profiles: ["obs"]
    command: -config.file=/etc/loki/loki.yaml
    volumes:
      - ./observability/loki/loki.yaml:/etc/loki/loki.yaml:ro
      - loki-data:/loki
    ports:
      - "3100:3100"
    mem_limit: 512m
    restart: unless-stopped

  alloy:
    image: grafana/alloy:v1.6.1
    container_name: vela-alloy
    profiles: ["obs"]
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - /etc/alloy/config.alloy
    volumes:
      - ./observability/alloy/config.alloy:/etc/alloy/config.alloy:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - alloy-data:/var/lib/alloy/data
    ports:
      - "12345:12345"
    mem_limit: 256m
    depends_on:
      - loki
    restart: unless-stopped

  grafana:
    image: grafana/grafana:11.5.1
    container_name: vela-grafana
    profiles: ["obs"]
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
      GF_ANALYTICS_REPORTING_ENABLED: "false"
      GF_ANALYTICS_CHECK_FOR_UPDATES: "false"
    volumes:
      - ./observability/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./observability/grafana/dashboards:/var/lib/grafana/dashboards:ro
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    mem_limit: 256m
    depends_on:
      - prometheus
      - loki
    restart: unless-stopped

  # Debugging tool, not observability. Lives in the kafka profile so it comes up
  # with Redpanda from Phase 1 onward, long before Grafana is wanted.
  kafbat-ui:
    image: ghcr.io/kafbat/kafka-ui:v1.1.0
    container_name: vela-kafbat-ui
    profiles: ["kafka", "obs"]
    environment:
      # Kafbat is a JVM app and defaults to a quarter of host RAM. Cap it.
      JAVA_OPTS: "-Xmx256m -Xss256k -XX:MaxMetaspaceSize=128m"
      KAFKA_CLUSTERS_0_NAME: vela-local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: redpanda:9092
      DYNAMIC_CONFIG_ENABLED: "true"
    ports:
      - "8080:8080"
    mem_limit: 512m
    restart: unless-stopped

volumes:
  prometheus-data:
  loki-data:
  grafana-data:
  alloy-data:
```

## `observability/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    env: local

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  # Redpanda exposes an application-level endpoint separate from the
  # internal one; /public_metrics is the one you want.
  - job_name: redpanda
    metrics_path: /public_metrics
    static_configs:
      - targets: ["redpanda:9644"]

  # Every Vela service ships /metrics per the definition of done.
  # Add targets here as services land, commented out until they exist,
  # otherwise Prometheus fills the UI with red.
  - job_name: vela-services
    metrics_path: /metrics
    static_configs:
      # - targets: ["ledger-core:8080"]
      #   labels: { service: ledger-core, language: java }
      # - targets: ["fx-ingester:8000"]
      #   labels: { service: fx-ingester, language: python }
      # - targets: ["gateway:8080"]
      #   labels: { service: gateway, language: go }
      - targets: []
```

Static targets are fine locally. Prometheus does have Docker service discovery, but it
requires scrape annotations as labels on every service, which is more work than editing
one file at this scale. Phase 2 switches to `kubernetes_sd_configs` anyway.

## `observability/loki/loki.yaml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: warn

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-04-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h
  allow_structured_metadata: true
  volume_enabled: true
  ingestion_rate_mb: 8
  ingestion_burst_size_mb: 16
  max_query_series: 5000

compactor:
  working_directory: /loki/compactor
  delete_request_store: filesystem
  retention_enabled: true

ruler:
  storage:
    type: local
    local:
      directory: /loki/rules

analytics:
  reporting_enabled: false
```

Seven-day retention with the compactor actually enforcing it. Without
`retention_enabled: true`, Loki keeps everything forever and quietly fills the disk.

## `observability/alloy/config.alloy`

```
// Discover every container in the `vela` compose project.
discovery.docker "vela" {
  host             = "unix:///var/run/docker.sock"
  refresh_interval = "15s"
}

discovery.relabel "vela" {
  targets = discovery.docker.vela.targets

  // Only our own containers, not whatever else is running on the machine.
  rule {
    source_labels = ["__meta_docker_container_label_com_docker_compose_project"]
    regex         = "vela"
    action        = "keep"
  }

  // The compose service name is the label we actually query by.
  rule {
    source_labels = ["__meta_docker_container_label_com_docker_compose_service"]
    target_label  = "service"
  }

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }
}

loki.source.docker "vela" {
  host             = "unix:///var/run/docker.sock"
  targets          = discovery.relabel.vela.output
  forward_to       = [loki.process.vela_json.receiver]
  refresh_interval = "15s"
}

loki.process "vela_json" {
  // Parse the structured JSON the services emit.
  stage.json {
    expressions = {
      level          = "level",
      correlation_id = "correlation_id",
      logger         = "logger",
    }
  }

  // level is low-cardinality, so it earns a label.
  stage.labels {
    values = { level = "" }
  }

  // correlation_id is one value per request. As a *label* it would create a
  // stream per request and destroy Loki's index. Structured metadata makes it
  // filterable without that cost. This is the single easiest way to break Loki.
  stage.structured_metadata {
    values = { correlation_id = "" }
  }

  // Non-JSON lines (JVM startup banners, stack traces) fall through
  // unparsed rather than erroring. That is intentional.
  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

Services log JSON to stdout and know nothing about Loki. Alloy reads the Docker socket.
Swapping the log backend later is a one-container change.

## `observability/grafana/provisioning/datasources/datasources.yaml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: 15s

  - name: Loki
    type: loki
    uid: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      maxLines: 1000
      # Placeholder for Phase 4: turns correlation_id in a log line into a
      # link to the trace. Harmless until Tempo exists.
      # derivedFields:
      #   - name: TraceID
      #     matcherType: label
      #     matcherRegex: correlation_id
      #     datasourceUid: tempo
      #     url: "$${__value.raw}"
```

## `observability/grafana/provisioning/dashboards/dashboards.yaml`

```yaml
apiVersion: 1

providers:
  - name: vela
    orgId: 1
    folder: Vela
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: false
```

Create `observability/grafana/dashboards/` with a `.gitkeep`. Grafana logs an error if
the mount path is missing. Build dashboards in the UI, then export the JSON into that
folder so they are version-controlled.

## Known gotcha

On Docker Desktop for macOS, mounting `/var/run/docker.sock` works but Alloy sometimes
needs `group_add` or a socket proxy. If Alloy starts cleanly but discovers zero targets,
that is the cause.
