# Custos

## Resumo

| Item | AWS | GCP |
|------|-----|-----|
| Cluster | $73 | $73 |
| Nodes (2x medium) | $60 | $48 |
| Storage | $3 | $2 |
| Prometheus | $1 | $0 (free tier) |
| Alerting | $0.02 | $0 (free tier) |
| **TOTAL/mês** | **$137** | **$123** |

## AWS Detalhado

### Infraestrutura
- EKS: $0.10/h = $73/mês
- 2x t3.medium: $0.0416/h × 2 = $60/mês
- EBS 40GB: $3/mês

### Observabilidade
- AMP Ingestion: ~17M samples/mês = $0.50
- AMP Storage: $0.06
- SNS email: $0.02
- SMS (opcional): +$45/mês (1000 msgs)

### Free Tier (12 meses)
- EKS grátis
- Economia: ~$70/mês
- **Total com free tier: $10-15/mês**

## GCP Detalhado

### Infraestrutura
- GKE: $0.10/h = $73/mês
- 2x e2-medium: $0.0335/h × 2 = $48/mês
- Disk 40GB: $2/mês

### Observabilidade
- GMP: **$0** (primeiros 50GB/mês grátis)
- Cloud Monitoring: **$0** (grátis)
- Pub/Sub: **$0** (primeiros 10GB grátis)
- Email: **$0** (grátis)

### Always Free Tier (permanente!)
- Lab fica 100% no free tier de observabilidade

## Otimizações

### AWS
- Spot Instances: -70% nos nodes
- Savings Plans (3 anos): -30% geral

### GCP  
- Preemptible VMs: -80% nos nodes
- Committed Use (3 anos): -57% geral
- **GKE Autopilot:** pode ser mais barato pra workloads pequenos

## Conclusão

GCP é ~$15/mês mais barato, principalmente porque observabilidade é grátis no free tier permanente.
