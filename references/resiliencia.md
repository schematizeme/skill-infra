# Resiliência — backup & DR, rede/TLS/hardening, cadeia de suprimentos

> Três frentes que decidem se o sistema **sobrevive** ao dia ruim: o dado volta depois do desastre
> (**backup & DR com restore testado**), a porta está fechada por padrão (**rede/TLS/hardening
> deny-by-default**) e o que você embarca é o que você acha que embarcou (**cadeia de
> suprimentos**). Este reference consolida o *como* de infra do §21/§13 da
> `schematize-engineering` e da `cadeia-suprimentos.md`.

## 1. Backup & DR — RPO/RTO e o restore que se PROVA

- **RPO e RTO definidos por dado.** **RPO** (quanto de dado você aceita perder — a janela entre
  backups) e **RTO** (quanto tempo até voltar) são **decisão de negócio, escrita**, não implícita.
  Dado de billing tem RPO/RTO diferente de cache.
- **Backup cujo restore NUNCA foi testado NÃO é backup — é esperança.** O piso: **restore
  ensaiado periodicamente** (DR drill / game day) num ambiente limpo, cronometrado contra o RTO,
  verificado contra o RPO. Backup que ninguém restaurou é dado corrompido esperando o incidente pra
  se revelar.
- **3-2-1:** 3 cópias, 2 mídias, 1 offsite. Backup **cifrado** (em trânsito e repouso) e com
  **acesso separado** do sistema principal (ransomware não pode cifrar o backup junto). Considere
  cópia **imutável**/WORM pra billing/auditoria.
- **Backup ANTES de migração destrutiva** (`iac.md`, `entrega.md`): migração de schema é
  **reversível** (`down` testado) e precedida de backup. `reset`/apagar dado é comando **gated, só
  dev/hml**, nunca prd (`isolamento.md` §5).
- **Runbook de DR** ensaiado: passo a passo de "a região caiu, e agora?" — quem faz o quê, em que
  ordem, com que comando do ops. Registre o resultado do drill no archive.

## 2. Redes — deny-by-default e segmentação

- **Tudo começa fechado.** Firewall / security group / NetworkPolicy **nega por padrão**; abre só
  o fluxo necessário e **nomeado** (`isolamento.md` §4). Sem `0.0.0.0/0` fora do ponto de entrada
  público (e esse atrás de WAF/proxy). **Egress** também restrito — corta exfiltração e SSRF de
  saída.
- **Segmentação:** public / private / data subnets; o banco **nunca** exposto à internet; bastion/
  túnel para acesso administrativo, não IP público no Postgres.
- **Zero-trust interno:** identidade de serviço (mTLS/SPIFFE), não confiança por estar na mesma
  rede. "Está na VPC, então é confiável" é premissa furada.

## 3. TLS e criptografia — em trânsito e repouso

- **TLS 1.2+ (prefira 1.3)** em toda borda pública; cifras fortes, HSTS. **Sem TLS terminado e
  esquecido**: certificado **automatizado** (cert-manager / ACME) com **renovação automática** —
  certificado vencido é incidente evitável.
- **mTLS interno** entre serviços onde a malha permite (autenticação mútua de serviço).
- **Criptografia em repouso** para dado pessoal/sensível (disco, banco, backup — §32 LGPD da
  engenharia). Chave no cofre, rotacionável (`segredos.md`).

## 4. Hardening — least-privilege em todas as camadas

- **IAM de nuvem least-privilege:** política com **ação e recurso específicos**, nunca `"*"`.
  Sem chave estática de longa vida — **OIDC/workload identity** com credencial de curta duração
  (`segredos.md`). Sem credencial de root/owner em automação.
- **Container/host hardened** (`isolamento.md`): não-root, read-only, `drop ALL` capabilities,
  `seccomp`, `NoNewPrivileges`, imagem mínima. Host com patch em dia, SSH por chave (sem senha),
  sem porta desnecessária aberta.
- **Superfície mínima:** menos serviço exposto, menos pacote instalado, menos permissão concedida
  = menos CVE e menos caminho de ataque. Deny-by-default é o tema que atravessa tudo.
- **A pentest prova o hardening.** Fechar a porta é infra; **provar que fechou** (SSRF, exposição
  de segredo, escalonamento de privilégio, cross-tenant) é a `schematize-pentest`. Hardening
  "revisei e tá seguro" sem teste que prove a rejeição é **suspeita, não fato**.

## 5. Cadeia de suprimentos — você entrega tudo que embarca

O ataque hoje raramente é no seu código: é numa dependência, numa imagem base ou no pipeline. Trate
build e dependência como **superfície de ataque de primeira classe** (`cadeia-suprimentos.md`).

- **Dependências:** **lockfile commitado**, build reproduzível a partir dele; verificar **nome e
  licença** de toda dependência nova (typosquatting é real; allowlist MIT/Apache-2.0/BSD/MPL-2.0/
  ISC — GPL/AGPL/SSPL/proprietária só com ADR); minimizar superfície transitiva. **VETADO** fixar
  por branch (`main`) em prd e `curl | sh` de origem não verificada no build.
- **SBOM:** gerar (CycloneDX/SPDX) no build e **versionar junto ao artefato** — é o inventário do
  que foi de fato embarcado (`syft`, `cargo cyclonedx`).
- **Scan que trava:** SAST (Semgrep/CodeQL), SCA (deps + licenças), container scan (Trivy/Grype),
  secret scan (Gitleaks) — **travam o merge** em `high`/`critical` sem ADR de aceite datado.
  Vulnerabilidade aceita conscientemente vira **ADR com prazo**, não silêncio.
- **Imagem:** base **pinada por digest** (`@sha256`), multi-stage, não-root/read-only; rebuild
  periódico pra absorver patch da base (`entrega.md` §4).
- **Proveniência (SLSA):** **build só no CI** (não na máquina do dev pra prd), a partir do código
  versionado — o pipeline é parte da TCB. **Assinar** o artefato/imagem (cosign/sigstore) e
  **verificar a assinatura na admissão** (deploy só aceita imagem assinada por pipeline confiável —
  assinar sem verificar não protege). **Atestação de build**: quem buildou, de qual commit, com
  qual SBOM; a tag inclui o commit.
- **Segredo no build:** nenhum segredo em layer (`segredos.md` §1); BuildKit secret mount efêmero;
  permissões mínimas no CI (token escopado).

## 6. Resiliência de runtime — degradação graciosa

- **Cada serviço sobe sozinho**, sem exigir outro no boot — **degradação graciosa** (outbox/retry/
  fila/circuit breaker) em vez de cascata. É o mesmo invariante do "falha no paralelo = bug de
  independência" (`isolamento.md` §6). Um serviço lento/fora não derruba os que dependem dele: eles
  degradam, não caem.
- **Timeout, retry com backoff e circuit breaker** em toda chamada externa/fila/banco. Sem
  timeout, uma dependência lenta vira thread exhaustion e o serviço inteiro trava.
- **Limite de recurso** (`MemoryMax`/limits do k8s) — um serviço em loop não estrangula o host
  (`isolamento.md` §2).

> Regra de bolso: **backup só conta depois do restore provado; a porta só conta fechada por
> padrão; o artefato só conta se você sabe exatamente o que tem dentro e quem o produziu.** RPO/RTO
> escritos e ensaiados, deny-by-default na rede/IAM, TLS que renova sozinho, SBOM + pinning +
> assinatura verificada. Sobreviver ao dia ruim é design, não sorte.
