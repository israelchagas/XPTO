# Arquitetura da Solução

## Visão Geral

A arquitetura proposta para a XPTO consiste em uma solução híbrida que integra o ambiente on-premises existente com serviços em nuvem, focando em alta disponibilidade, escalabilidade, segurança e otimização de custos. O objetivo principal é manter investimentos existentes enquanto se beneficia das vantagens da computação em nuvem.

## Abordagem Arquitetural

![Diagrama Conceitual](../images/diagrama_conceitual.png)

### Decisões Chave

1. **Containerização dos Serviços Existentes**
   - Os serviços atuais (Controle de Lançamentos e Consolidado Diário) serão containerizados para facilitar portabilidade
   - Uso do Kubernetes para orquestração, permitindo escalabilidade dinâmica

2. **Modelo Híbrido com Multi-Cloud**
   - Ambiente on-premises mantido para cargas de trabalho críticas e dados sensíveis
   - Cloud pública utilizada para escala elástica e serviços gerenciados
   - Estratégia multi-cloud para evitar lock-in de fornecedor e aumentar resiliência

3. **Padrões de Resiliência**
   - Circuit Breaker implementado para evitar falhas em cascata
   - Retry patterns para operações transitórias
   - Cache distribuído para reduzir latência e aumentar disponibilidade

4. **Dados Distribuídos**
   - Banco de dados primário on-premises (PostgreSQL)
   - Réplicas de leitura na nuvem para escala e disponibilidade
   - Sistema de mensageria (Kafka) para comunicação assíncrona

5. **Segurança em Camadas**
   - Princípio de menor privilégio em todo ambiente
   - Comunicação criptografada (TLS 1.3) entre todos os componentes
   - Segmentação de rede usando VPNs e Security Groups

## Serviços de Controle de Lançamentos

- Implementado como microserviço stateless
- Alta disponibilidade com múltiplas réplicas
- Prioridade para baixa latência de escrita

## Serviço de Consolidado Diário

- Arquitetura orientada a eventos
- Capacidade para processamento em lote com paralelização
- Escalabilidade horizontal para lidar com picos de demanda (50 req/s)

## Conectividade Híbrida

- Conexão dedicada entre ambiente on-premises e nuvem via VPN site-to-site
- Roteamento inteligente usando BGP
- Camadas de cache distribuído para reduzir tráfego entre ambientes

## Justificativas Tecnológicas

### Kubernetes vs. Serverless

Para esta solução, optamos por uma combinação de ambos:

- **Kubernetes**:
  - Utilizado para os serviços core com demanda previsível
  - Oferece maior controle sobre a infraestrutura
  - Permite portabilidade entre on-premises e cloud

- **Serverless**:
  - Utilizado para cargas de trabalho com picos imprevisíveis
  - Implementado para funções auxiliares (processamento de lotes, relatórios)
  - Reduz custos para operações intermitentes

### PostgreSQL vs. NoSQL

- Optamos pelo PostgreSQL como banco principal devido a:
  - Consistência transacional necessária para operações financeiras
  - Maturidade e confiabilidade do sistema
  - Suporte a replicação entre on-premises e cloud

- Redis implementado como cache distribuído para:
  - Reduzir latência de leitura
  - Aliviar carga do banco principal
  - Aumentar resiliência em caso de falhas

## Componentes da Solução

1. **Kubernetes Cluster Híbrido**
   - Cluster on-premises (3+ nós)
   - Cluster na nuvem (auto-scaling)
   - Federação para gestão unificada

2. **Rede e Segurança**
   - VPN site-to-site com redundância
   - Firewall de próxima geração
   - Web Application Firewall (WAF)

3. **Bancos de Dados**
   - PostgreSQL em alta disponibilidade
   - Redis para cache
   - Object Storage para backups

4. **Observabilidade**
   - Prometheus e Grafana para métricas
   - Elastic Stack para logs
   - Jaeger para tracing distribuído

5. **CI/CD**
   - GitOps com ArgoCD
   - Pipeline automatizado de CI/CD
   - Testes de infraestrutura

## Evoluções Futuras

1. Migração gradual para arquitetura de microserviços
2. Implementação de service mesh (Istio)
3. Adoção de práticas DevSecOps integradas ao pipeline
