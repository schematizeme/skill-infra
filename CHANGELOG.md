# Changelog — schematize-infra

Formato: [Keep a Changelog](https://keepachangelog.com/pt-BR/). Versionamento semântico.

## [0.2.0] — 2026-08-20

Propagação do piso **"efeito externo NUNCA sai de não-produção"** no recorte de INFRA: esta skill
assume as **camadas 1 (DNS), 3 (rede/segredo) e 4 (provedor)** da defesa em profundidade definida
na `schematize-engineering` (`references/efeitos-externos.md`) e as escreve como **IaC verificável**
— origem: um laço de teste disparou **>5.000 e-mails reais**, queimou reputação de IP/domínio e
derrubaria o **OTP de login** de produção.

### Adicionado
- **SKILL.md** — piso inegociável **10** ("Efeito externo NUNCA sai de não-produção — as camadas
  de infra"): DNS em rota nula provado por `dig`, egress SMTP bloqueado em dev/hml, chave sandbox
  por ambiente, subdomínio de envio separado e alerta de reputação; 3 linhas novas no mapa de
  references; menção ao piso na `description` do frontmatter.
- **references/iac.md** §7 — o *como* concreto: **Terraform/Cloudflare** do domínio de teste
  (**null MX `MX 0 .` RFC 7505 + SPF `v=spf1 -all` + DMARC `p=reject`**) com verificação por `dig`;
  **subdomínio de envio por ambiente** (`mail.<domain>` prd vs `mail-hml.<domain>` hml, DKIM/SPF
  próprios, sandbox/configuration set separados); **egress SMTP 25/465/587/2525 bloqueado** em
  dev/hml com receita para **NetworkPolicy (allowlist)**, **security group + NACL** e **nftables**
  (`reject` e não `drop`, com counter exportado); gate de prova das camadas.
- **references/segredos.md** §7 — **segredo por ambiente**: chave **sandbox** em não-prd, chave de
  prd **inexistente** fora de prd via policy do cofre (exemplo Vault), seed/`.env.example` sem
  credencial de provedor, regra de secret scan por prefixo (`SG.`/`re_`/`sk_live_`) e chave de prd
  em artefato de não-prd tratada como **incidente com rotação imediata**; extensão a PSP/SMS/push/
  webhook.
- **references/observabilidade.md** §9 — **métrica de envio e alerta de reputação**: contadores
  `mail_sent_total`/`mail_bounce_total`/`mail_complaint_total` (destinatário **hasheado**, nunca em
  claro), regras Prometheus prontas para **bounce > 2%**, **complaint > 0,1%** e **envio real fora
  de prd (página imediata)**, runbook de contenção (matar o job → revogar a chave → medir → pausar
  → warm-up) e consumo do webhook de bounce/complaint do provedor.
- **assets/CLAUDE.md** — piso sempre-on **10** com as quatro frentes de infra (DNS, rede, segredo,
  provedor) e os vetos.

### Mudado
- **references/isolamento.md** §4 — o egress deny-by-default agora nomeia explicitamente o
  **bloqueio de SMTP em dev/hml** (o bypass que o guard de aplicação não pega), apontando para
  `iac.md` §7.3.
- **references/entrega.md** §3 — config por ambiente passa a exigir **provider de efeito externo
  resolvido na composição** (sink por default fora de prd, chave sandbox, remetente separado);
  chave/remetente de prd nunca são promovidos junto com o artefato.

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
