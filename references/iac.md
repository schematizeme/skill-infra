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

## 7. Efeito externo fora de prd — DNS, rede e provedor como CÓDIGO

> Piso da casa: **efeito externo (e-mail, SMS/voz, push, webhook de terceiro, cobrança) NUNCA sai
> de não-produção**. A normativa é a `schematize-engineering` → `references/efeitos-externos.md`
> (as **4 camadas** de defesa em profundidade). Esta skill é a **dona das camadas 1 (DNS), 3
> (rede/segredo) e 4 (provedor)** — e aqui elas viram **IaC declarada, revisada e verificada**, não
> combinado verbal. A camada 2 (sink por default + guard dentro do provider + cap por execução) é
> da engenharia e da skill de linguagem.
>
> **Motivo — cite sempre:** um laço de teste disparou **>5.000 e-mails reais** para endereços
> sintéticos. Hard bounce e spam trap em massa, **reputação de IP e domínio queimada**, o
> transacional de **produção** parando de chegar — inclusive o **OTP de login** —, **semanas de
> warm-up** pra recuperar, utilidade **zero**. Não tem undo. Por isso é infra, não recomendação.

### 7.1 Camada 1 — o domínio de teste em ROTA NULA (Terraform/Cloudflare)

`test.<domain>` existe **só pra não entregar**. **Null MX (RFC 7505)** faz o MTA remetente falhar
na hora — sem fila, sem retry, sem backscatter; **SPF `-all` + DMARC `p=reject`** impedem que ele
seja forjado como remetente e **isolam a reputação dele** da do domínio corporativo.

```hcl
# ops/iac/dns_test_domain.tf — infra como qualquer outra: plan -> review -> apply
locals {
  test_domain = "test.${var.domain}"   # destino de TODO endereco sintetico da casa
}

# Null MX (RFC 7505): "MX 0 ." = este dominio NAO aceita e-mail. O "." e literal (raiz do DNS).
resource "cloudflare_record" "test_null_mx" {
  zone_id  = var.cloudflare_zone_id
  name     = local.test_domain
  type     = "MX"
  priority = 0
  content  = "."
  proxied  = false
  comment  = "RFC 7505 null MX - rota nula do dominio de teste (piso: efeitos externos)"
}

# Ninguem envia em nome do dominio de teste.
resource "cloudflare_record" "test_spf" {
  zone_id = var.cloudflare_zone_id
  name    = local.test_domain
  type    = "TXT"
  content = "v=spf1 -all"
}

# Falha de alinhamento = REJEITA (nunca p=none, nunca p=quarantine).
resource "cloudflare_record" "test_dmarc" {
  zone_id = var.cloudflare_zone_id
  name    = "_dmarc.${local.test_domain}"
  type    = "TXT"
  content = "v=DMARC1; p=reject; sp=reject; rua=mailto:dmarc@${var.domain}"
}
```

- **Sem zona própria?** TLD reservado de **RFC 2606/6761**: `@*.test`, `@*.invalid`, `@*.example`.
  Não resolvem em lugar nenhum do mundo, **por norma** — rota nula sem precisar de DNS.
- **VETADO** apontar o domínio de teste pra uma caixa real "só pra dar uma olhada"; **VETADO**
  criá-lo como subdomínio do domínio que **envia** produção (contamina exatamente a reputação que
  se quer proteger).
- **Registro sem verificação é fé, não infra.** A prova roda no `ops doctor` e no CI:

```bash
dig +short MX  "test.${DOMAIN}"          # esperado: 0 .
dig +short TXT "test.${DOMAIN}"          # esperado: "v=spf1 -all"
dig +short TXT "_dmarc.test.${DOMAIN}"   # esperado: v=DMARC1; p=reject
```

### 7.2 Camada 4 — subdomínio de ENVIO separado por ambiente (reputação isolada)

Reputação é **por domínio/subdomínio de envio e por IP**. Se hml e prd enviam pelo mesmo
remetente, um laço em hml **queima o login de prd**. Por isso o remetente é **por ambiente**,
declarado na IaC:

| Ambiente | Subdomínio de envio | Provedor | Chave |
|---|---|---|---|
| `prd` | `mail.<domain>` | conta/pool de produção, com warm-up próprio | chave **de prd**, só no cofre de prd |
| `hml` | `mail-hml.<domain>` | **sandbox** do provedor (ou pool/subconta separada) | chave **sandbox** |
| `dev` | **não envia** — sink local (Mailpit) | — | **nenhuma** |

- **DKIM/SPF/DMARC são próprios de cada subdomínio de envio** (chave DKIM por subdomínio), e
  também declarados na IaC — nunca colados à mão no painel do provedor (vira drift invisível).
  O SPF do remetente inclui **só o provedor daquele ambiente**: `v=spf1 include:<provedor> -all`.
- O subdomínio de não-prd **existe pra suportar a exceção com ADR** (`efeitos-externos.md` §5), não
  pra "poder mandar quando precisar". Sem as cinco condições, o ambiente segue no **sink**.
- **VETADO** reusar o subdomínio de prd em hml/dev, e **VETADO** enviar pelo domínio raiz
  (`<domain>`) — o raiz carrega o e-mail corporativo; queimar ele é queimar a empresa.
- No provedor (SES/Resend/Postmark): **configuration set / message stream separado por ambiente**,
  com o não-prd em **sandbox mode** (SES sandbox só entrega a endereço verificado — mantenha assim).

### 7.3 Camada 3 — egress SMTP bloqueado em dev/hml (o bypass "chamei o SDK direto")

Guard de aplicação não protege de quem **não passa pelo provider**: script solto, `sendmail` do
host, biblioteca chamada direto, container de terceiro. A rede é a camada que não depende de
ninguém lembrar: **fora de prd, as portas de envio simplesmente não existem** — 25, 465, 587 e
**2525** (a porta alternativa que todo provedor oferece e todo mundo esquece de fechar).

**Kubernetes — NetworkPolicy é allowlist: o SMTP morre por AUSÊNCIA de regra.**

```yaml
# ops/iac/k8s/netpol-egress-hml.yaml — namespace de nao-producao
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: egress-allowlist, namespace: hml }
spec:
  podSelector: {}                 # todos os pods do namespace
  policyTypes: [Egress]
  egress:
    - to: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } } }]
      ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]   # DNS
    - to: [{ ipBlock: { cidr: 0.0.0.0/0 } }]
      ports: [{ protocol: TCP, port: 443 }]                               # HTTPS de saida
    - to: [{ podSelector: { matchLabels: { app: mailpit } } }]
      ports: [{ protocol: TCP, port: 1025 }]                              # o SINK, esse sim
# NAO EXISTE regra para 25/465/587/2525 -> egress SMTP bloqueado por deny-by-default.
```

**AWS — o security group é allow-only; o NACL é quem NEGA explicitamente.**

```hcl
# SG de app em nao-prd: sem a regra ampla de egress; so 443 (+ VPC endpoints).
resource "aws_security_group" "app_nonprd" {
  name   = "app-${var.env}"
  vpc_id = var.vpc_id
  egress {                                   # HTTPS de saida, e so
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # Sem egress 25/465/587/2525: bloqueio por ausencia (deny-by-default do SG).
}

# Cinto E suspensorio: NACL nega o SMTP mesmo se alguem "abrir tudo" num SG por engano.
locals { smtp_ports = [25, 465, 587, 2525] }

resource "aws_network_acl_rule" "deny_smtp_egress" {
  count          = length(local.smtp_ports)
  network_acl_id = var.nonprd_nacl_id
  rule_number    = 100 + count.index          # antes de qualquer regra de allow
  egress         = true
  protocol       = "tcp"
  rule_action    = "deny"
  cidr_block     = "0.0.0.0/0"
  from_port      = local.smtp_ports[count.index]
  to_port        = local.smtp_ports[count.index]
}
```

> A AWS já bloqueia a **porta 25** de saída por default em EC2 — **465/587/2525 não**. Contar com
> o default da nuvem é meia camada; declare as quatro.

**Host bare-metal/systemd (o padrão da casa em `/<app>/`) — nftables aplicado pelo ops:**

```nft
# /etc/nftables.d/50-no-smtp-egress.nft  (dev/hml apenas; aplicado por `ops install`)
table inet filter {
  chain output {
    type filter hook output priority 0; policy accept;
    tcp dport { 25, 465, 587, 2525 } counter reject with icmpx type admin-prohibited \
      comment "efeito externo: egress SMTP bloqueado fora de prd"
  }
}
```

- **`reject`, não `drop`.** `drop` faz o processo pendurar até o timeout e o dev culpar "a rede
  tá lenta"; `reject` **falha na hora**, com erro legível, e ensina o caminho (mensagem acionável).
- O `counter` da regra é **exportado como métrica**: contador de SMTP bloqueado subindo em hml é
  **incidente** (alguém tentou enviar de verdade), não curiosidade — alerte (`observabilidade.md`).
- Vale igual pro resto: em não-prd, **egress só pro sink** de webhook/SMS/PSP; URL de parceiro e
  endpoint de PSP real **não são alcançáveis** de dev/hml.

### 7.4 O gate — as camadas de infra são PROVADAS, não declaradas

- [ ] `test.<domain>` responde **null MX + SPF `-all` + DMARC `p=reject`** (provado por `dig` no CI/`ops doctor`).
- [ ] Subdomínio de envio **separado por ambiente**, com DKIM/SPF próprios na IaC (sem drift no painel).
- [ ] Egress **25/465/587/2525 bloqueado** em dev/hml (SG/NACL/NetworkPolicy/nftables no código, com prova).
- [ ] Chave de provedor de não-prd é **sandbox**; a de prd **não existe** fora de prd (`segredos.md` §7).
- [ ] **Alerta** de bounce > 2% / complaint > 0,1% e de envio real fora de prd (`observabilidade.md` §9).

Falta uma camada → é **bug de engenharia de infra**, não descuido de quem rodou o laço.

> Regra de bolso: **se não dá pra jogar o servidor fora e reconstruir do git com um comando, a
> infra não é código — é um bicho de estimação.** Declarativo, idempotente, revisável, com plan
> antes do apply e drift que alerta. Mão no servidor é break-glass auditado, nunca o normal.
