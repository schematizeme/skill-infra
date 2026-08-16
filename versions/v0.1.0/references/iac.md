# IaC — infra é código: declarativa, idempotente, revisável

> A casa **não opera servidor na mão**. Toda infra nasce de **código versionado** — declarativo,
> idempotente e revisável — e é aplicada por uma ferramenta, com `plan` antes do `apply` e **drift
> detection**. O servidor é **imutável por edição manual**: recebe só artefato promovido do git.
> Este reference é o *como* do "nada de mão no servidor" da `schematize-engineering` (operacao.md
> §21, ops.md §2). O control plane que executa é o **`<projeto>_ops`**.

O que não está em código **não existe**: não é reproduzível, não é revisável, não sobrevive a uma
troca de máquina. "Cliquei no console da nuvem", "editei o `nginx.conf` no servidor", "rodei um
`kubectl edit` na correria" são todos a mesma dívida — **drift** que ninguém consegue reconstruir.

## 1. Os três atributos inegociáveis

- **Declarativa.** Você descreve o **estado desejado** (o que deve existir), não a sequência de
  passos (Terraform/OpenTofu, Helm, Kustomize, cloud-init, Ansible declarativo). A ferramenta
  converge o mundo pro estado; você não "faz", você **declara**.
- **Idempotente.** Aplicar N vezes = aplicar 1 vez. Rodar `apply` de novo num estado já
  convergido **não muda nada** (`plan` vazio). Script que quebra na segunda execução, ou que
  acumula recurso a cada run, **não é IaC** — é script imperativo disfarçado.
- **Revisável.** A mudança de infra passa por **PR + review + CI**, igual código de app.
  `terraform plan` no PR mostra o diff **antes** de tocar produção; ninguém aprova o que não viu.
  Branch protegida, review obrigatório em infra de prd.

## 2. Plan-before-apply — nunca aplica às cegas

- **`plan` sempre antes do `apply`.** O `plan` é o **diff do mundo**: o que vai criar/alterar/
  **destruir**. Ler o plan é obrigatório; `apply` sem plan revisado é o `git push --force` da infra.
- **Atenção redobrada a `destroy`/`replace`** no plan — recurso com estado (banco, volume, IP
  estável) que o plan quer **recriar** é um alarme: pode significar perda de dado. Investigue antes
  de aprovar (troca de atributo imutável costuma forçar replace).
- **Apply gated em prd** (`entrega.md`): plan revisado → aprovação → apply. Preferir apply pelo
  **CI/ops**, não da máquina do dev (proveniência + auditoria).

## 3. State — a fonte da verdade, protegida

O state (Terraform/OpenTofu) é sensível e crítico:

- **State remoto, versionado e com lock** (S3+DynamoDB, GCS, backend gerenciado) — nunca state
  local commitado, nunca dois `apply` concorrentes sem lock (corrompe).
- **State pode conter segredo** (ex.: senha gerada) — trate o backend como **dado confidencial**
  (cifrado em repouso, acesso restrito). Não versione state no git.
- **Sem edição manual do state** (`state rm`/`import` só com intenção clara e registrada). Drift
  entre state e mundo real é bug a corrigir, não a ignorar.

## 4. Drift detection — o mundo não pode divergir do código

- **Drift = o servidor divergiu do git.** Alguém editou na mão, um processo mudou algo, a nuvem
  reconciliou diferente. O ops/IaC **detecta e alerta** (`terraform plan` agendado, `driftctl`,
  reconciliação do GitOps).
- **A correção do drift é re-aplicar o código, não editar de novo na mão.** Se o código está certo
  e o mundo divergiu, o `apply` traz de volta. Se a mudança manual era legítima, ela **vira
  código** — e aí some do drift. O drift nunca "vira o normal".
- **Filesystem read-only em hml/prd + acesso de escrita = break-glass auditado** — a exceção
  registrada no archive, nunca o fluxo (`operacao.md` §21).

## 5. GitOps / artefato imutável — o git é o trilho

- **O git é a fonte da verdade do estado desejado.** Merge no branch → o reconciliador (Argo CD/
  Flux) ou o `ops apply` converge o cluster. O que está no cluster == o que está no git.
- **Artefato imutável com proveniência git.** O deploy consome a **mesma imagem/commit SHA** que
  passou no CI (`entrega.md`) — nada de build na máquina do dev pra prd, nada de tag móvel. A tag
  da imagem **inclui o commit**.
- **Redeploy nasce do seed e é destrutivo na app, nunca nos dados** (`isolamento.md` §5): apaga o
  clone/build/config, recria do `/<app>/.env`, idempotente. Dado persistente só muda por
  **migration reversível** (`down` + backup — `resiliencia.md`).

## 6. O `<projeto>_ops` é a interface única (100% das operações)

- **Toda** operação — provisionar, subir, atualizar, configurar, migrar, reverter, diagnosticar —
  passa por um **comando do ops**. **Zero** exceção: proibido `ssh servidor` + comando ad-hoc,
  `kubectl edit`/`docker` na mão, script solto. Se **não existe** comando pra aquilo, **cria o
  comando no ops** — não faz por fora.
- **Superfície mínima, idempotente, com `--help` e saída machine-readable:** `bootstrap` ·
  `install`/`up` · `redeploy` (do seed) · `update` · `config` · `migrate` (reversível) ·
  `health`/`doctor` · `rollback` · `logs` · `reset` (destrói dados — gated, dev/hml).
- **Autonomia:** um humano provisiona o servidor **do zero** só com o ops, sem depender da IA.
  `ops install` sobe tudo; `ops redeploy` recria do seed; `ops doctor` diagnostica.
- **Instalação paralela por padrão** (= `nproc`); falha no paralelo = bug de independência,
  prioridade máxima (`isolamento.md` §6). Cada operação **grava o archive** (§28).

> Regra de bolso: **se não dá pra jogar o servidor fora e reconstruir do git com um comando, a
> infra não é código — é um bicho de estimação.** Declarativo, idempotente, revisável, com plan
> antes do apply e drift que alerta. Mão no servidor é break-glass auditado, nunca o normal.
