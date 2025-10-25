# Lab de Observabilidade em Kubernetes

Projeto de estudo pra testar observabilidade em K8s com Prometheus gerenciado. A ideia é simples: receber alerta quando pod cai ou memória estoura, sem precisar ficar subindo Prometheus na mão.

## O que tem aqui

- Coleta de métricas rodando (GMP ou AMP)
- Alertas básicos (pod down, memória >80%)
- Email quando dispara
- Tudo gerenciado pela cloud (menos coisa pra quebrar)

## Por que gerenciado?

Porque gerenciar Prometheus no cluster é um saco:
- Precisa configurar storage persistente
- Escalabilidade vira problema rápido
- Se o Prometheus cai, você fica cego
- Backup é sua responsabilidade

Com gerenciado você paga mais, mas não precisa se preocupar com isso.

## Estrutura do projeto

```
.
├── README.md                    # você tá aqui
├── LEARNING.md                  # conceitos básicos de Prometheus
├── ARCHITECTURE.md              # arquitetura geral
├── COSTS.md                     # estimativa de custos
│
├── aws/                         # AWS (AMP + SNS)
│   ├── setup-guide.md
│   ├── iam-policies/
│   ├── prometheus-config/
│   └── k8s-manifests/
│
└── gcp/                         # GCP (GMP + SMTP)
    ├── setup-guide.md           # Passo a passo do setup
    ├── DEBUG.md                 # API endpoints e troubleshooting
    ├── GMP-COMPONENTS.md        # Como funciona cada componente do GMP
    ├── METRICS-SOURCES.md       # Métricas: kubelet vs aplicação
    └── k8s-manifests/
        ├── alertmanager-config.yaml
        ├── alerting-rules.yaml  # ✅ Query corrigida (sem absent!)
        ├── pod-monitoring.yaml
        └── sample-app.yaml
```

## Quick start

### Nunca mexeu com Prometheus?

Dá uma olhada no `LEARNING.md` antes. Explica os conceitos básicos: métricas, PromQL, alertas, etc.

### Já conhece?

**GCP:**
```bash
cd gcp
cat setup-guide.md       # ~15min, só kubectl
cat GMP-COMPONENTS.md    # como funcionam os componentes do GMP
cat METRICS-SOURCES.md   # kubelet vs app: quando usar cada um
cat DEBUG.md             # endpoints da API, troubleshooting
```

**AWS:**
```bash
cd aws
cat setup-guide.md  # ~30min, tem IAM e SNS
```

## AWS vs GCP

**GCP:**
- Mais rápido de configurar (só YAML)
- GMP grátis (50GB/mês)
- AlertManager incluso
- SMTP externo (Gmail, SendGrid, etc)

**AWS:**
- Mais config (IAM, SNS)
- SNS pra notificações
- Free tier só no primeiro ano
- Mais integrado com AWS

Usa o que você já conhece.

## Quanto custa?

**AWS:** ~$145/mês (com free tier: ~$10-15/mês no primeiro ano)

**GCP:** ~$50-75/mês (GMP grátis + GKE)

Detalhes completos em `COSTS.md`.

A parte de observabilidade em si (Prometheus) é grátis no GCP e quase grátis no AWS (se ficar no free tier).

## Troubleshooting

**Métrica não aparece:**
- Collector rodando? `kubectl get pods -n gmp-system`
- App expõe `/metrics`? Testa com curl
- PodMonitoring tá certo? `kubectl get podmonitoring -A`
- Labels do pod batem com o selector?

**Alerta não dispara:**
- Rule foi criada? `kubectl get rules -A`
- Query PromQL tá certa? Testa no Metrics Explorer
- Namespace correto? (GCP precisa `gmp-public` pra config)
- Olha status: `kubectl describe rules -n default`

**Email não chega:**
- GCP: App Password do Gmail tá correta?
- AWS: Confirmou o subscription do SNS?
- Checa spam
- Força um alerta derrubando pods: `kubectl scale deployment sample-app --replicas=0`

Tem mais detalhes no `gcp/DEBUG.md` ou nos setup guides.

## O que aprendi montando isso

### Componentes do GMP
- **Collector**: Faz scrape dos pods e envia pro backend gerenciado
- **Rule Evaluator**: Executa queries PromQL e dispara alertas
- **AlertManager**: Agrupa e roteia notificações (SMTP, Slack, etc)
- **Operator**: Gerencia os CRDs (PodMonitoring, Rules, OperatorConfig)

### Métricas
- **Kubelet (automáticas)**: CPU, memória, disco, rede - já existem sem fazer nada
- **kube-state-metrics (automáticas)**: Status dos pods, deployments, nodes
- **Aplicação (custom)**: Request rate, latência, métricas de negócio - precisa instrumentar

### Queries PromQL
- ❌ **NUNCA use** `or absent()` em alertas - impede resolução
- ✅ **Use** `sum(up{...}) < 1` para alertas que precisam resolver
- Ou use métricas do kube-state-metrics: `kube_pod_status_phase{phase!="Running"}`

### AlertManager
- Configuração via Secret em `gmp-public` (YAML com SMTP)
- Agrupa alertas similares (`group_wait`, `group_interval`)
- Envia notificações de FIRING e RESOLVED (se `send_resolved: true`)

### Debug
- Namespace errado é o problema mais comum
- API REST do AlertManager é útil pra ver alertas ativos
- Logs do Collector mostram se scrape está funcionando

Isso aqui é lab, não produção. Se for usar de verdade você vai precisar:
- Mais alertas (CPU, disco, latência, erros da app)
- Múltiplos destinos (email, Slack, PagerDuty)
- Silence durante deploy
- Dashboards
- Monitorar a infra também (nodes, network)

Mas dá pra começar com isso.

## Contribuindo

Se você achar um bug, abrir uma issue ou PR é bem-vindo.

## Licença

MIT - faça o que quiser com isso.
