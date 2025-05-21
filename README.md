# Solução de Arquitetura Híbrida para XPTO - Fluxo de Caixa

## Visão Geral

Este repositório contém a arquitetura e documentação completa para a migração do sistema de fluxo de caixa da empresa XPTO para um ambiente híbrido (on-premises + cloud), garantindo alta disponibilidade, segurança, otimização de custos, resiliência e automação.

## Problema de Negócio

A empresa XPTO está em processo de transformação digital e deseja migrar para um ambiente híbrido, mantendo parte de seus investimentos on-premises enquanto aproveita os benefícios da nuvem. O produto atual de fluxo de caixa possui dois serviços principais:

- **Serviço de Controle de Lançamentos**: Gerencia todas as transações financeiras
- **Serviço de Consolidado Diário**: Processa e consolida as transações diárias

## Conteúdo do Repositório

- [Arquitetura da Solução](arquitetura.md) - Visão geral e justificativas de design
- [Diagrama de Topologia](./docs/topologia.md) - Diagrama detalhado da infraestrutura
- [Dimensionamento de Recursos](./docs/dimensionamento.md) - Especificações técnicas de hardware e capacidade
- [Estratégia FinOps](./docs/finops.md) - Abordagem para otimização de custos
- [Infrastructure as Code](./infra) - Código Terraform e Ansible para automação
- [Plano de Disaster Recovery](./docs/dr.md) - Estratégias de recuperação e continuidade
- [Monitoramento e Observabilidade](./docs/monitoramento.md) - Ferramentas e abordagens
- [Segurança e Modelo OSI](./docs/seguranca.md) - Implementação de segurança em todas as camadas

## Requisitos Atendidos

### Obrigatórios
- ✅ Dimensionamento de recursos (CPU, memória, escala)
- ✅ Metodologia FinOps aplicada
- ✅ Diagrama de topologia completo
- ✅ Justificativas detalhadas para escolhas tecnológicas
- ✅ Estratégia de automação com Terraform e Ansible

### Diferenciais
- ✅ Plano de Disaster Recovery com RTO e RPO definidos
- ✅ Monitoramento e observabilidade de rede
- ✅ Integração com Modelo OSI

### Não-Funcionais
- ✅ Independência entre serviços (lançamento continua mesmo se consolidado falhar)
- ✅ Performance em picos (50 req/s com menos de 5% de perda)

## Como Navegar este Repositório

1. Comece com a [Arquitetura da Solução](./docs/arquitetura.md) para entender a visão geral
2. Explore o [Diagrama de Topologia](./docs/topologia.md) para visualizar a implementação
3. Verifique o código IaC na pasta [infra](./infra) para detalhes de implementação
4. Confira as estratégias de [FinOps](./docs/finops.md) e [Disaster Recovery](./docs/dr.md)



