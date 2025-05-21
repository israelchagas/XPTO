# Dimensionamento de Recursos

## Análise de Demanda

O dimensionamento para a solução híbrida foi baseado nos seguintes fatores:

1. **Requisitos Não-Funcionais**
   - Suporte para 50 requisições por segundo no serviço de consolidado diário
   - Máximo de 5% de perda de requisições em picos
   - Independência operacional entre serviços

2. **Padrões de Uso**
   - Operação normal: 15-20 requisições por segundo
   - Picos promocionais: até 50 requisições por segundo
   - Variação diária: aumento de 300% em horários de pico (9-11h e 14-16h)

3. **Crescimento Projetado**
   - Aumento estimado de 30% ao ano na demanda base
   - Planejamento para 3 anos (capacidade para ~110 req/s)

## Especificação de Recursos

### Ambiente On-Premises

#### Cluster Kubernetes (Primário)

| Componente | Quantidade | CPU | Memória | Disco | Justificativa |
|------------|------------|-----|---------|-------|---------------|
| Control Plane | 3 | 4 vCPUs | 8 GB | 100 GB SSD | Alta disponibilidade com quórum para etcd |
| Worker Nodes | 6 | 8 vCPUs | 32 GB | 200 GB SSD | Capacidade para operação normal com N+2 redundância |
| Bastion Host | 2 | 2 vCPUs | 4 GB | 50 GB | Acesso seguro com redundância |

#### Banco de Dados

| Componente | Quantidade | CPU | Memória | Disco | Justificativa |
|------------|------------|-----|---------|-------|---------------|
| PostgreSQL Master | 1 | 8 vCPUs | 32 GB | 1 TB NVMe | Carga primária de dados transacionais |
| PostgreSQL Standby | 2 | 8 vCPUs | 32 GB | 1 TB NVMe | Alta disponibilidade e distribuição de leitura |
| Redis Cache | 3 | 4 vCPUs | 16 GB | 100 GB SSD | Cluster distribuído para alta disponibilidade |

#### Rede e Segurança

| Componente | Quantidade | CPU | Memória | Capacidade | Justificativa |
|------------|------------|-----|---------|------------|---------------|
| Load Balancer | 2 | 4 vCPUs | 8 GB | 10 Gbps | Distribuição de carga com failover |
| Firewall | 2 | 8 vCPUs | 16 GB | 5 Gbps | Segurança perimetral com failover |
| VPN Gateway | 2 | 4 vCPUs | 8 GB | 1 Gbps | Conectividade híbrida redundante |

### Ambiente Cloud

#### Cluster Kubernetes

| Componente | Min | Max | CPU | Memória | Disco | Justificativa |
|------------|-----|-----|-----|---------|-------|---------------|
| Control Plane | 3 | 3 | 4 vCPUs | 8 GB | 100 GB SSD | Serviço gerenciado com alta disponibilidade |
| Worker Nodes | 3 | 12 | 8 vCPUs | 32 GB | 200 GB SSD | Auto-scaling para lidar com picos de demanda |

#### Serviços Gerenciados

| Componente | Tipo | Capacidade | Justificativa |
|------------|------|------------|---------------|
| PostgreSQL | Instância Gerenciada - Média | 8 vCPUs, 32 GB, 1 TB | Réplica do banco on-premises |
| Redis | Cluster Gerenciado - Pequeno | 4 vCPUs, 16 GB | Cache distribuído |
| Object Storage | Standard | 5 TB | Backups e armazenamento de longo prazo |
| Functions | Serverless | 128-2048 MB | Processos em lote sob demanda |

## Capacidade e Escala

### Serviço de Controle de Lançamentos

**Requisitos por Pod:**
- CPU: 1 vCPU (limite: 2 vCPUs)
- Memória: 2 GB (limite: 4 GB)
- Conexões de banco: 50 por instância
- Throughput médio: 10 req/s por pod

**Estratégia de Escala:**
- Método: Horizontal Pod Autoscaler (HPA)
- Métricas: CPU > 70%, Memória > 80%, Requisições > 8/s
- Mínimo: 3 pods (alta disponibilidade)
- Máximo: 10 pods
- Cool-down: 3 minutos após escala

### Serviço de Consolidado Diário

**Requisitos por Pod:**
- CPU: 2 vCPUs (limite: 4 vCPUs)
- Memória: 4 GB (limite: 8 GB)
- Conexões de banco: 20 por instância
- Capacidade de processamento: 5 req/s por pod

**Estratégia de Escala:**
- Método: Horizontal Pod Autoscaler (HPA)
- Métricas: CPU > 60%, Tamanho da fila > 100
- Mínimo: 5 pods (para garantir <=5% perda em picos)
- Máximo: 15 pods
- Cool-down: 5 minutos após escala

## Estratégia de Provisionamento Dinâmico

### Kubernetes Cluster Autoscaling

```yaml
# Configuração do Cluster Autoscaler
apiVersion: autoscaling.k8s.io/v1
kind: ClusterAutoscaler
metadata:
  name: xpto-production
spec:
  scaleDown:
    enabled: true
    delayAfterAdd: 10m
    delayAfterDelete: 10m
    delayAfterFailure: 3m
  resourceLimits:
    maxNodesTotal: 15
    cores:
      min: 24  # 3 nodes x 8 vCPUs
      max: 96  # 12 nodes x 8 vCPUs
    memory:
      min: 96  # 3 nodes x 32 GB
      max: 384 # 12 nodes x 32 GB
```

### Aplicação Autoscaling

```yaml
# Horizontal Pod Autoscaler - Lançamentos
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: lancamentos-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: lancamentos
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 8
```

## Modelo de Crescimento e Capacidade

### Projeção de Crescimento

| Período | Requisições/s (Base) | Requisições/s (Pico) | Armazenamento (TB) |
|---------|----------------------|----------------------|--------------------|
| Atual   | 20                   | 50                   | 1                  |
| Ano 1   | 26                   | 65                   | 1.4                |
| Ano 2   | 34                   | 85                   | 2.0                |
| Ano 3   | 44                   | 110                  | 2.8                |

### Estratégia de Capacidade

1. **Planejamento Proativo**
   - Revisão trimestral de métricas de utilização
   - Ajuste dos limites de autoscaling com base em tendências

2. **Expansão On-Premises**
   - Aumento anual de 30% na capacidade
   - Upgrades durante janelas de manutenção planejadas

3. **Flexibilidade Cloud**
   - Ajuste contínuo dos limites máximos de autoscaling
   - Utilização de instâncias spot para cargas não críticas

## Tabela de Dimensionamento para Dias de Pico

| Componente | Capacidade Normal | Capacidade em Pico | Método de Escala |
|------------|-------------------|-------------------|------------------|
| Lançamentos | 30 req/s (3 pods) | 100 req/s (10 pods) | Horizontal |
| Consolidado | 25 req/s (5 pods) | 75 req/s (15 pods) | Horizontal |
| DB Master | 200 conexões | 500 conexões | Vertical (pré-provisionado) |
| DB Replicas | 400 conexões | 1000 conexões | Adição de réplicas |
| Cache | 5,000 ops/s | 15,000 ops/s | Expansão de cluster |
| Rede | 1 Gbps | 5 Gbps | Burst capacity |

## Considerações de Performance

1. **Otimização de Banco de Dados**
   - Índices otimizados para consultas frequentes
   - Particionamento de tabelas para dados históricos
   - Connection pooling para gerenciamento eficiente de conexões

2. **Estratégias de Cache**
   - Cache em múltiplos níveis (aplicação, distribuído)
   - Políticas de expiração baseadas em padrões de uso
   - Pré-aquecimento para relatórios programados

3. **Fluxo de Tráfego**
   - Compressão para reduzir volume de dados
   - CDN para conteúdo estático
   - Balanceamento de carga com afinidade de sessão
