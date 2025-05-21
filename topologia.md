# Diagrama de Topologia

## Visão Completa da Infraestrutura

O diagrama abaixo representa a topologia completa da solução híbrida, incluindo todos os componentes on-premises e cloud, suas interconexões e fluxos de dados.

```
+-----------------------------------------------------------------------------------------------+
|                                     INFRAESTRUTURA HÍBRIDA                                    |
+-----------------------------------------------------------------------------------------------+

+-----------------------------------+                      +-----------------------------------+
|       ON-PREMISES (XPTO)          |                      |           CLOUD PÚBLICA           |
|                                   |                      |                                   |
| +-------------------------------+ |                      | +-------------------------------+ |
| |     Aplicação - Kubernetes   | |                      | |     Aplicação - Kubernetes   | |
| |                              | |                      | |                              | |
| | +------------+ +------------+| |                      | | +------------+ +------------+| |
| | |Lançamentos | |Consolidado || |                      | | |Lançamentos | |Consolidado || |
| | | (Primário) | | (Réplica)  || |  Cluster Federation  | | | (Réplica)  | | (Primário) || |
| | +------------+ +------------+| |<-------------------->| | +------------+ +------------+| |
| | |   Istio     Service Mesh  | |                      | | |   Istio     Service Mesh  | |
| +-------------------------------+ |                      | +-------------------------------+ |
|                |                  |                      |                |                  |
| +-------------------------------+ |                      | +-------------------------------+ |
| |       Data & Storage         | |                      | |       Data & Storage         | |
| |                              | |                      | |                              | |
| | +------------+ +------------+| |    Replicação DB     | | +------------+ +------------+| |
| | |PostgreSQL  | |   Redis    || |<-------------------->| | |PostgreSQL  | |   Redis    || |
| | | (Master)   | |  Cache     || |                      | | | (Replica)  | |  Cache     || |
| | +------------+ +------------+| |                      | | +------------+ +------------+| |
| |        |               |      | |                      | |        |               |      | |
| |  +------------+  +----------+ | |                      | |  +------------+  +----------+ | |
| |  | Storage NFS|  | Backup   | | |     Object Sync     | |  | Object     |  | Backup   | | |
| |  +------------+  +----------+ | |<-------------------->| |  | Storage    |  | Managed  | | |
| +-------------------------------+ |                      | +-------------------------------+ |
|                |                  |                      |                |                  |
| +-------------------------------+ |                      | +-------------------------------+ |
| |       Rede & Segurança       | |                      | |       Rede & Segurança       | |
| |                              | |                      | |                              | |
| | +------------+ +------------+| |   VPN / Direct       | | +------------+ +------------+| |
| | |Load Balancer| |  Firewall  || |<-------------------->| | |Cloud LB    | |Security   || |
| | |  (HAProxy)  | |  (PFSense) || |     Connection      | | |            | |Groups     || |
| | +------------+ +------------+| |                      | | +------------+ +------------+| |
| |        |               |      | |                      | |        |               |      | |
| |  +------------+  +----------+ | |                      | |  +------------+  +----------+ | |
| |  | VPN Gateway|  |Web App FW| | |                      | |  |Transit GW  |  |  WAF     | | |
| |  +------------+  +----------+ | |                      | |  +------------+  +----------+ | |
| +-------------------------------+ |                      | +-------------------------------+ |
|                |                  |                      |                |                  |
| +-------------------------------+ |                      | +-------------------------------+ |
| |    Monitoramento & DevOps    | |                      | |    Serverless & Funções      | |
| |                              | |                      | |                              | |
| | +------------+ +------------+| |                      | | +------------+ +------------+| |
| | |Prometheus  | |  Grafana   || |                      | | |Relatórios  | |Processamento|| |
| | |            | |            || |                      | | |Periódicos  | |de Lotes    || |
| | +------------+ +------------+| |                      | | +------------+ +------------+| |
| |        |               |      | |    Log Shipping     | |        |               |      | |
| |  +------------+  +----------+ | |<-------------------->| |  +------------+  +----------+ | |
| |  | Elastic    |  | ArgoCD   | | |                      | |  | ETL        |  | Event    | | |
| |  | Stack      |  |          | | |                      | |  | Pipeline   |  | Bus      | | |
| |  +------------+  +----------+ | |                      | |  +------------+  +----------+ | |
| +-------------------------------+ |                      | +-------------------------------+ |
+-----------------------------------+                      +-----------------------------------+

+-----------------------------------------------------------------------------------------------+
|                                     USUÁRIOS E ACESSOS                                        |
+-----------------------------------------------------------------------------------------------+
|                                                                                               |
|  +----------------+  +----------------+  +----------------+  +----------------+               |
|  | Web Interface  |  | API Clients    |  | Mobile Apps    |  | Admin Access   |               |
|  | (Load-balanced)|  | (SDK/Direct)   |  | (REST/GraphQL) |  | (VPN/Bastion)  |               |
|  +----------------+  +----------------+  +----------------+  +----------------+               |
|                                                                                               |
+-----------------------------------------------------------------------------------------------+
```

## Fluxos de Tráfego Principais

### 1. Fluxo de Usuário para Aplicação

```
Usuário → CDN → WAF → Load Balancer → Ingress Kubernetes → Service Mesh → Pod Aplicação
```

### 2. Comunicação Entre Serviços

```
Serviço Lançamentos → Service Mesh (Istio) → Circuit Breaker → Serviço Consolidado
```

### 3. Replicação de Dados

```
PostgreSQL Master (On-Prem) → Replicação Síncrona → PostgreSQL Standby (On-Prem)
                            → Replicação Assíncrona → PostgreSQL Replica (Cloud)
```

### 4. Sistemas de Cache

```
Aplicação → Redis Cache Local → Redis Cache Distribuído → Banco de Dados
```

### 5. Conexão Híbrida

```
On-Premises Datacenter → VPN/Direct Connect Gateway → Transit Gateway → Cloud VPC
```

## Segmentação de Rede

A solução implementa segmentação de rede avançada:

### On-Premises
- **DMZ**: 10.0.1.0/24 - Componentes expostos externamente
- **Application Tier**: 10.0.2.0/24 - Serviços de aplicação
- **Database Tier**: 10.0.3.0/24 - Bancos de dados e storages
- **Management**: 10.0.4.0/24 - Ferramentas de gestão e monitoramento

### Cloud
- **Public Subnet**: 172.16.1.0/24 - Load balancers e componentes expostos
- **Application Subnet**: 172.16.2.0/24 - Pods Kubernetes
- **Data Subnet**: 172.16.3.0/24 - Bancos de dados e caches
- **Management Subnet**: 172.16.4.0/24 - Ferramentas de gestão

## Zonas de Disponibilidade

A arquitetura implementa alta disponibilidade com:

- **On-Premises**: Distribuição entre dois datacenters físicos
- **Cloud**: Distribuição entre três zonas de disponibilidade

## Conectividade Híbrida Detalhada

- **VPN Site-to-Site**: Conexão criptografada entre ambientes
  - Redundância com múltiplos túneis
  - Throughput de 1Gbps por túnel

- **Direct Connect** (opcional): Para cargas de trabalho com maior demanda de banda
  - Conexão dedicada de 10Gbps
  - Baixa latência para operações críticas

## Nível de Proteção

Múltiplas camadas de proteção são implementadas em cada segmento:

1. **Perímetro**: Firewall NG + WAF + Anti-DDoS
2. **Rede**: Segmentação + Security Groups + NACLs
3. **Aplicação**: Service Mesh + mTLS + Políticas de RBAC
4. **Dados**: Criptografia em repouso e em trânsito + Controle de acesso granular

## Componentes Adicionais

- **DNS**: Sistema de DNS híbrido com Route53 e servidores internos
- **Certificados**: Gestão centralizada com Cert Manager
- **Bastion Hosts**: Acesso administrativo controlado
