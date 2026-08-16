# CLAUDE.md — Infra/DevOps/SRE da Casa (sempre on)

> Copie para a **raiz do repositório** e ajuste `<project>`. Fica pinado no contexto de toda
> tarefa e garante o piso mesmo quando a skill `schematize-infra` não dispara sozinha. Em repo
> multi-skill, use **junto** com os `CLAUDE.md` das skills de engenharia/linguagem (rode
> `/infra-claude` que mescla, sem sobrescrever os outros blocos).

## Regra mestre

Este sistema roda em produção com **blast radius mínimo, tudo declarado e tudo auditado**. A skill
`schematize-infra` rege como se provisiona, deploya e opera a infra da casa. Em conflito entre
"faço na mão rapidinho que resolve" e este piso, **o piso vence**. Consulte o reference antes de
agir — não trabalhe de memória. Segurança **ofensiva** (provar que fechou) é a `schematize-pentest`.

## Pisos inegociáveis (VETADO — sem exceção)

1. **Isolamento por aplicação.** Cada serviço roda confinado: **user Linux dedicado** + **systemd
   unit hardened** (`NoNewPrivileges`, `ProtectSystem=strict`, capabilities mínimas) + **container
   não-root/read-only**. **Nunca** `root`; **nunca** dois serviços no mesmo user. Blast radius
   contido é design.
2. **Deny-by-default e least-privilege em tudo.** Rede, IAM de nuvem, capabilities, filesystem,
   token de CI — começam **fechados**, abrem só o mínimo nomeado e revisável. `0.0.0.0/0` fora do
   ponto público, `"*"` no IAM, admin token no CI = **achado**, não conveniência.
3. **Infra é código — nada de mão no servidor.** Declarativa, idempotente, revisável (IaC +
   `<project>_ops`), com `plan` antes do `apply` e **drift detection**. O servidor é **imutável por
   edição manual**: recebe só artefato promovido do git (commit SHA). "Editei direto no prd" é
   VETADO — mão no servidor é break-glass **auditado**, nunca o fluxo.
4. **Deploy GATED onde a casa exige.** Fluxo fixo `dev local → teste local verde → GitHub → hml →
   prd`; **nada pula etapa**, nada vai direto pra hml/prd. Onde a casa pede **deploy manual/gated**
   (ex.: o **front na Vercel**), respeite o gate. Hotfix segue o mesmo trilho, acelerado.
5. **Segredo nunca no repo.** Nunca em código, layer de imagem, `.env` commitado, `NEXT_PUBLIC_`,
   `ARG`/`ENV`, log. Cofre (Vault/Secret Manager/sealed-secrets) + **rotação documentada** +
   **secret scan que trava o merge**. Segredo achado no rastro → **rotação imediata**.
6. **Observabilidade integrada, não posterior.** Todo serviço nasce com logs estruturados (sem
   PII), métricas RED/USE, traços por `trace_id`, **SLO com error budget** e alertas versionados.
   Sem `/health` `/ready` `/metrics` + dashboard + alerta + chart, o serviço **não é "pronto"**.
7. **Backup com restore TESTADO.** RPO/RTO definidos por dado; backup cujo restore nunca foi provado
   **não é backup**. Restore ensaiado periodicamente (DR drill); migração de dado **reversível**
   (`down` + backup). Apagar dado é gated, só dev/hml.
8. **Cadeia de suprimentos de primeira classe.** Lockfile commitado, **SBOM** versionado com o
   artefato, scan que **trava** em `high`/`critical` sem ADR, imagem **pinada por digest**
   (`@sha256`, nunca `latest`), **assinatura verificada na admissão**. Build só no CI.
9. **Tudo auditado.** Toda operação de infra passa pelo `<project>_ops` e **grava rastro** (§28);
   operação sensível vai pra **trilha de auditoria imutável**. O que não está no ops/IaC **não
   aconteceu**.

## Como se opera aqui

- **Deploy (`/infra-deploy`):** fluxo fixo + artefato imutável com proveniência git + estratégia
  (rolling/blue-green/canary) + healthcheck gating + rollback automatizado; respeita o gate
  manual (front/Vercel, prd).
- **IaC (`/infra-iac`):** declarativa/idempotente/revisável, `plan` antes do `apply`, drift
  detection, state seguro, layout `/<app>/` + seed, ops como interface única.
- **Segredos (`/infra-secrets`):** nada no repo/layer, cofre + rotação, secret scan que trava,
  referência pelo seed; vazou = rotaciona já.
- **Observabilidade/Resiliência:** LGTM+ integrado, SLO/alertas versionados; backup/DR com restore
  testado, rede deny-by-default, TLS que renova sozinho, supply-chain (SBOM/pinning/assinatura).

## Relação com as outras skills

- **schematize-engineering** — a base: o **ops como interface única** (§2), o **fluxo de deploy**
  (§21), a **observabilidade LGTM+** (§16), a **cadeia de suprimentos** e o **archive** (§28). Esta
  skill é o *como* de infra disso.
- **schematize-pentest** — o oráculo do hardening: a infra fecha a porta; a pentest **prova** que
  fechou. Hardening sem teste de rejeição é suspeita, não fato.

## Gestão de contexto (sessões longas)

Ao se aproximar do teto de contexto: **PARE e** gere o handoff em `<project>_archive/context/`
(estado da infra + FEITO vs EM ABERTO) **antes** de compactar (`/infra-cc`).
