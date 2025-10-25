# Lab de Observabilidade em Kubernetes

Projeto pra aprender observabilidade em Kubernetes usando Prometheus gerenciado (AWS ou GCP). Quando algo dá errado no cluster, você recebe email/SMS avisando.

## O que tem aqui

Um setup funcional de:
- Coleta automática de métricas dos pods
- Regras de alerta customizadas (pod down, memória alta, etc)
- Notificações via email quando algo quebra
- Tudo usando serviços gerenciados (zero manutenção de Prometheus)

## Por que usar serviços gerenciados?

Você não precisa:
- Gerenciar storage do Prometheus
- Se preocupar com escalabilidade
- Configurar backup e alta disponibilidade
- Passar noite acordado quando o Prometheus cai

É mais caro que self-hosted? Sim. Mas você economiza tempo e dor de cabeça.

## Estrutura do projeto

```
.
├── README.md                    # você está aqui
├── LEARNING.md                  # explicação didática dos conceitos
├── ARCHITECTURE.md              # detalhes técnicos da arquitetura
├── COSTS.md                     # quanto vai custar
├── aws/                         # setup AWS (AMP + SNS)
│   ├── setup-guide.md
│   ├── iam-policies/
│   ├── prometheus-config/
│   └── k8s-manifests/
└── gcp/                         # setup GCP (GMP + SMTP direto)
    ├── setup-guide.md
    ├── iam-policies/
    ├── prometheus-config/
    └── k8s-manifests/
```

## Quick start

### Primeira vez com observabilidade?

Lê o `LEARNING.md` antes. Ele explica os conceitos de Prometheus, métricas, alertas, etc.

### Já sabe o que tá fazendo?

**GCP (mais simples):**
```bash
cd gcp
cat setup-guide.md
# 15 minutos de setup, tudo via kubectl apply
```

**AWS (mais completo):**
```bash
cd aws
cat setup-guide.md
# 30 minutos de setup, envolve IAM e SNS
```

## Diferenças entre AWS e GCP

**GCP:**
- Setup mais rápido (só YAML)
- Observabilidade grátis (até 50GB/mês)
- AlertManager já integrado
- Precisa configurar SMTP externo

**AWS:**
- Setup mais trabalhoso (IAM, SNS, etc)
- Notificações nativas via SNS
- Free tier limitado (primeiro ano)
- Mais recursos de integração

Ambos funcionam bem. Escolhe o que você já usa ou quer aprender.

## Quanto custa?

**AWS:** ~$145/mês (com free tier: ~$10-15/mês no primeiro ano)

**GCP:** ~$50-75/mês (GMP grátis + GKE)

Detalhes completos em `COSTS.md`.

A parte de observabilidade em si (Prometheus) é grátis no GCP e quase grátis no AWS (se ficar no free tier).

## Troubleshooting

**Métricas não aparecem:**
- Verifica se o collector/agent tá rodando
- Checa se o app expõe `/metrics`
- Confirma que o PodMonitoring/ServiceMonitor tá certo
- GCP: `kubectl get podmonitoring -A`
- AWS: checa logs do prometheus-agent

**Alertas não disparam:**
- Verifica se as Rules foram criadas
- Testa a query PromQL manualmente
- Confirma o namespace (GCP precisa ser `gmp-public`)
- GCP: `kubectl get rules -A`
- AWS: checa o AlertManager

**Notificações não chegam:**
- GCP: Confere Gmail App Password e SMTP
- AWS: Verifica subscription do SNS (confirma o email)
- Olha a pasta de spam
- Testa manualmente (força um alerta pra ver se chega)

Mais detalhes de troubleshooting nos setup guides específicos.

## O que você vai aprender

- Como coletar métricas de pods no Kubernetes
- Escrever queries PromQL pra buscar métricas
- Criar regras de alerta customizadas
- Configurar roteamento de alertas (email, slack, pagerduty)
- Debugar quando métricas não aparecem
- Usar Prometheus gerenciado vs self-hosted

Esse é um projeto de lab. Se for usar em produção, você vai precisar:
- Alertas mais sofisticados (ver awesome-prometheus-alerts)
- Múltiplos receivers (email, slack, pagerduty)
- Silenciamento durante deploys
- Dashboards customizados
- Monitoramento de infra (nodes, disk, network)

Mas o básico tá aqui e funciona.

## Contribuindo

Se você achar um bug, abrir uma issue ou PR é bem-vindo.

## Licença

MIT - faça o que quiser com isso.
