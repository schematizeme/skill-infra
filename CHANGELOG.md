# Changelog — schematize-infra

Formato: [Keep a Changelog](https://keepachangelog.com/pt-BR/). Versionamento semântico.

## [0.1.0] — 2026-08-15

Primeira versão da skill de **infra/DevOps/SRE** da casa — a disciplina de operar sistema em
produção com **blast radius mínimo, tudo declarado e tudo auditado**, deny-by-default, extraindo o
*como* de infra do que a `schematize-engineering` normatiza (ops §2/§3, deploy §21, observabilidade
§16, cadeia de suprimentos) pra rodar solta e agnóstica de linguagem.

### Adicionado
- **SKILL.md** com 9 pisos inegociáveis (isolamento por app, deny-by-default/least-privilege, infra
  é código/nada de mão no servidor, deploy gated onde a casa exige, segredo nunca no repo,
  observabilidade integrada, backup com restore testado, cadeia de suprimentos de primeira classe,
  tudo auditado) + mapa de references + relação com engineering/pentest.
- **references/**:
  - `isolamento.md` — isolamento por aplicação: user Linux dedicado + systemd unit hardened +
    container não-root/read-only por serviço, blast radius mínimo, isolamento de rede
    deny-by-default, layout `/<app>/` + seed, provisionamento pelo ops (nunca à mão).
  - `iac.md` — infra é código: declarativa/idempotente/revisável, plan-before-apply, state seguro,
    drift detection, artefato imutável com proveniência git, o `<projeto>_ops` como interface única.
  - `entrega.md` — CI/CD gated: fluxo fixo de promoção, deploy manual/gated onde a casa exige
    (front/Vercel, prd), imagem de container (não-root/read-only/distroless/multi-stage/pinada por
    digest), estratégia + healthcheck gating + rollback, feature flags, pipeline como TCB.
  - `segredos.md` — gestão de segredos: nunca no repo/layer/env commitado, cofre + rotação
    documentada + secret scan que trava, referência pelo seed, procedimento de vazamento (rotação
    imediata).
  - `observabilidade.md` — LGTM+ integrado: logs (sem PII) + métricas RED/USE + traços por
    `trace_id`, SLO com error budget, alertas versionados e acionáveis, healthchecks, trilha de
    auditoria imutável, metas de performance + FinOps.
  - `resiliencia.md` — backup & DR (RPO/RTO, restore testado), rede/TLS/mTLS/hardening
    (deny-by-default, least-privilege), cadeia de suprimentos (SBOM/pinning/assinatura/SLSA),
    degradação graciosa.
- **assets/commands/**: `/infra-help`, `/infra-deploy` (deploy gated), `/infra-iac` (IaC + ops),
  `/infra-secrets` (gestão de segredos), `/infra-load`, `/infra-claude`, `/infra-cc`,
  `/infra-handoff`.
- **assets/CLAUDE.md** — regra sempre-on: blast radius mínimo, tudo declarado, tudo auditado,
  deny-by-default.
