# Lab de Observabilidade em Kubernetes

Projeto de observabilidade pra clusters Kubernetes usando Prometheus gerenciado (AWS ou GCP) com alertas via email/SMS.

## O que é isso

Um lab prático que mostra como:
- Coletar métricas de pods no Kubernetes
- Configurar alertas personalizados
- Receber notificações quando algo dá errado
- Tudo usando serviços gerenciados (sem precisar subir servidor de Prometheus)

## Por que usar serviços gerenciados

- Não precisa gerenciar infra do Prometheus
- Escalabilidade automática
- Backup e HA já resolvidos
- Foco na aplicação, não no monitoring

## Estrutura do Projeto

```
.
├── README.md                    # este arquivo
├── ARCHITECTURE.md              # detalhes da arquitetura
├── COSTS.md                     # breakdown de custos
├── aws/                         # setup AWS
│   ├── setup-guide.md
│   ├── iam-policies/
│   ├── prometheus-config/
│   └── k8s-manifests/
└── gcp/                         # setup GCP
    ├── setup-guide.md
    ├── iam-policies/
    ├── prometheus-config/
    └── k8s-manifests/
```

## Quick Start

### AWS
```bash
cd aws
cat setup-guide.md
# siga os passos
```

### GCP
```bash
cd gcp
cat setup-guide.md
# siga os passos
```

## Custos esperados

**AWS:** ~$145/mês (com free tier: ~$10-15/mês no primeiro ano)
**GCP:** ~$122/mês (observabilidade fica no free tier permanente)

Detalhes completos em `COSTS.md`

## Troubleshooting comum

**Métricas não aparecem:**
- Verifica se o agent tá rodando
- Checa logs do Prometheus
- Confirma que o app tá expondo /metrics

**Alertas não disparam:**
- Verifica se as rules foram criadas
- Testa a query PromQL manualmente
- Checa se o AlertManager tá ativo

**Notificações não chegam:**
- Confirma subscription do email/SMS
- Verifica se confirmou o email
- Checa pasta de spam
- Testa mandando uma notificação manual
