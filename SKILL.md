---
name: schematize-infra
metadata:
  version: 0.1.0
description: Infraestrutura/DevOps/SRE da casa — a disciplina de operar sistema em produção com blast radius mínimo, tudo declarado, tudo auditado e deny-by-default. Cobre isolamento por aplicação (um user Linux + systemd hardened + container por serviço, nunca root, nunca dois serviços no mesmo user — casando com o `<projeto>_ops` e o ops.md da engenharia), IaC (declarativo, idempotente, revisável — nada de mão no servidor; artefato imutável com proveniência git; drift detection), CI/CD com **deploy gated/manual** onde a casa exige (o fluxo fixo `dev local → teste local verde → GitHub → hml → prd`, nada pula etapa; o front deploya **manual/gated** na Vercel), containers & orquestração (não-root, read-only, distroless, multi-stage; rolling/blue-green/canary com healthcheck gating e rollback automatizado), gestão de segredos (nunca no repo/layer/env commitado — cofre + rotação documentada + secret scan que trava o merge), observabilidade (logs/métricas/traços correlacionados por `trace_id`, RED/USE, SLO com error budget, alertas versionados como código — serviço só é "pronto" com `/metrics` + dashboard + alerta + chart), backup & DR (RPO/RTO definidos, **restore testado** — backup sem restore provado não é backup), redes/TLS/hardening (TLS 1.2+, mTLS interno, deny-by-default no firewall/security group, least-privilege no IAM de nuvem) e cadeia de suprimentos (SBOM versionado junto ao artefato, pinning por digest `@sha256`, assinatura verificada na admissão — SLSA). Pisos inegociáveis (isolamento por app, deny-by-default, least-privilege, segredo fora do repo, deploy gated, restore testado, tudo auditado) que vetam o atalho inseguro. Use SEMPRE que for projetar, revisar, provisionar ou operar infra, deploy, CI/CD, container, orquestração, rede, TLS, segredo, backup/DR, observabilidade ou hardening — mesmo sem citar "infra". Pareia com a schematize-engineering (a BASE: ops §2/§21, observabilidade §16, cadeia de suprimentos, archive §28) e delega segurança ofensiva à schematize-pentest.
---

# Infraestrutura/DevOps/SRE da casa (schematize-infra)

Disciplina normativa, **agnóstica de linguagem**, que responde a uma pergunta só: **este sistema
roda em produção com blast radius mínimo, tudo declarado e tudo auditado?** A casa não opera
servidor na mão. Cada serviço roda **isolado** (user Linux + systemd hardened + container), toda
mudança nasce de **código versionado** (IaC + artefato imutável), o deploy é **gated** onde a
casa exige, o segredo **nunca** mora no repo, e nada acontece fora do **`<projeto>_ops`** — o que
não está no ops/IaC **não aconteceu** e não é reproduzível.

É a contraparte de **operação** da engenharia: a `schematize-engineering` define *que* o ops é a
interface única (§2), *que* o deploy segue o fluxo fixo (§21) e *que* observabilidade é integrada
(§16); esta skill é o **como** dessas frentes de infra, com pisos que vetam o atalho ("editei
direto no prd", "botei o segredo no `.env` do repo", "o backup roda, nunca testei o restore").

**Versão:** skill `schematize-infra` v0.1.0. Changelog em `CHANGELOG.md`.

## Comandos (Claude Code)

Digite `/infra-help` pra ver todos. Em resumo:

| Comando | O que faz |
|---|---|
| `/infra-help` | lista todos os comandos do schematize-infra |
| `/infra-deploy` | audita/planeja o deploy **gated**: fluxo fixo (local→teste→git→hml→prd), artefato imutável com proveniência git, estratégia (rolling/blue-green/canary) + healthcheck gating + rollback; respeita o **manual/gated** onde a casa exige (ex.: front na Vercel) |
| `/infra-iac` | audita/scaffolda a **IaC** (declarativa, idempotente, revisável), verifica drift, plan-before-apply, state seguro e o layout `/<app>/` + seed do `<projeto>_ops` |
| `/infra-secrets` | audita a **gestão de segredos**: nada no repo/layer/env commitado, cofre + rotação documentada, secret scan que trava, referência pelo seed — sinaliza **rotação** se achar segredo vazado |
| `/infra-load` | carrega à força TODO o corpo normativo (isolamento, IaC, entrega, segredos, observabilidade, resiliência) e passa a aplicá-lo |
| `/infra-claude` | cria ou mescla o `CLAUDE.md` sempre-on de infra na raiz do repo |
| `/infra-cc` | context compact: gera handoff no archive e roda `/compact` |
| `/infra-handoff` | gera o handoff (context.md + checklist.md) sem compactar |

Os comandos ficam em `assets/commands/` e são instalados em `.claude/commands/`.

## Como usar esta skill

1. **Isole antes de tudo** (`references/isolamento.md`): um user Linux + systemd hardened +
   container por serviço, nunca `root`, nunca dois serviços no mesmo user. Blast radius contido é
   pré-condição, não enfeite — casa com o isolamento do `<projeto>_ops` (ops.md §3 da engenharia).
2. **Declare, não digite** (`references/iac.md`): toda infra é código — declarativo, idempotente,
   revisável, com `plan` antes do `apply` e drift detection. Nada de mão no servidor; o servidor é
   **imutável por edição manual** e recebe só artefato promovido do git.
3. **Deploye pelo trilho** (`references/entrega.md`): o fluxo fixo `dev local → teste local verde
   → GitHub → hml → prd`, nada pula etapa; **deploy gated/manual** onde a casa exige (o front
   deploya manual na Vercel). Container não-root/read-only/distroless; healthcheck gating +
   rollback automatizado.
4. **Guarde o segredo fora** (`references/segredos.md`): nunca no repo, nunca em layer, nunca em
   `.env` commitado — cofre + rotação + secret scan que trava o merge.
5. **Enxergue o sistema** (`references/observabilidade.md`): logs/métricas/traços correlacionados,
   SLO com error budget, alertas versionados. Serviço sem `/metrics` + dashboard + alerta **não é
   "pronto"**.
6. **Sobreviva ao desastre** (`references/resiliencia.md`): RPO/RTO definidos, **restore testado**;
   rede deny-by-default, TLS 1.2+/mTLS, hardening; supply-chain (SBOM + pinning + assinatura).
7. **Não trabalhe de memória** — os detalhes estão nos references. Aplique os pisos abaixo
   independentemente do reference carregado.

Mapa de references — leia o que casa com a tarefa:

| Tarefa | Reference |
|---|---|
| Isolamento por aplicação: user Linux dedicado + systemd hardened + container por serviço, blast radius, layout `/<app>/`, provisionamento pelo ops | `references/isolamento.md` |
| IaC declarativa/idempotente/revisável, plan-before-apply, state seguro, drift detection, artefato imutável, o ops como interface única | `references/iac.md` |
| CI/CD com deploy **gated/manual**, fluxo fixo de promoção, imagem de container (não-root/read-only/distroless/multi-stage), estratégia de deploy + healthcheck gating + rollback | `references/entrega.md` |
| Gestão de segredos: nunca no repo/layer, cofre, rotação, secret scan, referência pelo seed, rotação em caso de vazamento | `references/segredos.md` |
| Observabilidade: logs/métricas/traços (RED/USE), SLO/error budget, alertas versionados, healthchecks, trilha de auditoria imutável | `references/observabilidade.md` |
| Backup & DR (RPO/RTO, restore testado), redes/TLS/mTLS/hardening (deny-by-default, least-privilege), cadeia de suprimentos (SBOM/pinning/assinatura/SLSA) | `references/resiliencia.md` |

## Pisos inegociáveis (vetam o atalho)

Independente do reference, estes limites nunca são cruzados:

1. **Isolamento por aplicação.** Cada serviço roda **confinado**: um **user Linux dedicado** + um
   **systemd unit hardened** (`NoNewPrivileges`, `ProtectSystem=strict`, capabilities mínimas) +
   **container não-root/read-only**. **Nunca** `root`; **nunca** dois serviços no mesmo user.
   Blast radius contido é design, não sorte — um serviço comprometido não vaza para os vizinhos.
2. **Deny-by-default e least-privilege em tudo.** Rede (firewall/security group/network policy),
   IAM de nuvem, capabilities de container, permissão de filesystem, escopo de token de CI —
   tudo começa **fechado** e abre só o mínimo necessário, explícito e revisável. Regra ampla
   ("`0.0.0.0/0`", "`*` no IAM", token de admin no CI) é achado, não conveniência.
3. **Infra é código — nada de mão no servidor.** Toda infra é **declarativa, idempotente e
   revisável** (IaC + `<projeto>_ops`), com `plan` antes do `apply` e **drift detection**. O
   servidor é **imutável por edição manual**: recebe só **artefato promovido do git** (mesmo
   commit SHA). "Editei direto no prd pra resolver rápido" é o anti-padrão que esta skill mata.
4. **Deploy gated onde a casa exige.** O fluxo é fixo e sem atalho: `dev local → teste local
   verde → GitHub → hml → prd`. **Nada pula etapa**; nada vai direto pra hml/prd. Onde a casa pede
   **deploy manual/gated** (ex.: o front na Vercel), respeite o gate — urgência não autoriza
   mão no servidor; hotfix segue o mesmo trilho, acelerado.
5. **Segredo nunca no repo.** Nunca em código, nunca em layer de imagem, nunca em `.env`
   commitado, nunca `NEXT_PUBLIC_`/`ARG`/`ENV` persistido. Cofre (Vault / Secret Manager /
   sealed-secrets) + **rotação documentada** + **secret scan que trava o merge**. Segredo achado
   no rastro → **rotação imediata**, não "depois eu troco".
6. **Observabilidade integrada, não posterior.** Todo serviço nasce com logs estruturados,
   métricas (RED/USE), traços correlacionados por `trace_id`, **SLO com error budget** e alertas
   **versionados como código**. Sem `/metrics` + dashboard + alerta + chart, o serviço **não é
   "pronto"** (§35 da engenharia). PII/segredo **nunca** em log.
7. **Backup com restore TESTADO.** RPO/RTO **definidos** por dado; backup cujo **restore nunca
   foi provado não é backup** — é esperança. O restore é ensaiado periodicamente (game day / DR
   drill) e o resultado mora no archive. Migração de dado é **reversível** (`down` + backup).
8. **Cadeia de suprimentos é superfície de primeira classe.** Lockfile commitado, **SBOM
   versionado junto ao artefato**, scan que **trava** em `high`/`critical` sem ADR, imagem
   **pinada por digest** (`@sha256`, nunca `latest`), **assinatura verificada na admissão** (deploy
   só aceita imagem assinada por pipeline confiável). Você entrega tudo que embarca.
9. **Tudo auditado, saída durável no archive.** Toda operação de infra passa pelo `<projeto>_ops`
   e **grava rastro** (§28): quem, quando, de qual commit, com qual resultado. Operação sensível
   vai pra **trilha de auditoria imutável**. O que não está no ops/IaC **não aconteceu**.

## Relação com as outras skills

- **schematize-engineering** — a **BASE**. Esta skill é o *como* de infra do que a engenharia
  normatiza: o **`<projeto>_ops` como interface única** e o **isolamento por usuário** (ops.md
  §2/§3), o **deploy e o fluxo de promoção** (operacao.md §21), a **observabilidade LGTM+** (§16),
  a **cadeia de suprimentos** (SBOM/pinning/assinatura) e o **archive** (§28). Onde a engenharia
  diz "o servidor é imutável por edição manual", esta skill diz **como** garantir isso.
- **schematize-pentest** — o **oráculo** do hardening. A infra fecha a porta (deny-by-default,
  least-privilege, mTLS, container hardened); a pentest **prova** que fechou (SSRF, exposição de
  segredo, escalonamento, cross-tenant). Um hardening "revisei e tá seguro" sem teste que prove a
  rejeição é suspeita, não fato.
- **schematize-web / go / rust / elixir / c# / zig / ruby / node** — a infra é agnóstica: ela
  provisiona e opera o serviço **de qualquer stack**; a build e o gate de cada item rodam na
  linguagem daquele repo (`/<slug>-review`, testes, lint). O **front** tem regra própria de
  deploy **manual/gated** (Vercel), respeitada aqui.
