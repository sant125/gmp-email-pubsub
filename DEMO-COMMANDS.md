# Comandos para Demo do LinkedIn

## 1. Mostrar recursos do GMP rodando

```bash
# Pods do GMP (sistema gerenciado)
kubectl get pods -n gmp-system

# Seus recursos de monitoramento
kubectl get podmonitoring,rules,operatorconfig -A

# Detalhar a configuração do AlertManager
kubectl get secret alertmanager-config -n gmp-public -o yaml
```

**Print sugerido:** Screenshot mostrando os 4 componentes do GMP rodando

---

## 2. Mostrar o app rodando normal

```bash
# Pods da aplicação
kubectl get pods -l app=sample-app

# Métricas do app (opcional)
kubectl port-forward svc/sample-app 8080:8080
curl http://localhost:8080/metrics
```

**Print sugerido:** Pods rodando normalmente (2/2 Ready)

---

## 3. Verificar alertas ANTES de simular

```bash
# Port-forward pro AlertManager
kubectl port-forward -n gmp-system svc/alertmanager 9093:9093

# Ver alertas ativos (deve estar vazio ou com alertas RESOLVED)
curl http://localhost:9093/api/v2/alerts | jq
```

**Print sugerido:** JSON vazio `[]` ou alertas em estado "resolved"

---

## 4. Simular o problema: Escalar pra 0

```bash
# Timestamp antes de derrubar
date

# Escala pra 0 (simula crash total)
kubectl scale deployment sample-app --replicas=0

# Confirma que todos caíram
kubectl get pods -l app=sample-app
```

**Print sugerido:**
```
NAME                          READY   STATUS        RESTARTS   AGE
sample-app-79c6ddcd8c-abc12   0/1     Terminating   0          5m
sample-app-79c6ddcd8c-def34   0/1     Terminating   0          5m
```

---

## 5. Acompanhar o fluxo do alerta

### T+30s - Collector detecta

```bash
# Ver métricas via Metrics Explorer (ou aguardar)
# A métrica up{job="default/sample-app-monitoring"} vira 0
```

### T+60s - Alerta vira FIRING

```bash
# Checa status do alerta
curl http://localhost:9093/api/v2/alerts | jq

# Deve mostrar algo assim:
# {
#   "labels": {
#     "alertname": "PodDown",
#     "severity": "critical"
#   },
#   "status": {
#     "state": "active"
#   },
#   "startsAt": "2025-10-25T20:05:00Z"
# }
```

**Print sugerido:** JSON do alerta em estado "active"

### T+70s - Email enviado

```bash
# Checa logs do AlertManager
kubectl logs -n gmp-system alertmanager-0 -c alertmanager --tail=20

# Deve mostrar:
# level=info msg="Notify success" receiver=email
```

**Print sugerido:** Logs mostrando "Notify success"

### T+75s - Email chega

**Print sugerido:** Screenshot do email recebido mostrando:
- Subject: `[FIRING:1] PodDown default/sample-app`
- Body com detalhes do alerta
- Timestamp

---

## 6. Resolver o problema

```bash
# Volta as réplicas
kubectl scale deployment sample-app --replicas=2

# Aguarda pods subirem
kubectl get pods -l app=sample-app -w

# Aguarda ~2 minutos e checa email de RESOLVED (se send_resolved: true)
```

**Print sugerido:** Email de `[RESOLVED]` chegando

---

## 7. Timeline completa para o post

```
T+0s    └─ kubectl scale --replicas=0
T+0s       Pods começam a terminar

T+30s   └─ Collector faz scrape
           Detecta up=0
           Envia pro backend GMP

T+60s   └─ Rule Evaluator
           Query: sum(up{...}) < 1 → true há 1min
           Alerta muda: PENDING → FIRING

T+70s   └─ AlertManager
           Recebe alerta FIRING
           Aguarda group_wait (10s)
           Agrupa alertas similares
           Envia via SMTP

T+75s   └─ 📧 Email chega
           Subject: [FIRING:1] PodDown
```

---

## Estrutura sugerida para o LinkedIn

**Parte 1: Teoria** (o que você já tem)
- Explicação dos componentes
- API do AlertManager
- Arquitetura

**Parte 2: Na prática** (adicionar)
- "Vamos simular um problema real"
- Print dos componentes rodando
- Comando: `kubectl scale --replicas=0`
- Print do alerta na API do AlertManager
- Timeline: T+0s até T+75s
- Print do email chegando

**Parte 3: Call to action**
- Link pro repo
- Pergunta pro público

---

## Comandos extras para prints bonitos

```bash
# Mostrar todos os recursos do GMP de uma vez
kubectl get all -n gmp-system

# Mostrar suas configs de monitoring
kubectl get podmonitoring,rules -n default -o wide

# Ver detalhes da Rule
kubectl get rules lab-alerts -n default -o yaml

# Ver queries PromQL aplicadas
kubectl get rules lab-alerts -n default -o jsonpath='{.spec.groups[0].rules[*].alert}{"\n"}{.spec.groups[0].rules[*].expr}'
```

---

## Dicas para os prints

1. **Pods do GMP:** Mostrar que são gerenciados (você não os criou)
2. **Seus CRDs:** Mostrar PodMonitoring, Rules, OperatorConfig
3. **App rodando:** 2/2 pods healthy
4. **Escalar pra 0:** Pods em Terminating
5. **API do AlertManager:** JSON do alerta FIRING
6. **Email:** Screenshot completo (subject + body)
7. **Resolved:** Email de resolução (opcional mas impacta)

---

## Ordem recomendada no post

1. Arquitetura (diagrama texto)
2. **NOVO:** "Vamos ver isso funcionando na prática"
3. Print: Recursos do GMP rodando
4. Print: App com 2 pods rodando
5. Comando: `kubectl scale --replicas=0`
6. Timeline visual (T+0s até T+75s)
7. Print: Alerta na API do AlertManager
8. Print: Email chegando
9. Conclusão + link do repo

Isso vai deixar o post muito mais concreto e demonstrável!
