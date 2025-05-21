# Estratégia de FinOps

## Visão Geral

A abordagem de FinOps (Financial Operations) implementada para a infraestrutura híbrida da XPTO segue o framework FinOps oficial, com adaptações específicas para o contexto de migração híbrida. O objetivo é garantir otimização de custos sem comprometer a performance, segurança ou resiliência.

## Princípios Orientadores

1. **Visibilidade em Tempo Real**
   - Monitoramento contínuo de custos e uso de recursos
   - Dashboards específicos para análise financeira da infraestrutura

2. **Responsabilidade Compartilhada**
   - Equipes de desenvolvimento e operações conscientes do impacto financeiro
   - Tags e alocação de custos por serviço/equipe

3. **Otimização Contínua**
   - Revisões regulares de uso e custo
   - Automação de ajustes de escala e rightsizing

4. **Planejamento Preditivo**
   - Modelagem de tendências de crescimento
   - Análise proativa do custo total de propriedade (TCO)

## Framework de Governança

```
+--------------------------------------+
|            FinOps Lifecycle          |
+--------------------------------------+
|                                      |
|    +-------------------------+       |
|    |      INFORMAR          |       |
|    | • Visibilidade         |       |
|    | • Alocação             |       |
|    | • Benchmarks           |       |
|    +-------------------------+       |
|                |                     |
|                v                     |
|    +-------------------------+       |
|    |      OTIMIZAR          |       |
|    | • Rightsizing          |       |
|    | • Reservas             |       |
|    | • Automação            |       |
|    +-------------------------+       |
|                |                     |
|                v                     |
|    +-------------------------+       |
|    |      OPERAR            |       |
|    | • Políticas            |       |
|    | • Anomalias            |       |
|    | • Previsões            |       |
|    +-------------------------+       |
|                |                     |
|                v                     |
|           [Repetir Ciclo]            |
|                                      |
+--------------------------------------+
```

## Implementação de Otimização de Custos

### 1. Etiquetagem e Alocação

Todo recurso na infraestrutura híbrida terá tags obrigatórias:

| Tag | Descrição | Exemplo |
|-----|-----------|---------|
| `environment` | Ambiente de execução | `production`, `staging`, `development` |
| `service` | Serviço/aplicação | `lancamentos`, `consolidado` |
| `cost-center` | Centro de custo | `finance`, `it` |
| `owner` | Equipe responsável | `team-core`, `team-data` |
| `auto-scale` | Elegível para auto-scaling | `true`, `false` |

Implementado via políticas no Terraform:

```hcl
# Exemplo de política de tags obrigatórias
resource "aws_iam_policy" "require_tags" {
  name        = "require-resource-tags"
  description = "Require tags on all resources"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Deny"
        Action   = "*"
        Resource = "*"
        Condition = {
          "Null" = {
            "aws:RequestTag/environment" = "true"
            "aws:RequestTag/service"     = "true"
            "aws:RequestTag/cost-center" = "true"
            "aws:RequestTag/owner"       = "true"
          }
        }
      }
    ]
  })
}
```

### 2. Automação de Gerenciamento de Custos

#### Rightsizing Automatizado

```yaml
# Kubernetes HPA para escala automática baseada em demanda
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: consolidado-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: consolidado
  minReplicas: 5
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 20
        periodSeconds: 60
```

#### Agendamento de Recursos

```yaml
# Redução de recursos em períodos de baixa utilização
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: scheduled-scaling
spec:
  scaleTargetRef:
    name: non-critical-services
  triggers:
  - type: cron
    metadata:
      timezone: America/Sao_Paulo
      start: 30 19 * * *  # 19:30 - Reduzir escala
      end: 30 6 * * *     # 06:30 - Retornar escala normal
      desiredReplicas: "1"
```

### 3. Escolha de Instâncias e Recursos

#### Instâncias Spot para Cargas Não-Críticas

```hcl
# Terraform para configuração de node group com instâncias spot
resource "aws_eks_node_group" "spot_workers" {
  cluster_name    = aws_eks_cluster.xpto_cluster.name
  node_group_name = "spot-workers"
  node_role_arn   = aws_iam_role.eks_node_role.arn
  subnet_ids      = aws_subnet.private[*].id
  capacity_type   = "SPOT"  # Usando instâncias spot para economia
  
  scaling_config {
    desired_size = 3
    max_size     = 10
    min_size     = 1
  }

  labels = {
    workload_type = "non-critical"
  }

  tags = {
    Environment = "production"
    Service     = "batch-processing"
    CostCenter  = "finance"
    Owner       = "team-data"
  }
}
```

#### Instâncias Reservadas para Cargas Estáveis

```hcl
# Documentação de reserva de instâncias para workloads estáveis
# A reserva deve ser feita no console ou via API do provedor cloud
# Planejamento de instâncias reservadas:

# On-Demand Baseline: 30% da capacidade
# Reserved Instances: 50% da capacidade (1 ano de compromisso)
# Spot Instances: 20% da capacidade (workloads tolerantes a interrupções)
```

### 4. Monitoramento e Alertas de Custos

#### Dashboard Prometheus + Grafana

```yaml
# ConfigMap para Prometheus com métricas de custo
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-cost-metrics
data:
  prometheus.yml: |
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
      - job_name: 'kubecost'
        static_configs:
          - targets: ['kubecost-cost-analyzer.kubecost:9003']
```

#### Alertas de Anomalias de Custo

```yaml
# Alerta para gastos anômalos
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cost-anomaly-alerts
spec:
  groups:
  - name: cost.rules
    rules:
    - alert: DailyCostSpike
      expr: sum(increase(kubecost_cluster_costs_hourly[24h])) > sum(avg_over_time(kubecost_cluster_costs_hourly[7d])) * 1.3
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Aumento anômalo de custos detectado"
        description: "Os custos nas últimas 24h aumentaram mais de 30% em relação à média dos últimos 7 dias."
```

## Estratégias Específicas por Ambiente

### Ambiente On-Premises

1. **Otimização de Recursos Existentes**
   - Consolidação de servidores com virtualização
   - Implementação de sleep schedules para ambientes não-produtivos
   - Políticas de energy efficiency (desligamento programado)

2. **Extensão de Vida Útil**
   - Manutenção preventiva de hardware
   - Upgrades seletivos (memória, armazenamento)
   - Reuso de equipamentos para ambientes não-críticos

### Ambiente Cloud

1. **Gerenciamento de Compromissos**
   - Reserved Instances (RIs) para cargas de trabalho estáveis
   - Savings Plans para uso flexível de recursos
   - Spot Instances para workloads tolerantes a interrupções

2. **Otimização de Armazenamento**
   - Políticas de lifecycle para object storage
   - Compressão e deduplicação de dados
   - Classes de armazenamento por idade e frequência de acesso

```yaml
# Exemplo de política de lifecycle para S3
resource "aws_s3_bucket_lifecycle_configuration" "backup_lifecycle" {
  bucket = aws_s3_bucket.backup_bucket.id

  rule {
    id = "archive-old-backups"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }
}
```

## Métricas e KPIs de FinOps

| Métrica | Descrição | Meta | Frequência |
|---------|-----------|------|------------|
| Cost per Transaction | Custo total / nº transações | < R$ 0,002 | Diária |
| Utilização de Recursos | % utilização média CPU/RAM | > 65% | Horária |
| Elasticidade de Custos | % variação custo / % variação tráfego | < 0.8 | Semanal |
| Custo de Cloud vs On-Prem | Comparativo de TCO | Cloud < 90% On-Prem | Mensal |
| Economia por Otimização | R$ economizado por ação | > R$ 5.000/mês | Mensal |

## Relatórios e Revisões

### Relatórios Automatizados

- **Diários**: Alertas de anomalias e dashboard de gastos
- **Semanais**: Resumo de custos por serviço e oportunidades de otimização
- **Mensais**: Análise detalhada com tendências e recomendações

### Reuniões de Revisão

- **Mensais**: Review de métricas e ajustes imediatos (Ops + Dev)
- **Trimestrais**: Planejamento estratégico e ajustes arquiteturais
- **Anuais**: Revisão de compromissos e contratos (RIs, hardware)

## Ferramentas de FinOps

| Ferramenta | Propósito | Ambiente |
|------------|-----------|----------|
| Kubecost | Análise de custos Kubernetes | Híbrido |
| CloudHealth | Gestão de custos multi-cloud | Cloud |
| GreenIT Analytics | Eficiência energética | On-Premises |
| Custom Prometheus Exporters | Métricas customizadas | Híbrido |
| Terraform + Cost Estimation | Previsão pré-deployment | CI/CD |

## Roadmap de Otimização Contínua

### Fase 1: Visibilidade (Mês 1-2)
- Implementação de tags e alocação de custos
- Configuração de dashboards e relatórios

### Fase 2: Otimização (Mês 3-6)
- Rightsizing de recursos
- Implementação de políticas de autoscaling
- Análise e implementação de instâncias reservadas

### Fase 3: Automação (Mês 7-12)
- Detecção e correção automática de anomalias
- Previsão e ajuste dinâmico de recursos
- Integração de feedback de custos no pipeline de CI/CD
