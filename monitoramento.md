# Monitoramento e Observabilidade

## Visão Geral

A estratégia de monitoramento e observabilidade para a infraestrutura híbrida da XPTO foi projetada seguindo o princípio dos "Três Pilares da Observabilidade": Métricas, Logs e Traces. A abordagem garante visibilidade completa de todos os componentes da solução, com foco especial na camada de rede.

## Arquitetura de Observabilidade

```
+--------------------------------------------------+
|               Central de Observabilidade         |
+--------------------------------------------------+
|                                                  |
|  +----------------+    +----------------+        |
|  |    Prometheus  |    |    Grafana     |        |
|  |  (Métricas)    |    |  (Dashboards)  |        |
|  +----------------+    +----------------+        |
|                                                  |
|  +----------------+    +----------------+        |
|  |  Elasticsearch  |    |    Kibana     |        |
|  |     (Logs)      |    |  (Análise)    |        |
|  +----------------+    +----------------+        |
|                                                  |
|  +----------------+    +----------------+        |
|  |     Jaeger     |    |     Zipkin     |        |
|  |    (Traces)    |    |   (Traces)     |        |
|  +----------------+    +----------------+        |
|                                                  |
|  +----------------+    +----------------+        |
|  |  Network Probe |    |  AlertManager  |        |
|  | (NPM, Netflow) |    | (Notificações) |        |
|  +----------------+    +----------------+        |
|                                                  |
+--------------------------------------------------+
          ^                      ^                 
          |                      |                 
+-----------------+    +------------------+        
| On-Premises     |    | Cloud            |        
| Exporters       |    | Exporters        |        
+-----------------+    +------------------+        
```

## Pilares de Observabilidade

### 1. Métricas (Series Temporais)

**Ferramentas**: Prometheus, Grafana, Thanos para armazenamento de longo prazo

**Implementação**:
- Prometheus Server centralizado com alta disponibilidade
- Federation para coleta cross-environment
- Exporters específicos para cada componente:
  - Node Exporter: métricas de sistema
  - cAdvisor/Kubernetes Metrics: métricas de containers
  - PostgreSQL Exporter: métricas de banco de dados
  - Blackbox Exporter: testes sintéticos

**Exemplo de Configuração Prometheus**:
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts.yml"
  - "recording_rules.yml"

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager:9093

scrape_configs:
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: node
        replacement: $1

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

  - job_name: 'network-devices'
    static_configs:
      - targets:
        - 'snmp-exporter:9116'
    metrics_path: /snmp
    params:
      module: [network_devices]

  - job_name: 'netflow-collector'
    static_configs:
      - targets:
        - 'netflow-exporter:9999'
```

### 2. Logs (Eventos)

**Ferramentas**: Elasticsearch, Kibana, Fluentd/Fluent Bit, Vector

**Implementação**:
- Cluster Elasticsearch com 3+ nós para alta disponibilidade
- Coleta distribuída via Fluent Bit como DaemonSet
- Processamento e filtragem com Logstash
- Retenção diferenciada por importância do log

**Configuração Fluent Bit**:
```yaml
# fluent-bit.conf
[SERVICE]
    Flush        5
    Log_Level    info
    Daemon       off
    Parsers_File parsers.conf

[INPUT]
    Name         tail
    Path         /var/log/containers/*.log
    Parser       docker
    Tag          kube.*
    Mem_Buf_Limit 5MB

[FILTER]
    Name         kubernetes
    Match        kube.*
    Kube_URL     https://kubernetes.default.svc:443
    Merge_Log    On
    K8S-Logging.Parser On
    K8S-Logging.Exclude On

[OUTPUT]
    Name         es
    Match        *
    Host         elasticsearch-master
    Port         9200
    Index        logs-${HOSTNAME}-%Y.%m.%d
    Type         _doc
    Retry_Limit  False
```

### 3. Traces (Distribuídos)

**Ferramentas**: Jaeger, OpenTelemetry, Zipkin

**Implementação**:
- Jaeger All-in-One para ambientes menores
- Jaeger Distribuído para produção
  - Jaeger Agent como sidecar nos pods
  - Jaeger Collector centralizado
  - Elasticsearch como backend de armazenamento
- Amostragem adaptativa baseada na carga e latência

**Configuração de Jaeger**:
```yaml
# jaeger-operator-config.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://elasticsearch:9200
        username: elastic
        password: changeme
  ingress:
    enabled: true
    hosts:
      - jaeger.xpto.com
  agent:
    strategy: DaemonSet
  collector:
    replicas: 3
    resources:
      limits:
        memory: 1Gi
        cpu: 500m
  query:
    replicas: 2
```

## Monitoramento de Rede (Foco Especial)

### 1. Camadas do Modelo OSI Monitoradas

| Camada OSI | Métrica Monitorada | Ferramenta | Alerta |
|------------|-------------------|------------|--------|
| 1 - Física | Estado das interfaces, erros físicos | SNMP, Prometheus | > 10 erros/min |
| 2 - Enlace | Colisões, CRC, broadcast storm | SNMP, sFlow | > 5% colisões |
| 3 - Rede | Latência, packet loss, jitter | ICMP, Smokeping | Latência > 100ms |
| 4 - Transporte | Conexões TCP, retransmissões | NetFlow, tcpdump | Retransmit > 3% |
| 5 - Sessão | Sessões ativas/máximas | Custom exporters | Uso > 80% |
| 6 - Apresentação | Erros TLS/SSL, certificados | TLS probe | Cert < 30 dias |
| 7 - Aplicação | Tempo de resposta HTTP, erros | Blackbox exporter | Resp > 2s |

### 2. Ferramentas Especializadas para Rede

#### 2.1 Monitoramento de Tráfego

- **NetFlow/IPFIX Collector**
  - Análise de fluxos de tráfego
  - Identificação de consumo por aplicação
  - Detecção de anomalias

- **sFlow/jFlow**
  - Amostragem de pacotes para análise estatística
  - Visibilidade de tráfego em switches/routers

#### 2.2 Monitoramento de Conectividade

- **VPN Monitoring**
  - Estado dos túneis VPN site-to-site
  - Latência e throughput entre ambientes
  - Monitoramento de certificados VPN

```yaml
# blackbox-exporter.yml - Para testes de conectividade
modules:
  vpn_tunnel:
    prober: tcp
    timeout: 5s
    tcp:
      preferred_ip_protocol: "ip4"
      tls: true
      tls_config:
        insecure_skip_verify: false
    
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      method: GET
      no_follow_redirects: false
      fail_if_ssl: false
      fail_if_not_ssl: true
      preferred_ip_protocol: "ip4"
      tls_config:
        insecure_skip_verify: false
```

#### 2.3 Monitoramento de Desempenho

- **Smokeping**
  - Histórico de latência de longo prazo
  - Identificação de padrões e tendências

- **Network Weather Map**
  - Visualização da utilização de links
  - Capacidade de planejamento

### 3. Dashboards de Rede

#### 3.1 Dashboard de Visão Geral

![Network Dashboard Overview](../images/net_dash_overview.png)

Componentes:
- **Mapa topológico**: Visualização interativa
- **Status Conectividade**: Verde/amarelo/vermelho por link
- **Métricas Agregadas**: 
  - Utilização média de links (%)
  - Pacotes por segundo
  - Erros e descartes

#### 3.2 Dashboard de Links Críticos

Monitoramento específico para enlaces críticos:

- **VPN Site-to-Site**: 
  - Latência bidirecional
  - Jitter
  - Throughput em tempo real
  - Status do túnel
  - Utilização de criptografia

- **Conectividade Interna**:
  - Latência inter-service
  - Taxa de erros
  - Taxas de transferência
  - Queue depth em switches/routers

#### 3.3 Dashboard de Troubleshooting

Ferramentas para resolução de problemas:

- **Flow Analysis**: Top talkers, aplicações, protocolos
- **Packet Loss Heatmap**: Visualização temporal de perda de pacotes
- **TCP Retransmission**: Identificação de problemas de congestionamento
- **DNS Resolution Times**: Performance de resolução de nomes
- **TLS Handshake Times**: Performance de negociação TLS

## Alertas e Notificações

### Estratégia de Alerting

Estruturada em três níveis de severidade:

1. **Info**: Métricas fora do normal, sem impacto imediato
2. **Warning**: Potencial degradação, requer atenção
3. **Critical**: Impacto direto no serviço, requer ação imediata

### Regras de Alertas para Rede

```yaml
# network-alerts.yml
groups:
- name: network_alerts
  rules:
  - alert: HighPacketLoss
    expr: rate(node_network_receive_drop_total[5m]) / rate(node_network_receive_packets_total[5m]) > 0.01
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High packet loss detected"
      description: "Node {{ $labels.instance }} has {{ $value | humanizePercentage }} packet loss on interface {{ $labels.device }}"

  - alert: NetworkInterfaceDown
    expr: node_network_up{device!~"lo|veth.*|docker.*|br.*"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Network interface down"
      description: "Network interface {{ $labels.device }} on {{ $labels.instance }} is down"

  - alert: VPNTunnelDown
    expr: probe_success{job="vpn_tunnel"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "VPN tunnel down"
      description: "VPN connection to {{ $labels.destination }} is down for more than 5 minutes"

  - alert: HighNetworkLatency
    expr: avg_over_time(probe_duration_seconds{job="icmp_monitoring"}[5m]) > 0.1
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High network latency"
      description: "Network latency to {{ $labels.destination }} is {{ $value | humanizeDuration }} which exceeds threshold (100ms)"
```

### Canais de Notificação

- **Alertmanager**: Centralização e deduplicação de alertas
- **Integrações**:
  - Slack/Teams para comunicação em tempo real
  - Email para notificações formais
  - SMS/Ligação para alertas críticos
  - Tickets no sistema de ITSM

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'

route:
  receiver: 'slack-notifications'
  group_by: ['alertname', 'instance', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
  - match:
      severity: critical
    receiver: 'pagerduty'
    continue: true
  - match_re:
      service: ^(network|connectivity)$
    receiver: 'network-team'

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#monitoring'
    send_resolved: true
    title: '{{ template "slack.default.title" . }}'
    text: '{{ template "slack.default.text" . }}'

- name: 'pagerduty'
  pagerduty_configs:
  - service_key: XXXXXXXXXXXXXXXXXXXXXXXX
    send_resolved: true
    
- name: 'network-team'
  slack_configs:
  - channel: '#network-alerts'
    send_resolved: true
  email_configs:
  - to: 'network-team@xpto.com'
    send_resolved: true
```

## Integração com o Modelo OSI para Troubleshooting

### Fluxo de Diagnóstico Baseado em Camadas

1. **Detecção**: Alerta gerado por qualquer ferramenta de monitoramento
2. **Classificação**: Determinação da camada OSI afetada
3. **Isolamento**: Ferramentas específicas para cada camada
4. **Remediação**: Runbooks específicos para cada tipo de problema

### Runbooks por Camada OSI

| Camada | Sintoma | Ferramenta de Diagnóstico | Resolução Comum |
|--------|---------|---------------------------|-----------------|
| Física | Interface flapping | SNMP Polling | Verificar cabos, transceivers |
| Enlace | MAC duplicado | ARP tables, CAM tables | Verificar topologia, loop |
| Rede | Rotas inconsistentes | Traceroute, BGP status | Revisar política de roteamento |
| Transporte | Timeouts TCP | tcpdump, Wireshark | Ajustar TCP window, MTU |
| Sessão | Sessões órfãs | netstat, ss | Ajustar timeouts |
| Apresentação | Erros SSL | openssl, curl | Atualizar certificados |
| Aplicação | HTTP 5xx | Logs, tracing | Revisar código, escalabilidade |

## Métricas Específicas para o Sistema de Fluxo de Caixa

### Serviço de Controle de Lançamentos

| Métrica | Descrição | SLO | Fonte |
|---------|-----------|-----|-------|
| Latência de Transação | Tempo para processar uma transação | < 200ms | Application |
| Taxa de Erros | % de transações com erro | < 0.1% | Application |
| Disponibilidade | Uptime do serviço | 99.95% | Synthetic |
| Concorrência | Transações simultâneas | < 500 | Application |
| DB Connections | Conexões ativas com DB | < 80% pool | Database |

### Serviço de Consolidado Diário

| Métrica | Descrição | SLO | Fonte |
|---------|-----------|-----|-------|
| Tempo de Processamento | Duração do processamento diário | < 30 min | Application |
| Registro por Segundo | Velocidade de processamento | > 100/s | Application |
| Queue Depth | Tamanho da fila de mensagens | < 10,000 | Messaging |
| Rejections | Transações rejeitadas | < 5% | Application |
| Resource Utilization | CPU/Memory durante picos | < 80% | Kubernetes |

## Logs Estruturados

Padronização de logs em formato JSON para facilitar análise:

```json
{
  "timestamp": "2024-04-15T14:32:53.142Z",
  "service": "lancamentos-api",
  "instance": "pod-lancamentos-57f7d5b67-2nlwj",
  "level": "INFO",
  "traceId": "abcdef123456",
  "userId": "user-123",
  "message": "Transaction processed successfully",
  "transactionId": "tx-789",
  "amount": 1500.50,
  "processingTime": 45,
  "source": "web-app"
}
```

## Capacidade e Retenção

| Tipo de Dado | Volume Diário | Retenção | Armazenamento |
|--------------|---------------|----------|--------------|
| Métricas | ~50GB | 15 dias (alta resolução) | Prometheus |
| Métricas | ~10GB | 1 ano (baixa resolução) | Thanos |
| Logs | ~200GB | 7 dias | Hot Elasticsearch |
| Logs | ~1.5TB | 30 dias | Warm Elasticsearch |
| Logs | ~6TB | 1 ano | Cold Storage |
| Traces | ~100GB | 3 dias | Jaeger |
| NetFlow | ~75GB | 30 dias | NetFlow Analyzer |

## Automação de Resposta a Incidentes

Procedimentos automatizados para resolução de problemas comuns:

1. **Auto-healing**: Remediar problemas sem intervenção humana
   - Restart automático de pods com falha
   - Reinicialização de interfaces flapping
   - Failover automático de túneis VPN

2. **Runbooks Automatizados**: Procedimentos semi-automatizados
   - Coleta proativa de informações de diagnóstico
   - Escalation inteligente com contexto apropriado
   - Geração automática de tickets com dados relevantes

## Roadmap de Observabilidade

| Fase | Objetivo | Timeline |
|------|----------|----------|
| 1 | Instrumentação básica e coleta centralizada | Mês 1-2 |
| 2 | Dashboards e alertas por componente | Mês 3-4 |
| 3 | Correlação cross-layer e automação | Mês 5-6 |
| 4 | Observabilidade orientada a negócio | Mês 7-12 |
