# Plano de Disaster Recovery (DR)

## Objetivos de Recuperação

O plano de Disaster Recovery (DR) para a infraestrutura híbrida da XPTO foi desenhado para garantir a continuidade dos negócios em caso de falhas ou desastres, com os seguintes objetivos críticos:

| Serviço | RTO (Recovery Time Objective) | RPO (Recovery Point Objective) | Prioridade |
|---------|-------------------------------|-------------------------------|------------|
| Controle de Lançamentos | 15 minutos | 5 minutos | Alta |
| Consolidado Diário | 30 minutos | 15 minutos | Média-Alta |
| Banco de Dados | 30 minutos | < 1 minuto | Alta |
| Infraestrutura de Rede | 10 minutos | N/A | Alta |
| Serviços de Backup | 60 minutos | 24 horas | Média |

## Cenários de Falha e Estratégias

### Cenário 1: Falha em Componente de Aplicação

**Descrição**: Falha em pods individuais ou deployments inteiros nos serviços de Lançamentos ou Consolidado.

**Estratégia de Mitigação**:
1. **Redundância**: Mínimo de 3 réplicas por serviço distribuídas em nós diferentes
2. **Health Checks**: Verificações de saúde (liveness/readiness) em cada pod
3. **Circuit Breaker**: Implementação de padrão Circuit Breaker para evitar falhas em cascata

**Estratégia de Recuperação**:
1. Kubernetes realiza recuperação automática através do Deployment Controller
2. Pod falho é terminado e substituído por nova instância
3. Service e Ingress redirecionam tráfego para pods saudáveis durante recuperação

**Código de Implementação (Kubernetes)**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lancamentos
  namespace: financial
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lancamentos
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: lancamentos
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - lancamentos
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: lancamentos
        image: xpto/lancamentos:v1.2.3
        resources:
          limits:
            cpu: "2"
            memory: "4Gi"
          requests:
            cpu: "1"
            memory: "2Gi"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
```

### Cenário 2: Falha em Nó do Kubernetes

**Descrição**: Falha completa em um ou mais nós de um cluster Kubernetes (on-premises ou cloud).

**Estratégia de Mitigação**:
1. **Redundância de Nós**: Mínimo N+2 nós em cada cluster
2. **Anti-Afinidade**: Distribuição de pods em diferentes nós/zonas
3. **Node Auto-healing**: Recuperação automática de nós via ferramentas de automação

**Estratégia de Recuperação**:
1. Kubernetes redistribui pods em outros nós disponíveis
2. Auto-scaling do cluster adiciona novos nós se necessário
3. Node auto-healing inicia recuperação do nó falho

**Tempo Médio de Recuperação**: 5-10 minutos

### Cenário 3: Falha de Banco de Dados

**Descrição**: Falha do servidor de banco de dados primário (PostgreSQL).

**Estratégia de Mitigação**:
1. **Replicação Síncrona**: Para banco primário on-premises
2. **Replicação Assíncrona**: Para banco secundário na nuvem
3. **Transaction Log Shipping**: Contínuo para minimizar perda de dados

**Estratégia de Recuperação**:
1. **Failover Automático**: Promoção do standby síncrono a master
2. **Reconfiguração de Conexão**: Aplicações redirecionadas via DNS/Service Discovery
3. **Recuperação do Primário**: Reconstrução do primário falho como novo standby

```bash
# Exemplo de configuração para Patroni (PostgreSQL HA)
cat > /etc/patroni/config.yml << EOF
scope: xpto-production
namespace: /db/
name: postgresql-node-01

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.3.10:8008

etcd:
  hosts: 10.0.3.20:2379,10.0.3.21:2379,10.0.3.22:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 500
        shared_buffers: 8GB
        work_mem: 16MB
        maintenance_work_mem: 1GB
        
  initdb:
  - encoding: UTF8
  - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.3.10:5432
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/14/bin
  
  synchronous_mode: true
  synchronous_node_count: 1
  
  authentication:
    replication:
      username: replicator
      password: "xxxxxxxx"
    superuser:
      username: postgres
      password: "xxxxxxxx"
EOF
```

### Cenário 4: Perda de Conectividade Híbrida

**Descrição**: Falha na conexão VPN/Direct Connect entre ambiente on-premises e cloud.

**Estratégia de Mitigação**:
1. **Conexões Redundantes**: Múltiplos túneis VPN por diferentes rotas
2. **Monitoramento Proativo**: Verificação contínua da latência e disponibilidade
3. **Modo de Operação Desconectado**: Capacidade de funcionamento isolado

**Estratégia de Recuperação**:
1. **Failover de Conectividade**: Ativação automática de conexão secundária
2. **Modo de Operação Degradado**: Priorização de operações críticas
3. **Replicação Pendente**: Acumulação de operações para sincronização posterior

**Diagrama de Recuperação**:
```
+------------------------+                +------------------------+
|  Conectividade Normal  |                |  Conectividade Falha   |
+------------------------+                +------------------------+
| On-Prem <---> Cloud    |     ---->     | On-Prem   X   Cloud    |
| (Túnel VPN Principal)  |                | (Falha de Túnel)       |
+------------------------+                +------------------------+
           |                                        |
           v                                        v
+------------------------+                +------------------------+
| Detecção de Problema   |                | Operação Independente  |
| - Monitoramento VPN    |                | - Lançamentos On-Prem  |
| - Alertas              |                | - Consolidado Cloud    |
+------------------------+                +------------------------+
           |                                        |
           v                                        v
+------------------------+                +------------------------+
| Ativação Contingência  |                | Restauração Conexão    |
| - Túnel VPN Secundário |                | - Reparo Automático    |
| - Rota Alternativa     |                | - Intervenção Manual   |
+------------------------+                +------------------------+
           |                                        |
           v                                        v
+------------------------+                +------------------------+
| Operação Recuperada    |                | Sincronização Dados    |
| - Conexão Restabelecida|                | - Log de Transações    |
| - Tráfego Normalizado  |                | - Resolução Conflitos  |
+------------------------+                +------------------------+
```

### Cenário 5: Desastre Completo em Datacenter On-Premises

**Descrição**: Perda total do datacenter on-premises (incêndio, inundação, catástrofe natural).

**Estratégia de Mitigação**:
1. **Ambiente Cloud Completo**: Réplica da infraestrutura crítica na nuvem
2. **Backups Off-site**: Replicação de dados para cloud e localidade secundária
3. **Documentação Detalhada**: Procedimentos de DR atualizados e testados

**Estratégia de Recuperação**:
1. **Ativação do Site DR**: Promoção do ambiente cloud a produção
2. **Reconfiguração de DNS**: Atualização de entradas DNS para novos endpoints
3. **Recuperação Gradual**: Priorização de serviços críticos, seguido por não-críticos

**Tempo de Recuperação**:
- Serviços Críticos: 1-2 horas
- Recuperação Completa: 24-48 horas

## Estratégias de Backup

### Banco de Dados

| Tipo | Frequência | Retenção | Armazenamento | Validação |
|------|------------|----------|--------------|-----------|
| Full Backup | Diário | 30 dias | Object Storage + Tape | Semanal |
| Incremental | 6 horas | 7 dias | Object Storage | Diária |
| Transaction Logs | Contínuo | 7 dias | Replicado | Real-time |

**Script de Backup PostgreSQL**:
```bash
#!/bin/bash
# Backup PostgreSQL - Executado via cron

DATE=$(date +%Y%m%d_%H%M)
BACKUP_DIR="/backups/postgresql"
S3_BUCKET="s3://xpto-backups/postgresql"

# Full backup com pg_basebackup
pg_basebackup -D ${BACKUP_DIR}/full_${DATE} -Ft -z -P -U replicator

# Compactar backup
tar -czf ${BACKUP_DIR}/full_${DATE}.tar.gz ${BACKUP_DIR}/full_${DATE}
rm -rf ${BACKUP_DIR}/full_${DATE}

# Upload para Object Storage
aws s3 cp ${BACKUP_DIR}/full_${DATE}.tar.gz ${S3_BUCKET}/full_${DATE}.tar.gz

# Limpeza de backups antigos (manter 30 dias)
find ${BACKUP_DIR} -name "full_*" -type f -mtime +30 -delete
aws s3 ls ${S3_BUCKET}/full_ --recursive | sort | head -n -30 | awk '{print $4}' | xargs -I {} aws s3 rm ${S3_BUCKET}/{}
```

### Aplicação e Configuração

| Componente | Método | Frequência | Armazenamento |
|------------|--------|------------|---------------|
| Código de Aplicação | GitOps/Git | Contínuo | Repositório Git |
| Configuração K8s | Velero | Diário | Object Storage |
| Secrets | Vault | Backup Semanal | Encriptado |
| Estado Terraform | State File | A cada alteração | Backend Remoto |

## Testes e Validação de DR

### Programação de Testes

| Tipo de Teste | Frequência | Escopo | Responsável |
|---------------|------------|--------|-------------|
| Recuperação de Backup | Semanal | Database, Configs | DBA, DevOps |
| Failover de Componente | Mensal | App, DB, Network | DevOps, SRE |
| Simulation Day | Trimestral | Sistema Completo | Todas Equipes |
| DR Completo | Semestral | Infraestrutura Completa | DR Team |

### Procedimentos de Teste

1. **Teste de Recuperação de Backup**
   - Recuperação em ambiente isolado
   - Validação de integridade de dados
   - Medição de tempo de recuperação

2. **Teste de Failover**
   - Simulação de falha controlada
   - Verificação de detecção automática
   - Validação de continuidade de serviço

3. **Simulation Day (SimDay)**
   - Injeção de múltiplas falhas
   - Exercício de procedimentos manuais
   - Colaboração entre equipes

4. **DR Completo**
   - Simulação de perda de datacenter
   - Ativação de site secundário
   - Validação de operação em contingência

## Organização e Responsabilidades

### Equipe de DR

| Papel | Responsabilidade | Contato Primário | Contato Secundário |
|-------|------------------|------------------|-------------------|
| DR Coordinator | Coordenação geral e comunicação | [TBD] | [TBD] |
| Infrastructure Lead | Recuperação de infraestrutura | [TBD] | [TBD] |
| Database Lead | Recuperação de dados | [TBD] | [TBD] |
| Application Lead | Verificação de aplicações | [TBD] | [TBD] |
| Security Lead | Validação de segurança | [TBD] | [TBD] |

### Procedimento de Ativação de DR

1. **Detecção e Alerta**
   - Sistemas de monitoramento detectam anomalia
   - Alerta enviado via múltiplos canais (email, SMS, Slack)
   - Avaliação inicial por equipe de plantão

2. **Avaliação e Declaração**
   - Análise da severidade e impacto
   - Declaração formal de incidente (ferramenta de IMS)
   - Acionamento da equipe de DR

3. **Mobilização**
   - Conference call de emergência (número dedicado)
   - Atribuição de responsabilidades
   - Comunicação para stakeholders

4. **Execução**
   - Ativação dos runbooks de recuperação
   - Updates regulares (30 minutos)
   - Monitoramento de progresso

5. **Estabilização e Retorno**
   - Validação de operação em contingência
   - Plano de retorno à operação normal
   - Comunicação de resolução

## Documentação e Procedimentos

### Runbooks de Recuperação

Todos os runbooks são mantidos em formato digital e impresso, armazenados em:
- **Digital**: Repositório Git, Wiki interna, Recovery Portal
- **Físico**: Pasta DR nos locais-chave, acesso offline

### Procedimentos Específicos

1. **[DR-DB-001]** - Failover de Banco de Dados PostgreSQL
2. **[DR-APP-001]** - Recuperação de Ambiente Kubernetes
3. **[DR-NET-001]** - Reestabelecimento de Conectividade Híbrida
4. **[DR-FULL-001]** - Ativação de Ambiente DR Completo

## Melhorias Contínuas do Plano de DR

### Roadmap de Evolução

| Fase | Objetivo | Timeline | Status |
|------|----------|----------|--------|
| 1 | Automação de backups e testes | Q3 2024 | Planejado |
| 2 | Redução de RTO/RPO | Q4 2024 | Planejado |
| 3 | DR como código | Q1 2025 | Planejado |
| 4 | Otimização de custos DR | Q2 2025 | Planejado |

### Aprendizados de Testes e Incidentes

Após cada teste de DR ou incidente real, um relatório é gerado com:
- Cronologia dos eventos
- Pontos de sucesso
- Oportunidades de melhoria
- Ações concretas para evolução do plano

## Anexos

1. **Contatos Emergenciais**
   - Equipe interna
   - Fornecedores (Cloud, Telecom, Hardware)
   - Stakeholders críticos

2. **Inventário de Recursos Críticos**
   - Servidores e suas funções
   - Bancos de dados
   - Endpoints e APIs

3. **Checklists de Validação**
   - Validação de banco de dados
   - Validação de aplicação
   - Validação de segurança
