# Observabilidade

Stack de observabilidade (Prometheus + Alertmanager) para um cluster Kubernetes homelab, com alertas de infraestrutura (CPU, memória, disco) roteados para o Telegram.

Repositório: [lotechdevops/observabilidade](https://github.com/lotechdevops/observabilidade)

## Arquitetura

```
node_exporter (DaemonSet, já incluso no kube-prometheus-stack)
    ↓
Prometheus (avalia PrometheusRules a cada scrape)
    ↓
PrometheusRule (dispara alerta quando threshold é ultrapassado)
    ↓
AlertmanagerConfig (roteia, formata e inibe)
    ↓
Telegram (canal de notificação)
```

- **Namespace:** `observabilidade`
- **Stack base:** `prometheus-community/kube-prometheus-stack` (Helm)
- **Selector exigido pelo Prometheus:** todo `PrometheusRule` precisa da label `release: kube-prometheus-stack` para ser carregado (`ruleSelector`)
- **Selector exigido pelo Alertmanager:** todo alerta precisa da label `channel: telegram-homelab` para ser roteado ao receiver do Telegram (`AlertmanagerConfig.route.matchers`)

## Estrutura do repositório

```
observabilidade/
├── prometheusRules/
│   ├── cpu_nodes_alerts.yaml       # HighCPUUsage (warning), CriticalCPUUsage (critical)
│   ├── memory_nodes_alerts.yaml    # HighMemoryUsage (warning), CriticalMemoryUsage (critical)
│   └── disk_nodes_alerts.yaml      # HighDiskUsage (warning), CriticalDiskUsage (critical)
└── alertmanager/
    ├── secret.yaml                       # token do bot do Telegram (não versionar com valor real)
    └── telegram-alertmanager-config.yaml # roteamento, template de mensagem e inhibitRules
```

## Alertas

| Alerta | Severidade | Threshold | Métrica base |
|---|---|---|---|
| `HighCPUUsage` | warning | > 70% por 5m | `node_cpu_seconds_total` |
| `CriticalCPUUsage` | critical | > 90% por 2m | `node_cpu_seconds_total` |
| `HighMemoryUsage` | warning | > 70% por 5m | `node_memory_MemAvailable_bytes` / `node_memory_MemTotal_bytes` |
| `CriticalMemoryUsage` | critical | > 90% por 2m | `node_memory_MemAvailable_bytes` / `node_memory_MemTotal_bytes` |
| `HighDiskUsage` | warning | > 70% por 5m | `node_filesystem_avail_bytes` / `node_filesystem_size_bytes` (mountpoint `/`) |
| `CriticalDiskUsage` | critical | > 90% por 2m | `node_filesystem_avail_bytes` / `node_filesystem_size_bytes` (mountpoint `/`) |

Todos os alertas críticos usam `for` menor que o equivalente warning (2m vs 5m) — garante que o alerta mais severo confirme primeiro, permitindo que a regra de inibição já esteja ativa quando o warning completar seu próprio tempo de espera.

## Inibição (inhibitRules)

O `AlertmanagerConfig` define uma regra de inibição baseada em `severity`, não em `alertname` — por isso ela se aplica automaticamente aos três domínios (CPU, memória, disco), sem precisar de uma regra por métrica:

```yaml
inhibitRules:
  - sourceMatch:
      - name: 'severity'
        value: 'critical'
        matchType: '='
    targetMatch:
      - name: 'severity'
        value: 'warning'
        matchType: '='
    equal: ['instance']
```

Se um `critical` está ativo para uma `instance`, o `warning` equivalente da mesma instância não gera notificação — mas continua visível como `Firing` em `http://localhost:9090/alerts`, e aparece marcado como **Inhibited** na UI do Alertmanager.

**Limitação conhecida:** ao resolver, os thresholds (70% e 90%) não caem exatamente ao mesmo tempo — existe uma janela intermediária (entre 70% e 90%, descendo) onde o critical já resolveu mas o warning ainda está ativo. Nessa janela, o warning notifica normalmente, mesmo tendo sido inibido durante o pico. É um comportamento esperado do `inhibit_rules` (reage ao estado atual, não ao histórico), não um erro de configuração.

## Pré-requisitos para reproduzir

- Cluster com `kube-prometheus-stack` instalado no namespace `observabilidade`
- Bot do Telegram criado via `@BotFather` (token) e `chat_id` do destino (canal ou conversa)
- `secret.yaml` preenchido com o token real, **não commitado** — usar `.gitignore` ou versionar como `.example.yaml` com placeholder

## Aplicando

```bash
kubectl apply -f alertmanager/secret.yaml
kubectl apply -f alertmanager/telegram-alertmanager-config.yaml
kubectl apply -f prometheusRules/cpu_nodes_alerts.yaml
kubectl apply -f prometheusRules/memory_nodes_alerts.yaml
kubectl apply -f prometheusRules/disk_nodes_alerts.yaml
```

## Validação

```bash
# Confirma que as regras carregaram sem erro
kubectl get prometheusrules -n observabilidade

# Confirma que o AlertmanagerConfig foi aceito
kubectl exec -n observabilidade alertmanager-kube-prometheus-stack-alertmanager-0 \
  -- amtool config show --alertmanager.url=http://localhost:9093

# UI do Prometheus — estado dos alertas
kubectl port-forward -n observabilidade svc/kube-prometheus-stack-prometheus 9090:9090
# http://localhost:9090/alerts

# UI do Alertmanager — roteamento e inibições ativas
kubectl port-forward -n observabilidade svc/kube-prometheus-stack-alertmanager 9093:9093
# http://localhost:9093 (marcar filtro "Inhibited" para confirmar supressão)
```