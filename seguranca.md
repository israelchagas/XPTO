# Segurança e Modelo OSI

## Visão Geral

Este documento detalha a estratégia de segurança para a infraestrutura híbrida da XPTO, estruturada de acordo com as camadas do Modelo OSI. A abordagem implementa o conceito de "defesa em profundidade", garantindo proteção em múltiplas camadas e minimizando a superfície de ataque.

## Modelo OSI como Framework de Segurança

O Modelo OSI (Open Systems Interconnection) com suas sete camadas serve como framework para implementação de controles de segurança.

### Segurança por Camada OSI

```
+---------------------------------------------+
|           CAMADA 7 - APLICAÇÃO              |
|                                             |
| • WAF (Web Application Firewall)            |
| • Autenticação Multifator                   |
| • API Gateway com Rate Limiting             |
| • Sanitização de Input                      |
+---------------------------------------------+
                    ↓
+---------------------------------------------+
|          CAMADA 6 - APRESENTAÇÃO            |
|                                             |
| • Criptografia TLS 1.3                      |
| • Certificados X.509                        |
| • Compressão Segura                         |
| • Content Security Policy                   |
+---------------------------------------------+
                    ↓
+---------------------------------------------+
|             CAMADA 5 - SESSÃO               |
|                                             |
| • Gerenciamento de Sessão                   |
| • Token JWT com Rotação                     |
| • Timeout de Inatividade                    |
| • Validação de Origem                       |
+---------------------------------------------+
                    ↓
+---------------------------------------------+
|           CAMADA 4 - TRANSPORTE             |
|                                             |
| • Firewalls Stateful                        |
| • TCP SYN Flood Protection                  |
| • DoS/DDoS Mitigation                       |
| • Connection Rate Limiting                  |
+---------------------------------------------+
                    ↓
+---------------------------------------------+
|              CAMADA 3 - REDE                |
|                                             |
| • Segmentação via VLANs/Subnets             |
| • ACLs em Roteadores                        |
| • VPN Site-to-Site                          |
| • Monitoramento NetFlow                     |
+---------------------------------------------+
                    ↓
+---------------------------------------------+
|             CAMADA 2 - ENLACE               |
|                                             |
| • MAC Filtering                             |
| • Proteção ARP Spoofing                     |
| • Port Security                             |
| • Proteção contra Broadcast Storm           |
+---------------------------------------------+
                    ↓
+---------------------------------------------+
|              CAMADA 1 - FÍSICA              |
|                                             |
| • Segurança Física de Datacenter            |
| • Redundância de Enlaces                    |
| • Proteção contra Interferência             |
| • Controle de Acesso Físico                 |
+---------------------------------------------+
```

## Implementação Detalhada por Camada

### Camada 1 - Física

#### Controles Implementados

1. **Segurança Física do Datacenter**
   - Acesso biométrico multi-fator
   - CCTV com gravação 24/7
   - Detecção de intrusão

2. **Redundância de Enlaces**
   - Múltiplos provedores de internet
   - Diversidade de rotas físicas
   - UPS e geração de energia backup

#### Benefícios para o Sistema de Fluxo de Caixa
- Garantia de disponibilidade física dos servidores
- Proteção contra interrupções de energia
- Mitigação de ataques físicos

#### Implementação Técnica

```bash
# Configuração de Monitoramento de Porta de Switch
# (Cisco Switch)
switch(config)# interface GigabitEthernet1/0/1
switch(config-if)# description Servidor-Lancamentos-1
switch(config-if)# spanning-tree portfast
switch(config-if)# spanning-tree bpduguard enable
switch(config-if)# storm-control broadcast level 20
switch(config-if)# storm-control action shutdown
```

### Camada 2 - Enlace

#### Controles Implementados

1. **Port Security**
   - Limitação de MACs por porta
   - Sticky MAC para servidores críticos
   - Desativação de portas não utilizadas

2. **Proteção contra ARP Spoofing**
   - Dynamic ARP Inspection (DAI)
   - DHCP Snooping
   - MAC Binding estático para equipamentos críticos

#### Benefícios para o Sistema de Fluxo de Caixa
- Prevenção de sniffing e man-in-the-middle
- Estabilidade de rede no nível de acesso
- Isolamento em caso de comprometimento

#### Implementação Técnica

```bash
# Configuração de Port Security
switch(config)# interface range GigabitEthernet1/0/1-24
switch(config-if-range)# switchport port-security
switch(config-if-range)# switchport port-security maximum 2
switch(config-if-range)# switchport port-security violation restrict
switch(config-if-range)# switchport port-security aging time 60
switch(config-if-range)# switchport port-security aging type inactivity

# DHCP Snooping
switch(config)# ip dhcp snooping
switch(config)# ip dhcp snooping vlan 10,20,30
switch(config)# no ip dhcp snooping information option
switch(config)# interface GigabitEthernet1/0/24
switch(config-if)# ip dhcp snooping trust
```

### Camada 3 - Rede

#### Controles Implementados

1. **Segmentação de Rede**
   - Micro-segmentação com VLANs específicas por função
   - Subnets isoladas para ambientes críticos
   - Zone-based Firewall entre segmentos

2. **VPN Site-to-Site**
   - IPsec com AES-256 para tráfego entre on-premises e cloud
   - Autenticação baseada em certificados
   - Perfect Forward Secrecy (PFS)

3. **Roteamento Seguro**
   - BGP com autenticação
   - Filtragem de rotas (Route Maps)
   - Prevenção contra route hijacking

#### Benefícios para o Sistema de Fluxo de Caixa
- Compartimentalização de ambiente
- Transmissão segura entre on-premises e cloud
- Mitigação de ataques de roteamento

#### Implementação Técnica

```
# IPSec VPN Configuration (Cisco)
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 24
 lifetime 86400

crypto ipsec transform-set TS esp-aes 256 esp-sha-hmac
 mode tunnel

crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set TS
 set pfs group24
 match address 110

interface GigabitEthernet0/0
 crypto map CMAP

# ACL para controle de tráfego
access-list 110 permit ip 10.0.1.0 0.0.0.255 172.16.1.0 0.0.0.255
```

### Camada 4 - Transporte

#### Controles Implementados

1. **Firewall Stateful**
   - Inspeção de estado de conexões
   - Proteção contra varredura de portas
   - Controle por aplicação

2. **Proteção DDoS**
   - Rate limiting por IP/subnet
   - TCP SYN cookies
   - Threshold-based Connection Limiting

3. **Load Balancing Seguro**
   - HA Proxy com mode tcp strict
   - TCP Health Checks
   - TLS Offloading com Perfect Forward Secrecy

#### Benefícios para o Sistema de Fluxo de Caixa
- Proteção contra ataques volumétricos
- Resiliência a ataques de negação de serviço
- Alta disponibilidade em casos de ataque

#### Implementação Técnica

```
# HAProxy Config para Load Balancing Seguro
frontend ft_http
    bind *:80
    redirect scheme https code 301 if !{ ssl_fc }

frontend ft_https
    bind *:443 ssl crt /etc/ssl/certs/xpto.pem alpn h2,http/1.1
    option forwardfor
    http-request deny if { src_conn_rate(3) gt 30 }
    http-request deny if { src_conn_cur gt 20 }
    use_backend bk_lancamentos if { path_beg /api/lancamentos }
    use_backend bk_consolidado if { path_beg /api/consolidado }
    default_backend bk_default

backend bk_lancamentos
    balance roundrobin
    option httpchk GET /health
    server srv1 10.0.2.10:8080 check ssl verify
    server srv2 10.0.2.11:8080 check ssl verify
    server srv3 10.0.2.12:8080 check ssl verify
```

### Camada 5 - Sessão

#### Controles Implementados

1. **Gerenciamento de Sessão**
   - Tokens JWT assinados e criptografados
   - Rotação automática de tokens
   - Validação de origem da sessão

2. **TimeOut Automático**
   - Sessões administrativas: 15 minutos
   - Sessões de usuário: 60 minutos
   - Sessões de API: baseado em token com max 24h

3. **Circuit Breaking**
   - Proteção contra cascata de falhas
   - Retry budgets configurados
   - Fallback para operações críticas

#### Benefícios para o Sistema de Fluxo de Caixa
- Prevenção de session hijacking
- Proteção contra replay attacks
- Resiliência em cenários de falha parcial

#### Implementação Técnica

```yaml
# Configuração Istio para Circuit Breaking
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker-lancamentos
spec:
  host: lancamentos-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 10
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

### Camada 6 - Apresentação

#### Controles Implementados

1. **Criptografia TLS**
   - TLS 1.3 para todas comunicações
   - Certificados X.509 com no mínimo 2048 bits
   - Rotação de certificados a cada 90 dias

2. **Headers de Segurança**
   - Content-Security-Policy
   - Strict-Transport-Security
   - X-Content-Type-Options

3. **Gestão de Certificados**
   - Autoridade certificadora privada
   - Auto-renovação via cert-manager
   - Monitoramento de expiração

#### Benefícios para o Sistema de Fluxo de Caixa
- Comunicação criptografada end-to-end
- Proteção contra ataques de downgrade
- Integridade da camada de apresentação

#### Implementação Técnica

```yaml
# Cert-Manager para Gestão Automatizada de Certificados
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: lancamentos-cert
  namespace: financial
spec:
  secretName: lancamentos-tls
  duration: 2160h # 90 days
  renewBefore: 360h # 15 days
  subject:
    organizations:
      - XPTO Financial
  privateKey:
    algorithm: RSA
    size: 2048
  usages:
    - server auth
    - client auth
  dnsNames:
    - lancamentos.xpto.com
    - lancamentos.service.internal
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
```

```nginx
# Configuração Nginx com Headers de Segurança
server {
    listen 443 ssl http2;
    server_name lancamentos.xpto.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    ssl_session_timeout 10m;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
}
```

### Camada 7 - Aplicação

#### Controles Implementados

1. **Web Application Firewall (WAF)**
   - Proteção contra OWASP Top 10
   - Regras customizadas para proteção de APIs
   - Prevenção contra ataques de injeção

2. **Autenticação e Autorização**
   - OAuth 2.0 com OpenID Connect
   - RBAC (Role-Based Access Control)
   - MFA para acesso administrativo

3. **API Security**
   - API Gateway com validação de schema
   - Rate limiting por usuário/IP
   - Validação de payload e sanitização

#### Benefícios para o Sistema de Fluxo de Caixa
- Proteção contra ataques específicos de aplicação
- Controle granular de acesso
- Prevenção de fraudes e abusos

#### Implementação Técnica

```yaml
# Kong API Gateway Config para Rate Limiting
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting
config:
  minute: 60
  hour: 1000
  policy: local
  fault_tolerant: true
  hide_client_headers: false
  redis_ssl: true
  redis_ssl_verify: true
  redis_server_name: redis.xpto.internal
```

```yaml
# Keycloak OIDC Configuration (excerpt)
apiVersion: keycloak.org/v1alpha1
kind: KeycloakRealm
metadata:
  name: xpto-financial
  namespace: security
spec:
  realm:
    id: financial
    realm: financial
    enabled: true
    displayName: XPTO Financial
    sslRequired: "external"
    registrationAllowed: false
    passwordPolicy: "length(10) and forceExpiredPasswordChange(365) and notUsername"
    bruteForceProtected: true
    roles:
      realm:
        - name: financial-admin
        - name: lancamentos-user
        - name: consolidado-user
        - name: report-viewer
    clients:
      - clientId: lancamentos-api
        protocol: openid-connect
        standardFlowEnabled: true
        implicitFlowEnabled: false
        directAccessGrantsEnabled: false
        serviceAccountsEnabled: true
        authorizationServicesEnabled: true
        redirectUris:
          - "https://lancamentos.xpto.com/*"
```

## Proteção Específica para Serviços Financeiros

### Controles para Sistema de Lançamentos

1. **Validação de Transações**
   - Detecção de padrões anômalos
   - Validação multi-nível de transações
   - Logging de auditoria imutável

2. **Criptografia de Dados Sensíveis**
   - Criptografia em repouso (AES-256)
   - Tokenização de dados sensíveis
   - Segregação de chaves criptográficas

### Controles para Sistema de Consolidado Diário

1. **Integridade de Dados**
   - Assinatura digital de relatórios
   - Checksums para validação
   - Reconciliação automática

2. **Controle de Acesso**
   - Segregação de funções
   - Just-In-Time Access para relatórios
   - Auditoria de visualização

## Automação de Segurança (SecOps)

### Pipeline de CI/CD Seguro

```
+---------------------------------------------------+
|              Secure CI/CD Pipeline                |
+---------------------------------------------------+
|                                                   |
| +-------------+  +-----------+  +---------------+ |
| | Code Commit |->| SAST Scan |->| Dependency    | |
| |             |  |           |  | Vulnerability | |
| +-------------+  +-----------+  +---------------+ |
|        |                                 |        |
|        v                                 v        |
| +-------------+  +-----------+  +---------------+ |
| | Build       |->| Container |->| Image         | |
| | Application |  | Scan      |  | Signing       | |
| +-------------+  +-----------+  +---------------+ |
|        |                                 |        |
|        v                                 v        |
| +-------------+  +-----------+  +---------------+ |
| | Deploy to   |->| DAST Scan |->| Security      | |
| | Test Env    |  |           |  | Validation    | |
| +-------------+  +-----------+  +---------------+ |
|        |                                 |        |
|        v                                 v        |
| +-------------+  +-----------+  +---------------+ |
| | Approval    |->| Deploy    |->| Continuous    | |
| | Gate        |  | Production|  | Monitoring    | |
| +-------------+  +-----------+  +---------------+ |
|                                                   |
+---------------------------------------------------+
```

### Implementação de DevSecOps

1. **Verificações de Segurança Automatizadas**
   - SAST (Static Application Security Testing)
   - DAST (Dynamic Application Security Testing)
   - SCA (Software Composition Analysis)

2. **Policy as Code**
   - Open Policy Agent (OPA) para validação de compliance
   - Políticas de segurança versionadas
   - Validação contínua

```yaml
# Open Policy Agent para Kubernetes
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-default-system-main
  namespace: opa
data:
  main: |
    package system

    import data.kubernetes.admission

    main = {
      "apiVersion": "admission.k8s.io/v1",
      "kind": "AdmissionReview",
      "response": response,
    }

    default response = {"allowed": true}

    response = {
      "allowed": false,
      "status": {"reason": reason},
    } {
      reason = concat(", ", admission.deny)
      reason != ""
    }
```

## Gestão de Vulnerabilidades

### Processo de Ciclo de Vida

1. **Descoberta**
   - Scans automatizados semanais
   - Monitoramento de CVEs
   - Bug Bounty Program

2. **Avaliação e Priorização**
   - CVSS Score + impacto ao negócio
   - Classificação por criticidade
   - SLAs por nível de severidade

3. **Remediação**
   - Patching automatizado para não-críticos
   - Janelas de manutenção programadas
   - Virtual patching via WAF quando necessário

### Política de Atualizações

| Nível | SLA Remediação | Abordagem | Aprovação |
|-------|----------------|-----------|-----------|
| Crítico | 24 horas | Emergencial | CISO |
| Alto | 7 dias | Planejada | Security Manager |
| Médio | 30 dias | Janela Regular | Security Lead |
| Baixo | 90 dias | Consolidada | Automática |

## Modelo de Ameaças

### Atores de Ameaças Considerados

1. **Externos**
   - Hackers oportunistas
   - Competidores
   - Crime organizado

2. **Internos**
   - Usuários privilegiados
   - Funcionários insatisfeitos
   - Contratados temporários

### Mitigações por Tipo de Ameaça

#### Roubo de Credenciais
- MFA para todas contas
- Rotação automática de segredos
- Detecção de anomalias de login

#### Exfiltração de Dados
- DLP (Data Loss Prevention)
- Monitoramento de tráfego sainte
- Encryption-in-use com confidential computing

#### Ataques de Ransomware
- Backups imutáveis
- Microsegmentação
- Detecção comportamental

## Resposta a Incidentes

### Playbooks para Cenários Comuns

1. **Detecção de Intrusão**
   - Isolamento de host/container afetado
   - Coleta de evidências forenses
   - Análise de indicadores de comprometimento

2. **Exposição de Dados**
   - Contenção imediata da fonte
   - Avaliação de impacto
   - Comunicação a stakeholders

3. **Ataque de Negação de Serviço**
   - Ativação de mitigação DDoS
   - Filtros de tráfego adicionais
   - Escalonamento para provedores

### Simulações e Testes

- Red Team trimestral
- Blue Team mensal
- Tabletop Exercises bimestrais

## Conformidade e Auditoria

### Frameworks Aplicáveis

- PCI-DSS para processamento de pagamentos
- ISO 27001 para gestão de segurança
- SOC 2 Type II para controles operacionais

### Logs de Auditoria

- Retenção mínima de 1 ano
- Implementação Write-Once-Read-Many (WORM)
- Timestamping verificável

```yaml
# Auditd Configuration para Log de Segurança
- name: Configure auditd
  lineinfile:
    dest: /etc/audit/auditd.conf
    regexp: '^{{ item.parameter }}='
    line: '{{ item.parameter }} = {{ item.value }}'
  with_items:
    - { parameter: 'log_file', value: '/var/log/audit/audit.log' }
    - { parameter: 'max_log_file', value: '100' }
    - { parameter: 'max_log_file_action', value: 'keep_logs' }
    - { parameter: 'space_left', value: '75' }
    - { parameter: 'space_left_action', value: 'email' }
    - { parameter: 'action_mail_acct', value: 'security@xpto.com' }
    - { parameter: 'admin_space_left', value: '50' }
    - { parameter: 'admin_space_left_action', value: 'halt' }
    - { parameter: 'disk_full_action', value: 'halt' }
    - { parameter: 'disk_error_action', value: 'halt' }
```

## Zero Trust Architecture

### Princípios Implementados

1. **Verificação Contínua**
   - Autenticação contínua baseada em risco
   - Verificação de postura de dispositivos
   - Autorização por transação

2. **Menor Privilégio**
   - Just-In-Time Access (JIT)
   - Temporary Elevated Access
   - Função baseada em contexto

3. **Segmentação Micro-Perimetral**
   - Service Mesh com mTLS obrigatório
   - Network Policies no Kubernetes
   - East-West Traffic Inspection

```yaml
# Network Policy Kubernetes para Micro-segmentação
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lancamentos-network-policy
  namespace: financial
spec:
  podSelector:
    matchLabels:
      app: lancamentos
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-system
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          name: financial
    - podSelector:
        matchLabels:
          app: consolidado
    ports:
    - protocol: TCP
      port: 8090
```

## Segurança como Código

### Terraform para Infraestrutura Segura

```hcl
# Security Group para Database Tier
resource "aws_security_group" "db_security_group" {
  name        = "db-security-group"
  description = "Security group for database tier"
  vpc_id      = aws_vpc.main.id

  # Permitir acesso apenas da camada de aplicação
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app_security_group.id]
  }

  # Permitir comunicação entre instâncias de banco para replicação
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    self            = true
  }

  # Monitoramento e backup
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "db-security-group"
    Environment = "production"
    Service     = "fluxo-caixa"
  }
}

# KMS para Gerenciamento de Chaves de Criptografia
resource "aws_kms_key" "database_encryption_key" {
  description             = "KMS key for database encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  policy                  = data.aws_iam_policy_document.kms_policy.json

  tags = {
    Name        = "database-encryption-key"
    Environment = "production"
    Service     = "fluxo-caixa"
  }
}

# Política de KMS com princípio de menor privilégio
data "aws_iam_policy_document" "kms_policy" {
  statement {
    sid    = "Enable IAM User Permissions"
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"]
    }
    actions   = ["kms:*"]
    resources = ["*"]
  }

  statement {
    sid    = "Allow access for Key Administrators"
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = local.key_admin_arns
    }
    actions = [
      "kms:Create*",
      "kms:Describe*",
      "kms:Enable*",
      "kms:List*",
      "kms:Put*",
      "kms:Update*",
      "kms:Revoke*",
      "kms:Disable*",
      "kms:Get*",
      "kms:Delete*",
      "kms:TagResource",
      "kms:UntagResource"
    ]
    resources = ["*"]
  }

  statement {
    sid    = "Allow use of the key for database service"
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = local.db_service_arns
    }
    actions = [
      "kms:Encrypt",
      "kms:Decrypt",
      "kms:ReEncrypt*",
      "kms:GenerateDataKey*",
      "kms:DescribeKey"
    ]
    resources = ["*"]
  }
}
```

## Conclusão

A abordagem de segurança baseada no modelo OSI proporciona proteção em profundidade para toda a infraestrutura híbrida da XPTO, garantindo que cada camada tenha controles específicos e eficazes para mitigar ameaças. Esta implementação:

1. **Protege os dados financeiros sensíveis** através de múltiplas camadas de defesa
2. **Garante disponibilidade dos serviços críticos** mesmo sob ataques ou condições adversas
3. **Viabiliza a conformidade regulatória** com padrões internacionais
4. **Permite operação segura em ambiente híbrido** mantendo consistência entre on-premises e cloud

A segurança é um processo contínuo, e esta arquitetura será continuamente aprimorada através de revisões periódicas, testes de penetração, exercícios de red team e atualizações baseadas em novas ameaças identificadas.

  
