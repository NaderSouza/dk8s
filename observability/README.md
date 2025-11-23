# Stack de Observabilidade

Este diretório adiciona uma stack completa de observabilidade ao laboratório do Dk8s utilizando Helm Charts oficiais e GitOps com Argo CD. A arquitetura cobre métricas, logs, traces e coleta centralizada via OpenTelemetry.

## Visão geral da arquitetura

| Componente | Namespace | Função |
| --- | --- | --- |
| kube-prometheus-stack (Prometheus, Alertmanager, Grafana, exporters) | `monitoring` | Métricas do cluster, dashboards e alertas. |
| Loki + Promtail | `logging` | Armazenamento de logs e coleta via DaemonSet. |
| Tempo | `tracing` | Backend de traces distribuídos compatível com OTLP. |
| OpenTelemetry Collector | `telemetry` | Coletor central para traces/métricas, expondo OTLP HTTP/gRPC e endpoint Prometheus. |

A figura mental abaixo ajuda a entender o fluxo:

```
Aplicações → (OTLP) → OTel Collector → Tempo (traces)
                                  ↘→ Prometheus (métricas)
                                  ↘→ Logging exporter (debug)
Promtail → Loki → Grafana (datasource)
Grafana → Prometheus / Loki / Tempo → Dashboards
```

## Layout dos arquivos

```
observability/
├── namespaces/                  # Namespaces dedicados (monitoring, logging, tracing, telemetry)
├── monitoring/                  # Chart Helm wrapper para kube-prometheus-stack + dashboards
├── logging/                     # Chart Helm wrapper para loki-stack (Loki + Promtail)
├── tracing/                     # Chart Helm wrapper para Tempo
├── otel-collector/              # Manifests Kustomize do OpenTelemetry Collector
└── README.md
argocd/
└── observability/
    ├── apps/                    # Applications individuais (monitoring, logging, tracing, otel)
    └── app-of-apps.yaml         # Application raiz (observability-stack)
```

## Deploy com Argo CD

1. Garanta que o cluster já possui o namespace `argocd` e o operador instalado.
2. Aplique os namespaces de observabilidade:
   ```bash
   kubectl apply -f observability/namespaces
   ```
3. Cadastre este repositório no Argo CD (ex.: `argocd repo add https://github.com/nader/dk8s.git`).
4. Aplique o App-of-Apps:
   ```bash
   kubectl apply -f argocd/observability/app-of-apps.yaml
   ```
5. A sincronização do recurso `Application/observability-stack` criará automaticamente as Applications filhas:
   - `observability-monitoring` → Helm chart `observability/monitoring` (kube-prometheus-stack)
   - `observability-logging` → Helm chart `observability/logging` (Loki + Promtail)
   - `observability-tracing` → Helm chart `observability/tracing` (Tempo)
   - `observability-otel-collector` → kustomize `observability/otel-collector`

Cada Application já vem com `syncPolicy.automated` (prune + selfHeal) e `syncOptions` para criar namespaces automaticamente.

## OpenTelemetry Collector

- Receivers: OTLP gRPC/HTTP (`otel-collector.telemetry.svc.cluster.local:4317/4318`).
- Pipelines:
  - `traces`: OTLP → batch → Tempo (`tempo-distributor.tracing.svc.cluster.local:4317`) + logging exporter (debug).
  - `metrics`: OTLP → batch → Prometheus exporter (porta 8889 /metrics).
- ServiceMonitor: gerado pelo chart de monitoring para que Prometheus scrapeie o endpoint `metrics` exposto pelo Service `otel-collector`.

## Configurações do Grafana

O chart de monitoring adiciona as datasources automaticamente:
- `Prometheus` (default) apontando para `kube-prometheus-stack-prometheus.monitoring.svc:9090`.
- `Loki` (`http://loki.logging.svc:3100`).
- `Tempo` (`http://tempo-query-frontend.tracing.svc:3100`).

Dashboards versionados:
- **Kubernetes Cluster Overview** — status dos nós e requisições ao API Server.
- **Application Health Overview** — métricas HTTP agregadas para serviços instrumentados.
- **Distributed Traces** — painel com o explorador nativo do Tempo.

## Instrumentando aplicações

Como exemplo, o Pod `day-1/kind/pod.yaml` foi atualizado com variáveis de ambiente OTEL:

```yaml
    env:
      - name: OTEL_SERVICE_NAME
        value: nhs-app
      - name: OTEL_EXPORTER_OTLP_ENDPOINT
        value: http://otel-collector.telemetry.svc.cluster.local:4318
      - name: OTEL_EXPORTER_OTLP_PROTOCOL
        value: http/protobuf
      - name: OTEL_RESOURCE_ATTRIBUTES
        value: service.namespace=day-1,env=lab
      - name: OTEL_TRACES_EXPORTER
        value: otlp
      - name: OTEL_METRICS_EXPORTER
        value: otlp
      - name: OTEL_LOGS_EXPORTER
        value: otlp
```

Para replicar em outros Deployments/Helm values:
1. Garanta que a aplicação carrega o SDK/auto-instrumentation adequado (ex.: `opentelemetry-instrumentation` em Python, `@opentelemetry/sdk-node` em Node.js, etc).
2. Configure a variável `OTEL_EXPORTER_OTLP_ENDPOINT` para `http://otel-collector.telemetry.svc.cluster.local:4318` ou `grpc://...:4317`.
3. Defina `OTEL_SERVICE_NAME` com o nome lógico do serviço e `OTEL_RESOURCE_ATTRIBUTES` para injetar `service.namespace`, `deployment.environment` ou tags usadas nas consultas/dashboards.
4. Opcional: defina `OTEL_METRICS_EXPORTER=otlp` e `OTEL_LOGS_EXPORTER=otlp` para enviar métricas e logs estruturados pelo mesmo collector.

## Acesso ao Grafana

- Service: `kube-prometheus-stack-grafana.monitoring.svc.cluster.local` (porta 80).
- Para acesso local: `kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80` e usar as credenciais padrão `admin/admin` (ajuste em `observability/monitoring/values.yaml`).
- Datasources e dashboards são sincronizados automaticamente pelos sidecars do chart.

## Próximos passos sugeridos

1. Habilitar persistência (PVC) para Loki, Prometheus e Tempo conforme necessidade do laboratório.
2. Adicionar regras de alerta customizadas no `kube-prometheus-stack` (PrometheusRules) seguindo o mesmo chart wrapper.
3. Instrumentar os demais serviços do repositório (Deployments, Jobs, etc.) repetindo as variáveis OTEL e, se necessário, sidecars/auto-instrumentation.
