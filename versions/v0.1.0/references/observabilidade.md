# Observabilidade — logs, métricas, traços, SLO e alertas (integrada, não posterior)

> Sistema que você não enxerga, você não opera — você reza. Todo serviço nasce com observabilidade
> **integrada de ponta a ponta**: logs estruturados, métricas RED/USE, traços correlacionados,
> **SLO com error budget** e alertas **versionados como código**. Nunca "a gente instrumenta
> depois". Este reference é o *como* de infra do §16 da `schematize-engineering` (stack LGTM+).

Um serviço só é **"pronto"** (§35) se expõe `/metrics`, emite logs estruturados e traces, e sobe
com **dashboard + alertas + chart** versionados junto do código. Observabilidade que não subiu com
o serviço não existe quando você precisa dela — às 3h da manhã, no incidente.

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

> Regra de bolso: **se caiu e você foi olhar o dashboard, ele já tinha que estar lá.** Logs
> estruturados sem PII, RED/USE, traço correlacionado, SLO que define "funcionando", alerta que
> dói no usuário e aponta runbook, `/health` `/ready` `/metrics` sempre. Subiu com o serviço ou não
> subiu.
