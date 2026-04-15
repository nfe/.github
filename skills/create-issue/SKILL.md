---
name: create-issue
description: Cria uma issue no GitHub seguindo o padrão nfe/management (Padrao_Issues.md + labels-catalog.md). Captura obrigatoriamente a origem da solicitação (quem pediu, canal e link/rastro) e exige pelo menos uma referência no corpo. Recebe texto livre, infere campos, conversa para preencher gaps, valida DoR e executa `gh issue create`. Use quando o usuário quiser abrir/criar uma issue, registrar um item de backlog, ou converter uma ideia em issue estruturada.
---

# create-issue

Cria issues no padrão da org `nfe` a partir de texto livre, de forma conversacional.

## Fontes de verdade (leia sempre antes de usar)

1. `/mnt/c/w/nfeio/management/operacional/Padrao_Issues.md` — estrutura do corpo, DoR, anti-padrões
2. `/mnt/c/w/nfeio/ops/rice/labels-catalog.md` — **catálogo canônico**. Toda a lista de valores válidos de `area:*` produto, `area:*` transversal, `type:*`, `prio:*`, `sev:*`, `deadline:*` deve ser lida daqui em runtime (parse da §2A para produto, §2B para transversal, §1 para type, etc). Não mantenha listas hardcoded nesta skill.
3. Este arquivo — fluxo conversacional

Se algum arquivo não existir no disco, pare e avise o usuário.

**Check de sanidade ao iniciar:** se detectar duas cópias de `labels-catalog.md` no repo (ex: `ops/rice/labels-catalog.md` e `management/operacional/labels-catalog.md` ambos como arquivos regulares), avise o usuário — é anti-padrão, pois duas fontes divergem silenciosamente. Sugerir consolidar em uma e deixar a outra como stub com link.

## Defaults

- **Repo destino:** `nfe/nfse-test` (confirmar a cada execução)
- **GitHub Project:** `Work` (org `nfe`, número `3`) — adicionar a issue ao projeto após criar
- **Banda RICE:** estimada empiricamente pela skill (heurística abaixo). **Não** aplicada como label na criação (o padrão diz que `banda:*` é calculada na sessão de priorização). A estimativa vai apenas como comentário no corpo: `<!-- banda sugerida: X — estimativa empírica, aplicar como label só após sessão de priorização com Metabase -->`.

## Memória de sessão

Se o usuário alterou defaults em execução anterior (repo ou project), a skill pode salvar na memória persistente (`/home/kita/.claude/projects/-mnt-c-w-nfeio-management/memory/`) como memória `user` ou `project` e oferecer esses valores na Etapa 0.1 em vez dos defaults originais.

## Fluxo conversacional obrigatório

Sempre segue 13 etapas, uma pergunta por vez, usando `AskUserQuestion`. Em toda etapa:

- Se fizer inferência, confirmar explicitamente antes de travar
- Se houver ambiguidade, reforçar com pergunta de esclarecimento antes de pedir confirmação
- Usuário pode responder `pular` a qualquer pergunta:
  - Campo **obrigatório (DoR)**: eu preencho best-effort, marco como `<!-- inferido, revisar -->` no corpo, e sinalizo no preview final
  - Campo **não obrigatório**: fica vazio
- Usuário pode responder `voltar` para refazer a etapa anterior

### Etapa 0 — Eco de entendimento
Resuma em 1 frase o que entendi do texto livre. Pergunte: "Sigo com esse entendimento? [sim / reformular]"

### Etapa 0.1 — Destino (repo + project) + pré-voo de labels

Confirmar destino:
- Repo: `nfe/nfse-test` (ou valor recuperado da memória de sessão)
- Project: `Work` (org `nfe`, #3)

Opções: `[usar defaults / alterar repo / alterar project / alterar ambos / sem project]`. Travar valores para Etapas 11 e 12.

**Pré-voo de labels — obrigatório:**

1. Executar `gh label list --repo <owner/repo> --limit 200` e capturar as labels existentes.
2. Comparar com o catálogo (parse de `labels-catalog.md` §2A, §2B, §1, §3, §4, §5, §6 — famílias `type:*`, `area:*`, `prio:*`, `sev:*`, `deadline:*`, `must-do`).
3. Se faltar qualquer label do catálogo no repo:
   - **Parar o fluxo.**
   - Mostrar a lista de labels faltantes.
   - Avisar: *"O repositório `<owner/repo>` não está configurado conforme o catálogo. É necessário configurar o projeto antes de criar issues. Execute a configuração de labels (responsável pelo repo) e rode o /create-issue novamente."*
   - **Não criar labels automaticamente.** Criar labels é decisão de configuração de projeto, não da skill.
   - Oferecer: `[parar / continuar mesmo assim (labels ausentes viram `<!-- label ausente no repo: X -->` no corpo)]`.

### Etapa 0.2 — Detecção de duplicatas

Antes de entrar nas perguntas de conteúdo, executar:

```bash
gh issue list --repo <owner/repo> --state open --search "<palavras-chave-do-texto-livre>" --limit 10
```

Se houver resultados, apresentar até 5 candidatas com `[<num> <título>]` e perguntar:
- `[nenhuma é duplicata / X é duplicata, abortar / X é relacionada, continuar e linkar]`

Se "relacionada, continuar", guardar para colocar em Dependências ou Referências na Etapa 9.

### Etapa 0.3 — Origem da solicitação (obrigatória)

Capturar o rastro formal de quem pediu e por qual canal, para garantir rastreabilidade e evitar issues de "quem pediu mesmo isso?".

Perguntar em sequência:

1. **Quem solicitou?** (ex: `@fulano`, time de CS, análise interna, regulamentação)
2. **Por qual canal?** (ex: Slack `#canal`, e-mail, reunião, ticket de suporte `#ID`, sprint review, análise de dados)
3. **Link ou referência:** URL, Slack permalink, número de ticket, ata de reunião (`AAAA-MM-DD`), ou fallback estruturado ex: `reunião 2026-04-15 com time X`

Se o usuário não fornecer link, perguntar ativamente:
> *"Tem um link do Slack, ticket, e-mail ou documento que originou esse pedido? Se não tiver, algo como 'reunião 2026-04-15 com @fulano' também serve."*

**O que fazer com a resposta:**
- Montar o bloco de origem: `> 📌 **Origem:** <quem> via <canal> — <link/referência>`
- Inserir automaticamente no corpo da issue como **primeira linha**, antes da seção `## Contexto`
- Se tiver link que também é referência, pré-popular a seção **Referências** com esse link (economiza trabalho na Etapa 9)
- Se o usuário explicitamente não tiver nenhum rastro: aceitar `sem rastro disponível — [motivo]` e marcar como TRIAGE com follow-up

### Etapa 1 — `type:*` (obrigatório)
Ler valores válidos da §1 do catálogo em runtime. Inferir do texto. Sugerir 1 valor + alternativas via AskUserQuestion.

### Etapa 2 — `area:*` produto (0 ou 1) — duas perguntas em cadeia

Como o catálogo tem >4 valores de produto (limite do AskUserQuestion), dividir em 2 perguntas:

**2a — Família:** `[emissao-* / consulta-* / plataforma-core (tax, identity, notification, usage, empresas, prefeituras) / nenhuma (transversal puro)]`

**2b — Valor específico:** apresentar apenas os valores da família escolhida, lidos do catálogo em runtime. Ex: se "emissao-*", mostrar `emissao-nfe`, `emissao-nfce`, `emissao-nfse`.

**Anti-padrão #5:** se o texto sugerir 2+ produtos, perguntar antes das duas etapas acima se quebra em issues separadas. Se "sim, guarda-chuva + N filhas", **setar flag para Etapa 13** (após criar a mãe, entrar no loop de criar filhas automaticamente).

### Etapa 3 — `area:*` transversais (0..N)
Ler valores válidos da §2B do catálogo em runtime. Detectar por palavras-chave do texto. AskUserQuestion com multiSelect.

### Etapa 4 — `deadline:*` (condicional)
Só pergunta se `area:compliance`. Ler §5 do catálogo. Se o texto mencionou data/mês vago, reforçar: "Qual data exata? homologação ou produção? é prazo legal ou interno?"

### Etapa 5 — `sev:*` (condicional)
Só pergunta se `type:bug` ou `type:incident`. Ler §4 do catálogo.

### Etapa 6 — `prio:*` (obrigatório)
Ler §3 do catálogo. `p0` para compliance vigente/le90d ou sev 1; `p1` para sev 2; `p2` default; `p3` para icebox/spike especulativo.

### Etapa 7 — Título
Formato `[Área] Verbo + objeto + qualificador`, ≤80 chars. Se sem `area:*` produto, usar rótulo descritivo (ex: `[API Prefectures]`, `[Docs]`, `[Infra]`).

### Etapa 8 — Assignee
Pergunta aberta. Se usuário pular, marca como `<!-- definir responsável -->`.

### Etapa 9 — Corpo (seção por seção)

Ordem: **Origem → Contexto → Objetivo → Escopo Inclui → Escopo Não-inclui → Critérios de Aceitação → Dependências → Referências**.

Para cada seção: `[eu rascunho / você dita / pular, eu preencho / pular, deixar vazio]`.

**Seção Origem:** já capturada na Etapa 0.3 — inserir automaticamente sem perguntar novamente. Se por algum motivo estiver faltando, perguntar agora (mesmas perguntas da Etapa 0.3).

**Reforço ativo** quando informação é insuficiente:
- Contexto vazio → "qual problema atual? o que acontece se não fizer? quem é impactado?"
- Critério subjetivo → pedir métrica concreta
- Escopo amplo → perguntar se quebra em sub-issues (se sim, voltar para Etapa 2 anti-padrão #5 + setar flag Etapa 13)

**Seção Referências — obrigatória:**
Deve ter pelo menos 1 entrada. Exemplos válidos: links, docs, tickets, normas, threads, PRs, ou fallback: `sem referência externa — [motivo]`.

Se o link capturado na Etapa 0.3 for uma referência útil, pré-popular aqui automaticamente.

Se o usuário tentar pular e a seção estiver vazia, perguntar ativamente:
> *"Tem algum documento, regulamentação, discussão ou ticket que embasou essa demanda? Pode ser o mesmo link da Origem."*

Se ainda assim não houver: aceitar, mas **obrigatoriamente** abrir em TRIAGE com o placeholder visível:
```markdown
## Referências
- TODO: sem referência externa confirmada — validar antes de sair de Triage
```

**Quando uma seção fica marcada como `<!-- inferido, revisar -->`**, adicionar **automaticamente** ao final do corpo um bloco de follow-up:

```markdown
## Follow-up (completar antes de sair de Triage)
- [ ] Definir responsável
- [ ] Validar contexto com quem demandou a issue
- [ ] <outros itens específicos segundo o que foi inferido>
```

Isso transforma rascunhos fracos em itens acionáveis em vez de buracos no backlog.

### Etapa 10 — Validação DoR + banda estimada

Checar DoR (§5 do padrão). Se falhar: mostrar o que falta e perguntar `[completar agora / abrir como rascunho Triage]`.

DoR inclui agora:
- [ ] Seção **Origem** preenchida (Etapa 0.3)
- [ ] Seções **Contexto** e **Objetivo** preenchidas
- [ ] Pelo menos 1 Critério de Aceitação verificável
- [ ] Seção **Referências** com ao menos 1 entrada (link, doc, ticket, ata, ou `TODO: sem referência...`)
- [ ] Labels obrigatórias aplicadas
- [ ] Assignee definido
- [ ] Dependências listadas

**Estimar banda empiricamente** (comentário no corpo, **não** como label):

- **alta** (≥ 2,0): `area:compliance + deadline:vigente|le90d`; `type:bug|incident + sev:1|2`; reach amplo
- **media** (≥ 0,1): `type:feature|improvement` com produto + contexto de cliente; bugs sev 3
- **baixa** (< 0,1): `type:spike`, `type:docs`, tech debt vago, bugs sev 4

Aplicar no corpo como:
`<!-- banda sugerida: X — estimativa empírica pela skill, NÃO aplicar como label; confirmar na sessão de priorização com Metabase -->`

### Etapa 11 — Preview final + execução

Preview completo:
- Título
- **Status DoR** (✅ OK / ⚠️ TRIAGE — com lista do que falta destacada)
- Labels a aplicar (sem `banda:*`)
- Assignee
- Repo destino
- Project destino
- Corpo em markdown

Se DoR falhou, preview deve começar com:

```
⚠️ ISSUE SERÁ ABERTA EM TRIAGE — DoR incompleto
Pendências: <lista>
```

Perguntar: `[criar / editar / cancelar]`.

Se `criar`:

```bash
gh issue create \
  --repo <owner/repo> \
  --title "<título>" \
  --body-file <arquivo-temp.md> \
  --label "type:X,area:Y,prio:Z,..." \
  --assignee <usuario>
```

Capturar a URL retornada.

### Etapa 12 — Adicionar ao GitHub Project

Se Project configurado na Etapa 0.1, executar (sintaxe correta — número é posicional):

```bash
gh project item-add <project-number> --owner <project-owner> --url <issue-url>
```

Default: `gh project item-add 3 --owner nfe --url <issue-url>`.

### Etapa 13 — Vincular mãe ↔ filhas (condicional)

**Disparo automático** quando a flag da Etapa 2 (anti-padrão #5) foi setada. Não é opcional — se o usuário escolheu "guarda-chuva + N filhas" e a mãe foi criada, a skill entra em loop criando cada filha, sem voltar ao prompt entre elas (exceto para preview).

Para cada filha:

1. Rodar Etapas 1–11 com o escopo restrito ao produto específico da filha (label `area:*` produto diferente da mãe).
2. Após criar cada filha, obter ID numérico:
   ```bash
   FILHA_ID=$(gh api /repos/<owner>/<repo>/issues/<filha-number> --jq '.id')
   ```
3. Vincular à mãe:
   ```bash
   gh api -X POST /repos/<owner>/<repo>/issues/<mae-number>/sub_issues \
     -F sub_issue_id=$FILHA_ID
   ```
4. Adicionar cada filha também ao Project (Etapa 12).

**Atenção:** `gh issue view --json id` retorna o `node_id` (string) — **não** serve. Use `gh api .../issues/<n> --jq '.id'` para o inteiro esperado pelo endpoint.

Output final: `mãe #X com N sub-issues vinculadas: #A, #B, #C`.

## Regras invioláveis

1. **Nunca pular a confirmação** do usuário em nenhuma inferência.
2. **Nunca inventar** `area:*` combinando 2 produtos — se o caso exige, propor quebrar (anti-padrão #5).
3. **Nunca aplicar** `deadline:*` sem `area:compliance`.
4. **Nunca aplicar** `sev:*` fora de `bug`/`incident`.
5. **Nunca criar** a issue sem preview aprovado explicitamente.
6. **Nunca amendar** labels obrigatórias sem avisar — se DoR não passa, ou completa ou marca como Triage.
7. **Sempre** marcar campos inferidos com `<!-- inferido, revisar -->` e gerar bloco de follow-up.
8. **Nunca criar labels no repo.** Label ausente → parar e indicar necessidade de configuração do projeto.
9. **Nunca aplicar `banda:*` como label** na criação — apenas como comentário no corpo.
10. **Nunca hardcodar listas** de valores `type:*`/`area:*`/`prio:*`/`sev:*`/`deadline:*` nesta skill — ler sempre do `labels-catalog.md` em runtime.
11. **Nunca criar issue sem capturar a Origem** (Etapa 0.3) — quem pediu, canal e ao menos um rastro (link ou fallback estruturado).
12. **Nunca deixar a seção Referências completamente vazia** — se não houver referência externa, inserir o placeholder `TODO: sem referência externa confirmada — validar antes de sair de Triage` e forçar abertura em TRIAGE.

## Exemplo de gatilho

Usuário: `cria issue: cnpj alfanumérico consulta pj compliance prazo abril`

Skill:
1. Lê `Padrao_Issues.md` + `labels-catalog.md`; verifica que não há duplicata do catálogo
2. Etapa 0: eco de entendimento
3. Etapa 0.1: confirma destino + roda pré-voo de labels no repo; se faltar alguma, para e pede configuração
4. Etapa 0.2: `gh issue list --search "cnpj alfanumérico"` para detectar duplicatas
5. Etapa 1: `type:feature`
6. Etapa 2a: família → `consulta-*`; 2b: `consulta-pj`
7. ... segue o fluxo
