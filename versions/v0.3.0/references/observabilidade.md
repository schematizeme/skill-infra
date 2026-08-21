# Observabilidade — logs, métricas, traços, SLO e alertas (integrada, não posterior)

> Sistema que você não enxerga, você não opera — você reza. Todo serviço nasce com observabilidade
> **integrada de ponta a ponta**: logs estruturados, métricas RED/USE, traços correlacionados,
> **SLO com error budget** e alertas **versionados como código**. Nunca "a gente instrumenta
> depois". Este reference é o *como* de infra do §16 da `schematize-engineering` (stack LGTM+).

Um serviço só é **"pronto"** (§35) se expõe `/metrics`, emite logs estruturados e traces, e sobe
com **dashboard + alertas + chart** versionados junto do código. Observabilidade que não subiu com
o serviço não existe quando você precisa dela — às 3h da manhã, no incidente.

> **Gate:** `scripts/check-observabilidade.sh` (comando `/infra-check`) cobra liveness e readiness
> **distintos** (readiness que é apelido do liveness é healthcheck vacuamente verdadeiro), log sem
> PII/segredo e SLO escrito.

## 1. Stack — LGTM+ e OpenTelemetry

- **Instrumentação:** OpenTelemetry — traces, métricas e logs **correlacionados por `trace_id`**
  (exemplars ligando métrica ↔ trace). Padrão único, sem vendor lock por instrumentação.
- **Coleta:** Grafana Alloy (coletor/agente OTel) — ou o collector padrão do OTel.
- **Backends:** Loki (logs), Tempo (traces), Prometheus (scrape) + **Mimir** (métricas long-term,
  HA, multi-tenant); **SHOULD** Pyroscope (profiling contínuo).
- **Visualização e alerta:** Grafana — dashboards e regras de alerta **versionados como código**,
  entregues junto do serviço.
- **Deploy:** **Helm chart versionado por serviço** — o repo entrega seu chart + dashboards +
  alertas com o código. Avalie extras por fit: Grafana OnCall (on-call), k6 (carga), Beyla/eBPF
  (auto-instrumentação).

## 2. Logs — estruturados, correlacionados, sem PII

- **JSON estruturado**, com `trace_id`, `correlation_id`, `tenant_id` (se aplicável) em **toda
  request**. Níveis DEBUG/INFO/WARN/ERROR.
- **Proibido logar:** senha, token, JWT, PII (CPF, email, telefone), dado financeiro, payload de
  pagamento. **Mascaramento obrigatório**. Segredo em log = incidente de vazamento (`segredos.md`).
- **VETADO** logar request/response inteiros, headers ou body cru "pra debugar". Logue **campos
  específicos, mascarados**.
- **Retenção** definida por tipo de log; log não é banco de dados (não guarde tudo pra sempre).

## 3. Métricas — RED e USE

- **RED por endpoint/handler:** **R**ate, **E**rrors, **D**uration (histograma p50/p95/p99).
- **USE para infra/recurso:** **U**tilization, **S**aturation, **E**rrors (CPU, memória, disco,
  fila, conexão).
- **Métricas de negócio** (SHOULD): pedidos/min, conversão, fila de processamento — dashboards de
  negócio **separados** dos técnicos, KPIs instrumentados desde o dia 1.

## 4. Tracing — a jornada da request

- Toda **chamada externa, fila, banco e fluxo crítico** instrumentados.
- Propagação **W3C Trace Context** (`traceparent`) — o `trace_id` atravessa serviço, fila e job,
  ligando a jornada inteira. É o que transforma "o pagamento tá lento" em "o span do gateway
  externo leva 2s".

## 5. SLO, error budget e alertas úteis

- **SLI/SLO por serviço** em `/docs/slo.md` (ex.: 99,9% das requests < 300ms). O SLO é a
  **definição do que é "funcionando"** — sem ele, alerta é chute.
- **Error budget:** consumido o orçamento de erro → **freeze de features** até recuperar. Prioriza
  confiabilidade sobre feature quando o serviço está no vermelho.
- **Alertas versionados como código** e **acionáveis**: alerta que dispara sobre **sintoma que dói
  no usuário** (SLO burn rate, `/ready` falhando, fila crescendo sem parar), não sobre "CPU a 80%"
  que ninguém sabe o que fazer. **Alerta que ninguém age vira ruído** — e ruído esconde o alerta
  real. Cada alerta aponta um **runbook**.
- **On-call** com runbook por alerta (RUNBOOK.md, §29 templates da engenharia).

## 6. Healthchecks — o contrato com o orquestrador

Endpoints obrigatórios (o deploy/orquestração dependem deles — `entrega.md` §5):

- **`/health`** — liveness (o processo está vivo; se falha, reinicia).
- **`/ready`** — readiness (dependências OK, pronto pra tráfego; se falha, **sai do balanceador**
  sem ser morto).
- **`/metrics`** — Prometheus.

Liveness e readiness **não são a mesma coisa**: matar o pod porque uma dependência piscou
(liveness confundido com readiness) causa cascata. Readiness tira do tráfego; liveness reinicia.

## 7. Trilha de auditoria — imutável

Operação sensível (mudança de permissão, transação financeira, alteração de config, ação
administrativa, break-glass de infra) vai pra **trilha de auditoria imutável** (append-only, tabela
**separada** do domínio):

- Campos mínimos: `actor_id`, `tenant_id`, `action`, `resource`, `timestamp`, `ip`, `user_agent`,
  `result`. Retenção conforme regulação.
- É diferente de log de aplicação: é **evidência**, não debug. Casa com o piso "tudo auditado" —
  toda operação de infra pelo ops grava rastro no archive (§28).

## 8. Metas de performance e FinOps

- **Metas padrão** (sobrescrevíveis no `/docs/slo.md`): API p95 < 300ms, p99 < 1s, startup < 10s,
  imagem Docker < 200MB (Go/Rust) / < 400MB (Node).
- **FinOps** (SHOULD): custo por serviço/tenant, budget com alerta de threshold, revisão de
  overprovisioning, tags de billing consistentes na IaC, custo por request em serviço de alto
  volume. Observabilidade também mede **dinheiro**, não só latência.

## 9. Efeito externo — métrica de envio e alerta de reputação

Os **>5.000 e-mails** do incidente que originou o piso (`schematize-engineering` →
`references/efeitos-externos.md`) só existiram porque **nada estava contando**. Todo projeto que
envia qualquer coisa (e-mail, SMS/voz, push, webhook de terceiro, cobrança) instrumenta o envio e
**alerta na reputação**, que é o ativo que não tem undo:

- **Métrica obrigatória por envio** (real ou sinkado): `mail_sent_total{env,provider,template,
  status}`, `mail_bounce_total`, `mail_complaint_total` — com `run_id` e **hash** do destinatário.
  **Nunca o endereço em claro** no log/label (é PII — §2; e vira cardinalidade infinita).
- **Limiares da indústria** (SES/Resend/Postmark suspendem a conta acima deles): **bounce > 2%**
  e **complaint > 0,1%**. São **alerta crítico com runbook**, não painel pra olhar depois.
- **Envio real fora de `prd` é página imediata** — não existe volume "aceitável" de e-mail real em
  hml/dev. Também alerte o **contador da regra de firewall** que barra SMTP fora de prd
  (`iac.md` §7.3): tentativa bloqueada é sinal de bypass tentado.

```yaml
# ops/iac/alerts/efeitos-externos.yaml — versionado com o servico, como todo alerta
groups:
  - name: efeitos-externos
    rules:
      - alert: EnvioRealForaDeProducao
        expr: sum by (env) (increase(mail_sent_total{provider!="sink", env!="prd"}[5m])) > 0
        for: 0m
        labels: { severity: critical }
        annotations:
          summary: "e-mail REAL saindo de {{ $labels.env }} - parar o job e revogar a chave"
          runbook_url: "<projeto>_archive/runbooks/efeito-externo.md"
      - alert: BounceRateAcimaDoLimiar
        expr: sum(rate(mail_bounce_total[1h])) / sum(rate(mail_sent_total[1h])) > 0.02
        for: 15m
        labels: { severity: critical }
      - alert: ComplaintRateAcimaDoLimiar
        expr: sum(rate(mail_complaint_total[1h])) / sum(rate(mail_sent_total[1h])) > 0.001
        for: 15m
        labels: { severity: critical }
```

- **Runbook do alerta** (obrigatório, §5): **1)** matar o job/laço e **revogar a chave** — nessa
  ordem; **2)** medir enviados/bounce/complaint/spam trap no painel do provedor; **3)** acima do
  limiar, **pausar todo envio** daquele subdomínio e abrir ticket com o provedor **antes** que ele
  suspenda a conta; **4)** voltar por **warm-up** gradual, nunca no volume normal; **5)** o
  post-mortem fecha **com a camada implantada**, não com "combinamos de tomar cuidado".
- **Webhook de bounce/complaint do provedor** assinado e consumido (SNS/webhook) — é o que
  alimenta os contadores acima; sem ele, você descobre a reputação queimada pelo suporte.

> Regra de bolso: **se caiu e você foi olhar o dashboard, ele já tinha que estar lá.** Logs
> estruturados sem PII, RED/USE, traço correlacionado, SLO que define "funcionando", alerta que
> dói no usuário e aponta runbook, `/health` `/ready` `/metrics` sempre. Subiu com o serviço ou não
> subiu.
