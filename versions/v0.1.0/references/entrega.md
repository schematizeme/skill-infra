# Entrega — CI/CD gated, container e orquestração

> A casa promove software por um **trilho fixo** e, onde exige, **gated/manual**. Nada pula etapa,
> nada vai direto pro servidor, todo artefato é **imutável** e rastreável ao commit. Este
> reference é o *como* do fluxo de promoção da `schematize-engineering` (operacao.md §21) e da
> cadeia de suprimentos; a imagem/assinatura detalha em `resiliencia.md`.

O deploy não é "subir o que está na minha máquina". É **promover um artefato provado** por
estágios, com portão onde a casa quer olhar antes. A urgência não abre atalho — hotfix segue o
mesmo trilho, acelerado.

## 1. O fluxo de promoção — fixo, sem atalho

```
desenvolvimento local  →  teste local (verde)  →  GitHub (merge)  →  hml  →  prd
```

- **Nada pula etapa.** hml só recebe o que está no GitHub e passou no teste local; prd só recebe o
  que foi homologado em hml. Nunca direto pra hml/prd.
- **VETADO editar código direto no servidor (hml/prd).** O servidor é **imutável por edição
  manual** (`iac.md`) — recebe só **artefato promovido do git** (mesmo commit SHA). "Editei direto
  no prd" é o anti-padrão que o trilho existe pra matar.
- **Hotfix segue o mesmo fluxo, acelerado:** branch → teste local → git → hml → prd. Urgência
  **não** autoriza mão no servidor.

## 2. Deploy GATED/manual onde a casa exige

Nem todo deploy é automático — a casa **gateia** onde quer controle:

- **O front deploya manual/gated** (Vercel): o deploy é disparado **deliberadamente** (CLI
  prebuilt / promoção manual), não em todo push. Respeite o gate — é decisão da casa, não fricção
  a "consertar" ligando auto-deploy.
- **prd é sempre gated:** promoção com **aprovação humana** (environment protection do GitHub
  Actions, approval no pipeline). O CI **constrói e testa** sozinho; **promover pra prd** pede o
  portão.
- **`apply` de IaC em prd é gated** (`iac.md`): plan revisado → aprovação → apply.
- O gate é **auditável**: quem aprovou, qual artefato, qual commit — no archive.

## 3. Artefato imutável com proveniência git

- **Um artefato, todos os ambientes.** A **mesma imagem/commit SHA** que passou no CI vai pra hml
  e depois pra prd — não se rebuilda por ambiente (rebuild = artefato diferente = teste inválido).
- **Proveniência:** a tag da imagem **inclui o commit**; o deploy só aceita artefato com **origem
  git** (SHA). Build **só no CI**, nunca na máquina do dev pra produção (`resiliencia.md` §
  supply-chain).
- **Config por ambiente vem do seed/secret manager** no deploy, não embutida na imagem — a imagem
  é a mesma; o que muda é a config injetada (`segredos.md`, `iam` do ambiente).

## 4. Imagem de container — não-root, mínima, pinada

- **Multi-stage build:** toolchain só no estágio de build; a imagem final carrega **só o binário e
  o runtime necessário**.
- **Base mínima e pinada por digest** (`@sha256:...`), **nunca** tag móvel (`latest`/`1`).
  Distroless / chainguard / scratch / Alpine quando viável — menos pacote, menos CVE.
- **Não-root, filesystem read-only, `drop ALL` capabilities**, healthcheck na imagem
  (`isolamento.md` §3).
- **SBOM gerado no build e versionado junto ao artefato**; scan de imagem (`trivy`/`grype`) que
  **trava** em `high`/`critical` sem ADR; **imagem assinada** (cosign) e **assinatura verificada
  na admissão** (`resiliencia.md`).

## 5. Orquestração e estratégia de deploy

Kubernetes + Helm (chart versionado por serviço, entregue com o código); ou o `<projeto>_ops` em
alvos mais simples. A estratégia casa com o risco:

| Estratégia | Quando usar |
|---|---|
| Rolling update | Default para serviços comuns |
| Blue/green | Serviços críticos, rollback instantâneo necessário |
| Canary | Mudanças de alto impacto, rollout gradual com observação |

- **Healthcheck gating:** tráfego só vai pra pod **ready** (`/ready`); pod que não fica pronto não
  recebe request (`observabilidade.md` §healthchecks).
- **Rollback automatizado quando o healthcheck falha pós-deploy** — e rollback manual sempre à
  mão de um comando (`ops rollback`). Rollback é **artefato anterior** (imutável), não "reverte na
  mão".
- **Janela de validação** antes de declarar o deploy bem-sucedido (SLO/erro não regrediu).
- **Migração de schema é reversível** (`down` + backup) e roda **antes** do serviço que a consome,
  como dependência declarada no grafo de deploy (`resiliencia.md`, `iac.md`).

## 6. Feature flags — desacoplar deploy de release

- **Deploy ≠ release.** Código pode subir desligado atrás de flag e ser **ligado** depois
  (rollout por %, segmentação por tenant, **kill switch**, expiração da flag).
- Obrigatório pra **features críticas, migrações e rollouts graduais** — reduz o risco de cada
  deploy e dá reversão instantânea sem novo deploy.

## 7. Pipeline como base de confiança (TCB)

- **Build e teste no CI** (GitHub Actions), a partir do código versionado. O pipeline **é parte da
  base de confiança** — trate-o como produção.
- **Permissões mínimas no CI:** token escopado, sem segredo amplo; segredo via mecanismo efêmero,
  **nunca** em log de build (`segredos.md`). Secret scan + SAST + SCA + container scan travando o
  merge em `high`/`critical` (`resiliencia.md` § supply-chain).
- **Cada deploy grava rastro** no archive (§28): commit, artefato, quem aprovou, resultado.

> Regra de bolso: **promova, não copie.** Um artefato imutável, provado no CI, sobe por estágios
> com portão onde a casa quer olhar (prd, front). Nada pula etapa; nada é editado no servidor;
> rollback é voltar pro artefato anterior — nunca mexer na mão.
