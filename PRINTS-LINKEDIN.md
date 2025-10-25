# Prints para o Post do LinkedIn

## Status atual do seu ambiente

### ✅ Print 1: Componentes do GMP rodando

```
$ kubectl get pods -n gmp-system

NAME                              READY   STATUS    RESTARTS   AGE
alertmanager-0                    2/2     Running   0          6h
collector-89m7k                   2/2     Running   0          9d
collector-jchdt                   2/2     Running   0          9d
gmp-operator-7f8454f878-6ldhs     1/1     Running   0          9d
rule-evaluator-6c5bc8d6dc-rzswd   2/2     Running   0          6h
```

**Para o post:** "Os 4 componentes do GMP gerenciados pelo Google rodando no meu cluster"

---

### ✅ Print 2: Seus recursos de monitoramento

```
$ kubectl get podmonitoring,rules,operatorconfig -A

NAMESPACE   NAME                                                            AGE
default     podmonitoring.monitoring.googleapis.com/sample-app-monitoring   36m

NAMESPACE   NAME                                         AGE
default     rules.monitoring.googleapis.com/lab-alerts   6h

NAMESPACE    NAME                                              AGE
gmp-public   operatorconfig.monitoring.googleapis.com/config   30d
```

**Para o post:** "Meus CRDs: PodMonitoring (o que coletar), Rules (quando alertar), OperatorConfig (SMTP)"

---

### ✅ Print 3: App rodando normal (ANTES)

```
$ kubectl get pods -l app=sample-app

NAME                          READY   STATUS    RESTARTS   AGE
sample-app-79c6ddcd8c-bwp2k   1/1     Running   0          26m
sample-app-79c6ddcd8c-k5hls   1/1     Running   0          26m
```

**Para o post:** "App rodando normalmente com 2 réplicas"

---

## Agora vamos simular o problema!

### Comando para escalar pra 0

```bash
# Anota o horário
date

# Escala pra 0
kubectl scale deployment sample-app --replicas=0

# Checa status (vai mostrar Terminating)
kubectl get pods -l app=sample-app
```

### 📸 Print 4: Pods terminando (logo após escalar)

**Você vai ver algo assim:**
```
NAME                          READY   STATUS        RESTARTS   AGE
sample-app-79c6ddcd8c-bwp2k   0/1     Terminating   0          27m
sample-app-79c6ddcd8c-k5hls   0/1     Terminating   0          27m
```

**Ou vazio (se já terminaram):**
```
No resources found in default namespace.
```

**Para o post:** "Simulando um crash total: 0 réplicas rodando"

---

## Acompanhando o alerta

### Port-forward pro AlertManager

```bash
# Em um terminal separado, deixa rodando
kubectl port-forward -n gmp-system svc/alertmanager 9093:9093
```

### Aguardar ~2 minutos

Espera o alerta passar de PENDING → FIRING (1min de `for: 1m`).

### 📸 Print 5: Alerta ativo na API

```bash
curl http://localhost:9093/api/v2/alerts | jq
```

**Você vai receber algo assim:**
```json
[
  {
    "labels": {
      "alertname": "PodDown",
      "severity": "critical",
      "namespace": "default"
    },
    "annotations": {
      "description": "Sample app não está respondendo",
      "summary": "0 pods ativos (esperado >= 1)"
    },
    "status": {
      "state": "active",
      "silencedBy": [],
      "inhibitedBy": []
    },
    "receivers": [
      "email"
    ],
    "fingerprint": "abc123def456",
    "startsAt": "2025-10-25T20:05:30.000Z",
    "endsAt": "0001-01-01T00:00:00.000Z",
    "updatedAt": "2025-10-25T20:06:30.000Z"
  }
]
```

**Para o post:** "Alerta ativo retornado pela API do AlertManager (state: active)"

---

### 📸 Print 6: Email chegando

**Aguardar mais ~15-30 segundos e verificar o email.**

Você vai receber um email com:
- **Subject:** `[FIRING:1] PodDown default/sample-app`
- **From:** Seu email configurado no AlertManager
- **Body:** Detalhes do alerta (summary, description, labels)

**IMPORTANTE:** Tire screenshot do email completo mostrando:
- Assunto
- Horário de recebimento
- Corpo do email com os detalhes

**Para o post:** "Email de alerta chegando 75 segundos depois do problema"

---

## Resolver o problema (opcional)

```bash
# Volta os pods
kubectl scale deployment sample-app --replicas=2

# Espera subirem
kubectl get pods -l app=sample-app -w
```

Se você configurou `send_resolved: true` no AlertManager, vai receber outro email:
- **Subject:** `[RESOLVED] PodDown default/sample-app`

---

## Checklist para o post do LinkedIn

- [ ] Print 1: Pods do GMP (gmp-system)
- [ ] Print 2: Seus CRDs (PodMonitoring, Rules, OperatorConfig)
- [ ] Print 3: App com 2 pods rodando (ANTES)
- [ ] Print 4: App escalado pra 0 (pods Terminating ou vazio)
- [ ] Print 5: JSON do alerta ativo (API do AlertManager)
- [ ] Print 6: Screenshot do email de [FIRING]
- [ ] (Opcional) Print 7: Email de [RESOLVED]

---

## Estrutura recomendada do post

1. **Introdução** (texto)
   - 3 descobertas que ninguém te conta

2. **Descoberta 1:** Métricas automáticas
   - Explicação
   - Código de exemplo

3. **Descoberta 2:** NUNCA use `or absent()`
   - Exemplo quebrado vs correto

4. **Descoberta 3:** API do AlertManager
   - Como acessar
   - Casos de uso

5. **Arquitetura** (diagrama ASCII)
   - 4 componentes
   - Fluxo de dados

6. **Timeline** (texto)
   - T+0s até T+75s

7. **DEMO PRÁTICA** ⭐
   - "Vamos ver isso funcionando"
   - **[Print 1]** Pods do GMP
   - **[Print 2]** CRDs criados
   - **[Print 3]** App rodando (2 pods)
   - Comando: `kubectl scale --replicas=0`
   - **[Print 4]** Pods terminando
   - Timeline visual (T+0s, T+30s, T+60s, T+70s, T+75s)
   - **[Print 5]** JSON do alerta ativo
   - **[Print 6]** Email chegando
   - "Total: 75 segundos do problema até a notificação"

8. **Conclusão**
   - Link do repo
   - Call to action

---

## Dicas para prints bonitos

1. **Terminal:** Use tema escuro ou claro consistente
2. **JSON:** Use `| jq` para formatar (colorido)
3. **Email:** Mostre o email completo (não corte)
4. **Fonte:** Tamanho legível (aumenta zoom se necessário)
5. **Destacar:** Pode circular/destacar partes importantes

---

## Comandos extras para copiar/colar

```bash
# Ver logs do AlertManager (quando envia email)
kubectl logs -n gmp-system alertmanager-0 -c alertmanager --tail=50 | grep -i "notify"

# Ver métricas no Metrics Explorer (opcional)
# https://console.cloud.google.com/monitoring/metrics-explorer
# Query: up{job="default/sample-app-monitoring"}

# Ver detalhes da Rule
kubectl get rules lab-alerts -n default -o yaml

# Forçar reload do AlertManager (se precisar)
curl -X POST http://localhost:9093/-/reload
```

---

## Pronto para rodar?

1. Abra 2 terminais:
   - Terminal 1: comandos normais
   - Terminal 2: port-forward do AlertManager

2. Tenha pronto para screenshot:
   - Ferramenta de print (Snipping Tool, macOS Screenshot, etc)
   - Email aberto em outra aba

3. Execute na ordem:
   - Print 1, 2, 3 (estado atual)
   - Escala pra 0 → Print 4
   - Aguarda 2 min → Print 5 (API)
   - Aguarda mais 30s → Print 6 (email)

4. Compila tudo no post!

Bora fazer esse demo? 🚀
