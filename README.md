# Plano de Migração — Ellenco → Template Pure Pilates (todos os fluxos)

> **Modo de execução:** TODAS as alterações são feitas **manualmente na UI do n8n** (criar workflows, nodes, configurar parâmetros, conectar). Este plano **não** gera JSONs prontos pra import. Use os JSONs canônicos em `reference/pure-pilates/` (referência) e os JSONs atuais em `clients/Ellenco/original/` (origem dos prompts/queries) lado a lado.

> **Histórico:** Este plano começou cobrindo só o Fluxo 3 (agente IA WhatsApp). Foi expandido em 2026-05-05 pra cobrir os 5 fluxos do Ellenco. As Phases 0-5 são o "fluxo canônico" do template (mensageria WhatsApp). Phases 6-8 são extensões pra os outros ecossistemas (Forms, Documentos, Cleanup) usando o mesmo padrão Producer/Consumer/Rabbit.

**Goal:** Migrar todos os 5 workflows do cliente Ellenco para a arquitetura do template Pure Pilates: producer/consumer/agente separados, RabbitMQ como bus, Chatwoot (WhatsApp Cloud API oficial) substituindo Evolution API, Redis com debounce real, DLQ, e Postgres Chat Memory em vez de buffer Redis manual.

**Architecture:** Hoje os 5 workflows são monolíticos e usam Evolution API. Vão virar **9 workflows independentes** organizados em 4 ecossistemas que compartilham o broker `rabbit.ellenco.com.br` (exchange `messagesExchange` direct) com queues por tipo de evento:

| Ecossistema | Producer | Consumer(s) | Queue |
|---|---|---|---|
| **WhatsApp inbound** (Fluxo 3) | `ellenco/rabbit_post` | `ellenco/post_messages` + agente `Ellenco - Ambiente de Produção` | `post/messages` |
| **Forms inbound** (Fluxos 1+2) | `ellenco/forms_post` | `ellenco/forms_processor` + agente `Ellenco - Forms Classifier` | `forms/messages` |
| **Documentos inbound** (Fluxo 3.1) | branch do `ellenco/rabbit_post` | `ellenco/documents_consumer` | `documents/messages` |
| **Cleanup Drive** (Excluir Pasta) | `ellenco/cleanup_post` | `ellenco/cleanup_consumer` | `cleanup/folders` |
| **DLQ** (transversal) | qualquer consumer em erro | `ellenco/error_messages` | `error/messages` |

**Tech Stack:** n8n (UI manual), RabbitMQ broker dedicado `rabbit.ellenco.com.br`, Chatwoot (WhatsApp Cloud API oficial), Supabase project `vexyxaugfyoktqlscfst` (tabelas: `candidatos_pre_aprovados`, `conversations`, `documents`, `conclusão`), Redis, OpenAI gpt-4.1-mini (orquestrador + classifier), Google Gemini 2.5 Flash (análise PDF), Google Sheets (vagas), Google Drive (pastas dos candidatos).

---

## Decisões consolidadas

| # | Decisão | Valor |
|---|---|---|
| 1 | Canal de mensageria | Chatwoot (WhatsApp Cloud API oficial) |
| 2 | Banco | Supabase Ellenco (`vexyxaugfyoktqlscfst`), schema atual |
| 3 | DLQ (`error_messages`) | Incluir |
| 4 | Debounce Redis | Incluir (igual template) |
| 5 | Transcrição de áudio | Adiar (Phase 5+) |
| 6 | Intentions/roteamento | Genérico — 1 agente, sem switch |
| 7 | Webhook paths | `ellenco/oficial/messages` (producer) + `ellenco/agents/production` (agente) |
| 8 | RabbitMQ broker | `rabbit.ellenco.com.br` (dedicado, broker próprio do Ellenco) |
| 9 | Exchange + queues | `messagesExchange` (direct) + `post/messages` + `error/messages` (idêntico ao template) |
| 10 | Forms (Fluxos 1+2) | Manter no Rabbit (consistência com WhatsApp). Routing key `forms/messages`. Agente classificador separado. |
| 11 | Documentos (Fluxo 3.1) | Detectar anexo no producer principal e rotear pra `documents/messages` (em vez de `post/messages`). Consumer dedicado faz Drive + IA. |
| 12 | Cleanup Drive (Excluir Pasta) | Routear via Rabbit (`cleanup/folders`) por consistência. Trade-off: overkill pra um delete one-shot, mas mantém o padrão "tudo passa pelo broker" pedido pelo Luiz. |

---

## Visão geral — todos os fluxos depois da migração

```
                                          ┌────────────────────────────────┐
                                          │ RabbitMQ broker rabbit.ellenco │
                                          │ Exchange: messagesExchange     │
                                          │   (direct)                     │
                                          │ Queues:                        │
                                          │   • post/messages              │
                                          │   • forms/messages             │
                                          │   • documents/messages         │
                                          │   • cleanup/folders            │
                                          │   • error/messages (DLQ)       │
                                          └─┬────┬────┬────┬───────────────┘
                                            │    │    │    │
   ┌─────────────────────┐         publish  │    │    │    │
   │ Google Forms        │──webhook──┐      │    │    │    │
   └─────────────────────┘           ▼      │    │    │    │
   ┌─────────────────────┐    [P]ellenco/forms_post ───────│    │    │
   │ Chatwoot WhatsApp   │──webhook──┐                    │    │    │
   └─────────────────────┘           ▼                    │    │    │
                          [P]ellenco/rabbit_post ─────────┴────│    │
                            └ se attachment ─────────────────► │    │
                                                               │    │
   ┌─────────────────────┐                                     │    │
   │ Supabase delete row │──webhook──┐                         │    │
   └─────────────────────┘           ▼                         │    │
                          [P]ellenco/cleanup_post ─────────────│────│
                                                               │    │    │
                                                consume    consume consume consume
                                                   ▼          ▼      ▼      ▼
                                            [C]ellenco/   [C]ellenco/ [C]ellenco/ [C]ellenco/
                                            post_messages forms_      documents_  cleanup_
                                                          processor   consumer    consumer
                                                ▼              ▼          ▼          ▼
                                          [A]Ambiente    [A]Forms     Drive +    Drive
                                          Produção       Classifier   Gemini/    delete
                                                                      OpenAI     folder
                                                ▼              ▼          ▼          
                                          Chatwoot       Chatwoot    Supabase
                                          createMessage  createConv  documents
                                                         + first msg
                                                ▼              ▼          ▼          ▼
                                              em erro     em erro    em erro    em erro
                                                └──────────┴──────────┴──────────┘
                                                              │
                                                              ▼
                                                       error/messages
                                                              │
                                                              ▼
                                                  [DLQ]ellenco/error_messages
                                                  (filtro 24h, reinjeta)
```

**Legenda:** `[P]` = Producer, `[C]` = Consumer, `[A]` = Agente IA (workflow separado, chamado por webhook a partir do consumer).

**Por que essa estrutura é boa pra você:**

- **Mesmo padrão pra tudo.** Toda entrada (Forms, WhatsApp, Drive cleanup) entra primeiro num producer minimalista que normaliza e publica. Toda saída sai de um consumer que tem retry e DLQ.
- **Agentes IA viram "cérebros puros".** Recebem `{contexto}`, devolvem `{resposta estruturada}`. Não tocam em DB nem em API externa de mensageria. Isso é o que vai facilitar replicar pra próximos clientes.
- **Trocar canal só mexe nos producers.** Se um dia trocar Chatwoot por outro CRM, só os producers e os 2-3 nodes de envio dos consumers mudam. Toda a lógica de orquestração (lookup, locks, branches, agentes) segue igual.
- **Falha em qualquer ponto vira DLQ.** Hoje uma falha = mensagem perdida. Depois = 24h de janela de reprocessamento.

---

## Fluxo 3 (WhatsApp) — Diff arquitetural antes/depois

### Hoje (fluxo 3 atual)

```
[Evolution API webhook]
         ↓
   [Workflow único: 3. AGENTE IA - PRINCIPAL]
   ├─ webhook /whatsapp-ellenco
   ├─ check Redis lock (cooldown rejeição)
   ├─ check candidato pré-aprovado (Supabase)
   ├─ buffer Redis manual + wait 10s
   ├─ agente IA gpt-4.1-mini (system prompt RH)
   ├─ split mensagens
   ├─ envia via Evolution API /chat/sendMessage
   └─ salva análise em Supabase.conclusão
```

**Problemas dessa estrutura:**
- Se o n8n cai no meio do agente, mensagem é perdida (sem fila durável).
- Agente não pode ser testado/iterado isoladamente — qualquer mexida no prompt afeta produção direto.
- Buffer com `wait 10s` é frágil: não cancela quando chega mensagem nova rapidamente, e processa duplicado em race conditions.
- Sem retry: se Chatwoot/Supabase falhar, mensagem some.

### Depois (4 workflows + Chatwoot)

```
[Chatwoot inbox WhatsApp Cloud API]
         ↓ (webhook nativo)
[A. Producer ellenco/rabbit_post]
   webhook /ellenco/oficial/messages
   normaliza payload Chatwoot → shape unificado
   filtra event.type ∈ {messages, message_created} + fromMe=false
         ↓
   publica em exchange `messagesExchange` (direct)
         ↓
   ┌──────────────────────────────────┐
   │ RabbitMQ broker rabbit.ellenco.com.br
   │   queue: post/messages           │   queue: error/messages
   └──────────────────────────────────┘
         ↓                                  ↑
[B. Consumer ellenco/post_messages]         │
   rabbitmqTrigger post/messages            │
   ├─ checkCache Redis                      │
   ├─ get/create candidato (candidatos_pre_aprovados)
   ├─ get/create conversation               │
   ├─ check assigned_to (HUMAN → skip IA)   │
   ├─ HTTP POST → /ellenco/agents/production├──────┐
   ├─ recebe resposta agente                │      │
   ├─ split + delay + Chatwoot createMessage│      │
   └─ em erro → publish error/messages ─────┘      │
                                                    ↓
                                  [C. Agente Ellenco - Ambiente de Produção]
                                     webhook /ellenco/agents/production
                                     ├─ Respond to Webhook ({queued:true})
                                     ├─ pushToQueue Redis (debounce)
                                     ├─ getQueue + checkTimer + wait loop
                                     ├─ chatHistory Redis + normalizeHistory
                                     ├─ Agente OpenAI 4.1 Mini
                                     │  (system prompt RH portado do fluxo 3)
                                     ├─ Postgres Chat Memory (conversations)
                                     └─ outputFormat → callback ao consumer

[D. DLQ ellenco/error_messages]
   rabbitmqTrigger error/messages
   filter timestamp ≥ now - 24h
   reinjeta via webhook → producer
```

### O que muda e por quê (alto nível)

| Mudança | O que faz | Por quê |
|---|---|---|
| Separar producer / consumer / agente | 1 workflow → 4 workflows | Permite testar/evoluir cada peça isoladamente. Agente fica num "Ambiente de Produção" estilo template, podendo ter um "Ambiente de Testes" gêmeo depois. |
| Adicionar RabbitMQ no caminho WhatsApp | mensagem entra no broker antes de ser processada | Durabilidade: se o consumer cai, mensagem fica enfileirada. Permite retry e DLQ. |
| Trocar Evolution API → Chatwoot | mudar fonte do webhook + node de envio | Chatwoot é o canal canônico do template e já está com WhatsApp Cloud API oficial conectada. Ganha inbox unificada e SLA da Meta. |
| Adicionar DLQ (`error/messages`) | workflow que captura erros e reinjeta | Hoje erro = mensagem perdida. Com DLQ + filtro 24h, mensagens transientes são reprocessadas; mensagens muito antigas são descartadas. |
| Trocar buffer manual por debounce Redis (template) | `wait 10s` simples → `pushToQueue` + `checkTimer` + `wait` em loop | Junta mensagens consecutivas corretamente, evita race conditions, evita processar mensagem já processada (`mid` check). Mais barato em chamadas OpenAI. |
| Manter Supabase | sem trocar de DB | Supabase já é Postgres, schema atual já tem `id_chatwoot` e `conversations` compatível. Trocar seria custo sem ganho. |
| Sem intentions | 1 agente direto, sem switch de intenção | Domínio Ellenco é nichado (RH/recrutamento), não precisa de roteamento entre 6 agentes especialistas como Pilates. |
| Manter Redis lock de cooldown rejeição | comportamento específico Ellenco fica no consumer | Bloquear candidato reprovado por 5min é regra de negócio do RH, não do template. Vive no consumer onde a decisão é tomada. |

---

## Phase 0: Pré-requisitos compartilhados

> Estes setups precisam estar prontos antes de começar qualquer workflow novo. Também servem pros fluxos 1/2 quando forem migrados depois.

### Task 0.1: Validar credencial RabbitMQ no n8n

**Contexto:** o n8n do Ellenco é dedicado a esse cliente, então a credencial existente (`gHxkd5JMUqm79RiZ` "Credencial - RabbitMQ") já é inequívoca pelo contexto — não precisa renomear. Só precisa confirmar que ela aponta pro broker certo.

- [ ] **Step 1: Abrir a credencial existente**

No n8n: **Settings → Credentials → "Credencial - RabbitMQ"** (a que o fluxo 1 atual usa).

- [ ] **Step 2: Verificar que aponta pra `rabbit.ellenco.com.br`**

Confirme:
- **Hostname:** `rabbit.ellenco.com.br`
- **Port:** 5671 (TLS) ou 5672 (não-TLS)
- **User:** `Ellenco`
- **Vhost:** `/` (default)
- **SSL/TLS:** habilitado se a porta for 5671

- [ ] **Step 3: Testar a conexão**

Clique em **Test connection**. Esperado: badge verde "Connection successful".

> Nas próximas tasks, onde o plano fala "credencial `RabbitMQ - Ellenco`", use essa credencial existente (qualquer que seja o nome dela).

### Task 0.2: Criar exchange e queues no broker

**O que muda:** hoje o broker do Ellenco só tem a queue `filas-forms-ellenco` (usada pelo fluxo 1 atual). Precisamos criar a infraestrutura `messagesExchange` + 5 queues novas (mesmos nomes do template + extensões dos outros ecossistemas).

> **Nota:** a queue legada `filas-forms-ellenco` pode ser **deletada depois** que o Fluxo 1 novo (`ellenco/forms_post`, Phase 6) estiver virado. Não delete agora — fica de rede de proteção pro rollback.

- [ ] **Step 1: Acessar a UI de admin do RabbitMQ**

Abra `https://rabbit.ellenco.com.br` no browser, login com user `Ellenco`.

- [ ] **Step 2: Criar exchange `messagesExchange`**

Em **Exchanges → Add a new exchange**:
- **Name:** `messagesExchange`
- **Type:** `direct`
- **Durability:** `Durable`
- **Auto delete:** `No`
- **Internal:** `No`

Clique **Add exchange**.

- [ ] **Step 3: Criar as 5 queues**

Em **Queues → Add a new queue**, repita pra cada uma das 5 queues abaixo:

| Queue | Tipo | Usada em |
|---|---|---|
| `post/messages` | Quorum, durable | Phase 1-2 (WhatsApp consumer) |
| `error/messages` | Quorum, durable | Phase 4 (DLQ) |
| `forms/messages` | Quorum, durable | Phase 6 (Forms consumer) |
| `documents/messages` | Quorum, durable | Phase 7 (Documents consumer) |
| `cleanup/folders` | Quorum, durable | Phase 8 (Drive cleanup consumer) |

Pra cada uma:
- **Name:** (do quadro acima)
- **Type:** `Quorum` (consistente com a queue legada `filas-forms-ellenco`, mais resiliente que classic)
- **Durability:** `Durable`
- **Auto delete:** `No`
- **Arguments:** vazio (defaults)

Clique **Add queue** cada vez.

- [ ] **Step 4: Bindar todas as queues ao exchange**

Em **Exchanges → messagesExchange → Bindings → Add binding from this exchange**, repita pra cada queue:

| To queue | Routing key |
|---|---|
| `post/messages` | `post/messages` |
| `error/messages` | `error/messages` |
| `forms/messages` | `forms/messages` |
| `documents/messages` | `documents/messages` |
| `cleanup/folders` | `cleanup/folders` |

> **Por que routing key igual ao nome da queue?** Por simplicidade. Como o exchange é `direct`, cada producer escolhe explicitamente onde publicar via routing key. Manter routing-key = nome-da-queue evita confusão. (No template Pure só existem 2 queues e essa convenção também é seguida.)

- [ ] **Step 5: Validar**

Em **Queues**, devem aparecer as 5 queues novas (+ `filas-forms-ellenco` legada) com 0 mensagens. Em **Exchanges → messagesExchange → Bindings**, devem aparecer 5 bindings.

### Task 0.3: Configurar inbox Chatwoot WhatsApp Cloud API

**O que muda:** o Chatwoot precisa estar configurado pra receber mensagens da WhatsApp Business API e disparar webhook pro nosso producer.

> **Nota:** se o inbox WhatsApp já estiver configurado e funcional no Chatwoot do Ellenco (isso é o que você quis dizer com "está com a api oficial conectada"), pule pros Steps 4-6 (que cuidam só do webhook saída).

- [ ] **Step 1: Abrir Chatwoot → Inboxes**

Login no Chatwoot do Ellenco como admin. Vá em **Settings → Inboxes**.

- [ ] **Step 2: Verificar que existe inbox WhatsApp**

Deve existir um inbox tipo "WhatsApp" (com ícone do WhatsApp). Anote o **Inbox ID** (visível na URL ao abrir o inbox: `/app/accounts/{account_id}/settings/inboxes/{inbox_id}`).

- [ ] **Step 3: Anotar Account ID**

Anote o **Account ID** (mesmo lugar, primeiro segmento da URL).

- [ ] **Step 4: Configurar webhook de saída (Chatwoot → n8n)**

Em **Settings → Integrations → Webhooks → Add new webhook**:
- **Webhook URL:** `https://<seu-host-n8n>/webhook/ellenco/oficial/messages`
  - Substitua `<seu-host-n8n>` pelo host do n8n do Ellenco (ex.: `n8n-n8n.hgabnb.easypanel.host`)
- **Subscriptions:** marque pelo menos:
  - `Conversation Created`
  - `Message Created`
  - `Message Updated`

Clique **Create**.

- [ ] **Step 5: Anotar URL do Chatwoot**

Anote a URL base do Chatwoot (ex.: `https://chat.ellenco.com.br`). Vai ser usada nos nodes de envio de mensagem.

### Task 0.4: Criar credencial Chatwoot no n8n

**O que muda:** o consumer vai precisar enviar mensagens via Chatwoot API. No fluxo 3 atual o envio era via Evolution API.

- [ ] **Step 1: Gerar API access token no Chatwoot**

No Chatwoot: clique no avatar (canto superior direito) → **Profile Settings → Access Token → Copy Token**.

- [ ] **Step 2: Criar credencial Header Auth no n8n**

No n8n: **Settings → Credentials → Add credential → Header Auth**.
- **Name:** `Chatwoot - Api Access Token`
- **Header Name:** `api_access_token`
- **Header Value:** o token copiado no Step 1

Salve.

- [ ] **Step 3: Validar**

Faça um teste manual com `curl` (ou um node HTTP Request temporário no n8n):
```
curl -X GET https://chatwoot.ellenco.com.br/api/v1/accounts/1/conversations \
  -H "api_access_token: <seu-token>"
```
Esperado: JSON com lista de conversas (`{ data: { meta: ..., payload: [...] } }`) — pode estar vazia. Se vier 401, é problema de token.

### Task 0.5: Validar / ajustar schema Supabase

**O que muda:** o schema atual do Supabase Ellenco já está bem alinhado com o template. Esta task confirma o estado e aplica ajustes mínimos só onde necessário.

**Estado atual** (verificado via MCP em 2026-05-05):
- `candidatos_pre_aprovados` (954 rows) — equivalente a `clients` do template, tem `id_chatwoot bigint nullable` ✅
- `conversations` (14652 rows) — colunas `(session_id bigint, message jsonb, id int)` — **bate exatamente** com o que o template espera pro Postgres Chat Memory ✅
- `documents` (954 rows) — específico Ellenco, mantido
- `conclusão` (72 rows) — específico Ellenco, mantido

- [ ] **Step 1: Verificar se `candidatos_pre_aprovados.id_chatwoot` está sendo populado**

No Supabase MCP / SQL Editor, rodar:
```sql
SELECT count(*) FROM candidatos_pre_aprovados WHERE id_chatwoot IS NOT NULL;
```

Se for 0 ou muito baixo: vamos **popular** durante a fase de cutover (consumer cuida disso ao criar candidato no Supabase a partir da conversation Chatwoot).

- [ ] **Step 2: Verificar se há colunas extras necessárias**

Rodar:
```sql
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'candidatos_pre_aprovados';
```

**Decisão:** o schema atual é suficiente. **Não vamos adicionar `last_activity_at`, `follow_status`, `assigned_to` agora** — o template Pure Pilates usa essas colunas no `conversations`, mas o Ellenco não tem o conceito de "follow status" no DB hoje (usa Redis lock + tabela `conclusão`). Para fidelidade ao template **sem quebrar o schema atual**, vamos manter o `id_chatwoot` como vínculo e usar a tabela `conclusão` pro estado terminal (aprovado/reprovado). Se mais pra frente fizer falta, adicionamos.

- [ ] **Step 3: Confirmar credencial Postgres no n8n**

Localize a credencial Supabase do Ellenco no n8n (provavelmente nome `Supabase - Ellenco` ou similar; ID `ancWAnnjifKK4yEJ` segundo o fluxo 3 atual). Confirme que aponta pro projeto `vexyxaugfyoktqlscfst`.

Renomeie pra `Postgres - Ellenco` se ainda não estiver com nome consistente (o template usa `Postgres - Pure Pilates`).

---

## Phase 1: Workflow Producer (`ellenco/rabbit_post`)

### Diff vs fluxo 3 atual

| Aspecto | Fluxo 3 atual | Producer novo | Por quê |
|---|---|---|---|
| Webhook source | Evolution API (payload com `body.data.key.remoteJid`, `messageType`, etc) | Chatwoot (payload com `event`, `account.id`, `conversation`, etc) | Migração de canal decidida |
| Webhook path | `whatsapp-ellenco` | `ellenco/oficial/messages` | Convenção do template |
| Saída | processa inline o payload | publica em `messagesExchange` (direct) e termina | Separação produtor/consumidor |
| Filtros | check candidato + Redis lock | só filtra `event.type` + `fromMe=false` | Producer é "burro" — só normaliza e enfileira |
| Hub challenge (verificação Meta) | não existia | existe, igual ao template | Caso o WhatsApp Cloud API queira reverificar o webhook diretamente algum dia, fica pronto |

### Task 1.1: Criar workflow vazio

- [ ] **Step 1: Criar workflow novo**

No n8n: **Workflows → Add workflow**.
- **Name:** `ellenco/rabbit_post`
- **Tags:** adicionar tag `ellenco` (criar se não existir)

Salve.

### Task 1.2: Criar webhook GET (hub.challenge)

**Por quê:** Meta/WhatsApp Cloud API faz uma verificação inicial no webhook por GET. O template tem isso. Mesmo que o Chatwoot já tenha verificado, deixar pronto pro caso de futuramente apontar a Meta direto.

- [ ] **Step 1: Adicionar node Webhook**

Pesquisar `Webhook` na barra. Adicionar.
- **Authentication:** None
- **HTTP Method:** GET
- **Path:** `ellenco/oficial/messages`
- **Respond:** `Using Respond to Webhook node`

Renomear o node para `registerWebhook`.

- [ ] **Step 2: Adicionar node Set para hub data**

Adicionar **Edit Fields (Set)** após o webhook.
- **Mode:** Manual mapping
- Adicionar 3 campos String:
  - `hub.mode` = `={{ $json.query['hub.mode'] }}`
  - `hub.challenge` = `={{ $json.query['hub.challenge'] }}`
  - `hub.verify_token` = `={{ $json.query['hub.verify_token'] }}`

Renomear o node para `setHubData`.

- [ ] **Step 3: Adicionar Respond to Webhook**

- **Respond With:** Text
- **Response Body:** `={{ $json.hub.challenge }}`

Renomear para `respondHub`.

Conectar: `registerWebhook → setHubData → respondHub`.

### Task 1.3: Criar webhook POST principal

- [ ] **Step 1: Adicionar segundo node Webhook**

- **Authentication:** None
- **HTTP Method:** POST
- **Path:** `ellenco/oficial/messages`
- **Respond:** Immediately (default)

Renomear pra `post/messages`.

> **Atenção:** o n8n permite 2 webhooks com mesmo path se forem métodos HTTP diferentes (GET + POST). Use o mesmo `webhookId` (n8n gera) — eles convivem.

### Task 1.4: Criar nó `initData2` (normalização Chatwoot)

**Por quê:** o template tem 2 nodes de normalização — `initData2` (payload Chatwoot) e `initData` (payload Meta direto). Como vamos usar Chatwoot como única fonte, precisamos só do `initData2`.

> **Referência:** ver `reference/pure-pilates/rabbit_post.json` linhas 152-282 (node `initData2`). Vamos copiar exatamente os mesmos campos. Mantenha os nomes idênticos pra que o consumer (que reusa estrutura do template) ache o que espera.

- [ ] **Step 1: Adicionar node Edit Fields (Set)**

Após `post/messages` (webhook POST). Renomear pra `initData2`.
- **Mode:** Manual mapping
- **Include other input fields:** No

- [ ] **Step 2: Adicionar os 18 campos seguintes**

Copiar os 18 campos exatamente como em `reference/pure-pilates/rabbit_post.json` node `initData2`. Ordem e tipo:

| Campo | Tipo | Valor (expressão) |
|---|---|---|
| `lead.name` | String | `={{ $json.body.conversation?.meta?.sender?.name }}` |
| `lead.phone` | String | `=+{{ $json.body.conversation.meta?.sender?.phone_number.replace(/\D/g, '') \|\| $json.body.conversation.messages?.first().conversation.contact_inbox.source_id.replace(/\D/g, '') \|\| null }}` |
| `event.type` | String | `={{ $json.body.event \|\| null }}` |
| `message.fromMe` | Boolean | `={{ $json.body.message_type === 'outgoing' }}` |
| `message.id` | String | `={{ $json.body.source_id \|\| $json.body.conversation.messages[0].id }}` |
| `message.timestamp` | String | `={{ $json.body.conversation.timestamp \|\| $now.toSeconds() }}` |
| `message.type` | String | `={{ $json.body?.attachments?.first()?.file_type \|\| 'text' }}` |
| `message.content` | Object | `={{ $json.body.attachments?.first() \|\| {"body": $json.body.content} \|\| null }}` |
| `source.app` | String | `={{ $json.body?.entry?.first()?.changes[0]?.value?.messaging_product \|\| 'whatsapp' }}` |
| `source.originType` | String | `={{ $json.body.entry?.first()?.changes[0]?.value?.messages[0]?.referral?.source_type \|\| 'organic' }}` |
| `source.metadata` | Object | (ver JSON canônico — expressão grande pra capturar referral de ad) |
| `response.totalQueueTime` | Number | `15` |
| `response.checkQueue` | Number | `5` |
| `response.messagesDelay` | Number | `3` |
| `chatwoot.account_id` | Number | `={{ $json.body.account?.id }}` |
| `chatwoot.contact_id` | Number | `={{ $json.body.conversation.contact_inbox.contact_id }}` |
| `chatwoot.inbox_id` | Number | `={{ $json.body.conversation?.contact_inbox?.inbox_id }}` |
| `chatwoot.conversation_id` | Number | `={{ $json.body.conversation?.id }}` |
| `chatwoot.source_id` | String | `={{ $json.body.conversation.contact_inbox.source_id }}` |

> **Truque pra ser preciso:** no n8n, abra o JSON canônico em paralelo (split screen) e copie/cole cada expressão. Não tente reescrever de memória.

### Task 1.5: Criar filtro `fromMe`

- [ ] **Step 1: Adicionar node Filter**

Após `initData2`. Renomear pra `fromMe`.
- **Conditions:**
  - Boolean: `={{ $json.message.fromMe }}` is `false`

Tudo que `fromMe=true` (ou seja, mensagens enviadas pelo próprio bot/agente humano) é descartado. Não queremos processar isso.

### Task 1.6: Criar filtro `eventType`

- [ ] **Step 1: Adicionar node Filter**

Após `fromMe`. Renomear pra `eventType`.
- **Combinator:** OR
- **Condition 1:** String `={{ $json.event.type }}` equals `messages`
- **Condition 2:** String `={{ $json.event.type }}` equals `message_created`

### Task 1.7: Criar nó `receiveMessages` (publish RabbitMQ)

> **Atenção:** o nome do node é confuso — ele PUBLICA, não recebe. Mantenha o nome `receiveMessages` por consistência com o template (o consumer espera achar o mesmo padrão).

- [ ] **Step 1: Adicionar node RabbitMQ**

Após `eventType`. Renomear pra `receiveMessages`.
- **Credential:** `RabbitMQ - Ellenco` (Task 0.1)
- **Mode:** Send Message to Exchange
- **Exchange:** `messagesExchange`
- **Exchange Type:** `direct`
- **Routing Key:** `post/messages`
- **Message:** deixar default (envia o JSON do item — o consumer vai parsear)
- **Options:** vazio

### Task 1.8: Conectar e ativar

- [ ] **Step 1: Verificar conexões**

Layout final esperado:
```
registerWebhook (GET) → setHubData → respondHub
post/messages (POST) → initData2 → fromMe → eventType → receiveMessages
```

- [ ] **Step 2: Ativar workflow**

Toggle **Active** no canto superior direito.

### Task 1.9: Validar com payload de teste

- [ ] **Step 1: Disparar mensagem real no Chatwoot**

Manda uma mensagem de teste via WhatsApp pra um número que esteja conectado ao inbox.

- [ ] **Step 2: Verificar execução no n8n**

Em **Executions → ellenco/rabbit_post**, deve aparecer 1 execução verde. Abrir e verificar:
- `initData2` populou `lead.phone`, `chatwoot.conversation_id`, `message.content.body`.
- `eventType` passou (`event.type === 'message_created'`).
- `receiveMessages` executou sem erro.

- [ ] **Step 3: Verificar mensagem na queue**

No admin RabbitMQ, **Queues → post/messages**, deve ter **1 mensagem ready**. (Como o consumer ainda não existe, ela fica aí.)

- [ ] **Step 4: Inspecionar payload**

Em **Get Messages** na queue, peça pra mostrar 1 mensagem (Ack mode: Nack — pra não consumir). Confirme que o JSON tem `lead`, `chatwoot`, `message`, `event`, `response` no shape esperado.

---

## Phase 2: Workflow Consumer (`ellenco/post_messages`)

### Diff vs fluxo 3 atual

Esse é o workflow que recebe **mais lógica vinda do fluxo 3 atual**. Aqui ficam todos os checks de orquestração e o envio da resposta. **O agente em si sai daqui** e vira workflow separado (Phase 3).

| O que vem do fluxo 3 atual | Para onde vai | Diff |
|---|---|---|
| Check Redis lock cooldown rejeição | Mantém no consumer | Sem mudança lógica, só renomeia chave Redis pra padrão template |
| Lookup candidato no Supabase | Mantém no consumer | Query: `SELECT * FROM candidatos_pre_aprovados WHERE phone = $1 OR id_chatwoot = $2` |
| Decisão "passar pro agente ou não" | Mantém no consumer | Adiciona check de `assignee_id` do Chatwoot — se conversation já tem agente humano atribuído, skip IA |
| Envio Evolution API | **Substitui por Chatwoot createMessage** | Endpoint diferente, payload diferente, autenticação diferente |
| Bloco do agente IA (modelo, prompt, tools) | **Sai do consumer**, vai pro Workflow C (Phase 3) | Consumer chama o agente via HTTP webhook |
| Buffer Redis manual + wait 10s | **Sai do consumer**, vai pro Workflow C com debounce real | Consumer não mexe mais com fila Redis de mensagens |
| Salvar análise em `conclusão` | Mantém no consumer | Específico Ellenco, fica na orquestração |

**O que é novo (não existia no fluxo 3 atual):**
- `rabbitmqTrigger` na queue `post/messages` (entrada agora vem da fila, não do webhook direto)
- Publish em `error/messages` em caso de erro (ainda vamos ver na Phase 4)
- Hidratação de variáveis de ambiente via webhook (igual o `getEnvVariables` do template, opcional — pode ser SET node simples)

### Task 2.1: Criar workflow vazio

- [ ] **Step 1: Criar workflow**

Nome: `ellenco/post_messages`. Tag `ellenco`.

### Task 2.2: Adicionar `rabbitmqTrigger`

- [ ] **Step 1: Adicionar node RabbitMQ Trigger**

- **Credential:** `RabbitMQ - Ellenco`
- **Queue:** `post/messages`
- **Options → Parallel Messages:** 5 (igual ao template)
- **Options → Acknowledge mode:** `Execution Finishes` (consome só após execução OK; se der erro, mensagem volta pra queue — isso vamos refinar com DLQ na Phase 4)

Renomear para `post/messages`.

### Task 2.3: Adicionar `initData` (parse JSON da fila)

- [ ] **Step 1: Adicionar node Edit Fields (Set)**

Após `post/messages`.
- **Mode:** JSON
- **JSON Output:** `={{ $json.content.parseJson() }}`
- **Include other input fields:** No

Renomear pra `initData`.

> O conteúdo da mensagem RabbitMQ vem como string em `$json.content`. O parse devolve o objeto que o producer publicou.

### Task 2.4: Adicionar `setVariables` (env config)

**Por quê:** centralizar URLs e IDs em um único lugar pra facilitar manutenção. O template usa `getEnvVariables` HTTP, mas pra Ellenco vamos simplificar com SET.

- [ ] **Step 1: Adicionar node Edit Fields (Set)**

- **Mode:** Manual mapping

| Campo | Tipo | Valor |
|---|---|---|
| `chatwootUrl` | String | `https://chatwoot.ellenco.com.br` |
| `chatwootAccountId` | Number | `1` |
| `chatwootInboxId` | Number | `1` |
| `agentUrl` | String | `https://n8n.ellenco.com.br/webhook/ellenco/agents/production` |
| `errorWebhookUrl` | String | `https://n8n.ellenco.com.br/webhook/ellenco/oficial/rabbit` (vamos criar essa rota interna pra DLQ na Phase 4) |

Renomear pra `setVariables`.

### Task 2.5: Lookup do candidato (`tryGetClient`)

**Por quê:** precisa achar o candidato no Supabase pra saber se está pré-aprovado, em cooldown, ou se é primeiro contato.

> **Contexto descoberto via Supabase em 2026-05-05:**
> - **Phone na tabela está em formato JID** (`5515996823928@s.whatsapp.net`), não E.164. O Chatwoot vai mandar `+5515996823928` no `lead.phone`. Precisa normalizar (extrair só dígitos) pra dar match.
> - **`id_chatwoot` aparenta ser `conversation_id`** (sequencial, cresce com candidatos novos). Mas existem **duplicações** (ex.: id 1027 e 1025 ambos com id_chatwoot=3197) — provável bug do fluxo 2 atual sobrescrevendo. Não bloqueia migração mas vale flagar pro time depois.
> - **Cada candidato hoje tem APENAS 1 row** (954 phones únicos em 954 rows). O sistema atual não suporta múltiplas candidaturas por phone — sobrescreve. Vamos manter esse comportamento.

- [ ] **Step 1: Adicionar node Postgres**

- **Credential:** `Postgres - Ellenco` (Task 0.5)
- **Operation:** Execute Query
- **Query:**
  ```sql
  SELECT *,
    (id_chatwoot = $1) AS matched_by_chatwoot
  FROM candidatos_pre_aprovados
  WHERE id_chatwoot = $1
     OR REGEXP_REPLACE(phone, '\D', '', 'g') = $2
  ORDER BY matched_by_chatwoot DESC NULLS LAST, id DESC
  LIMIT 1;
  ```
- **Query Parameters:**
  - `={{ $json.chatwoot.conversation_id }}`
  - `={{ $json.lead.phone.replace(/\D/g, '') }}` — extrai só dígitos do phone E.164 do Chatwoot

> **Como o query funciona:**
> - Tenta achar por `id_chatwoot = conversation_id` (match preciso da conversation atual)
> - OU por phone (normalizado: regex tira `+`, espaços, `@s.whatsapp.net` da coluna; o JS tira `+` do parâmetro)
> - `ORDER BY matched_by_chatwoot DESC` garante que match por conversation_id vem antes de match por phone
> - `LIMIT 1` é seguro porque cada phone aparece em 1 row só

Renomear pra `tryGetClient`.

### Task 2.5b: Re-linkar candidato existente à conversation atual

**Por quê:** se achamos o candidato pelo **phone** mas o `id_chatwoot` da row é diferente (ou null) da `chatwoot.conversation_id` atual, significa que o candidato voltou a falar e o Chatwoot abriu uma conversation nova. Precisamos atualizar o `id_chatwoot` da row pra apontar pra conversation atual — assim a memória de chat (`conversations`) usa a session_id certa e mensagens futuras encontram a mesma row.

- [ ] **Step 1: Adicionar node IF**

Após `tryGetClient`. Renomear pra `needsRelink`.
- **Condition:**
  - `={{ $json.id }}` is not empty (achou candidato)
  - AND `={{ $json.id_chatwoot }}` not equals `={{ $('initData').item.json.chatwoot.conversation_id }}`

**True:** precisa atualizar.
**False:** segue (id_chatwoot já está correto).

- [ ] **Step 2: Adicionar node Postgres (Update)**

- **Operation:** Update
- **Schema:** public
- **Table:** candidatos_pre_aprovados
- **Match Columns:** id
- **Values:**
  - `id` = `={{ $json.id }}`
  - `id_chatwoot` = `={{ $('initData').item.json.chatwoot.conversation_id }}`

Renomear pra `relinkChatwoot`.

### Task 2.6: Branch — candidato existe?

- [ ] **Step 1: Adicionar node IF**

Após `tryGetClient`. Renomear pra `hasClient`.
- **Condition:** `={{ $json.id }}` is not empty

**True:** candidato existe → continua fluxo.
**False:** primeiro contato → cria candidato (próxima task).

### Task 2.7: Criar candidato novo (`createClient`) — branch False

> **Decisão Ellenco:** o fluxo 3 atual **não cria candidato** se ele não existe — assume que já foi criado pelo fluxo 2 (Processa Forms). Pra alinhar com o template (que cria), vamos **criar registro mínimo** se for primeiro contato direto via WhatsApp (caso raro mas possível). Isso pode ser revisto.

- [ ] **Step 1: Adicionar node Edit Fields**

Renomear pra `newClientFields`. Campos:

| Campo | Valor |
|---|---|
| `name` | `={{ $json.lead.name }}` |
| `phone` | `={{ $json.lead.phone }}` |
| `id_chatwoot` | `={{ $json.chatwoot.conversation_id }}` |

- [ ] **Step 2: Adicionar node Postgres**

- **Operation:** Insert
- **Schema:** public
- **Table:** candidatos_pre_aprovados
- **Columns:** name, phone, id_chatwoot
- **Mode:** Map each column manually
- **Values:** referenciar `newClientFields`

Renomear pra `createClient`.

- [ ] **Step 3: Merge das branches**

Adicionar node **Merge** (mode: `Append`) após `hasClient` (recebendo True direto e False pós-`createClient`). Renomear pra `joinClient`.

### Task 2.8: Check de cooldown rejeição (Redis lock)

**Por quê:** Ellenco-específico. Se candidato foi reprovado no fluxo 2 (Processa Forms), o Redis tem uma chave bloqueando interações por 5min. Mantemos esse check.

- [ ] **Step 1: Adicionar node Redis**

- **Credential:** Redis - VPS Extra (mesma do template) ou `Redis - Ellenco` se for cred separada — confirmar
- **Operation:** Get
- **Key:** `={{ $json.lead.phone }}` (ou o padrão atual do Ellenco — verificar no fluxo 3 original como a chave é montada)

Renomear pra `checkCooldownLock`.

- [ ] **Step 2: Adicionar node IF**

Renomear pra `isLocked`.
- **Condition:** `={{ $json.value }}` is not empty (ou seja, a chave existe → cooldown ativo)

**True:** sai do fluxo (não responde nada — candidato em cooldown).
**False:** continua.

### Task 2.9: Check de assignee humano (Chatwoot)

**Por quê:** se o candidato foi atribuído a um agente humano no Chatwoot, IA não responde.

- [ ] **Step 1: Adicionar node HTTP Request**

- **Method:** GET
- **URL:** `={{ $json.chatwootUrl }}/api/v1/accounts/{{ $json.chatwoot.account_id }}/conversations/{{ $json.chatwoot.conversation_id }}`
- **Authentication:** Predefined Credential Type
- **Credential Type:** Header Auth → `Chatwoot - Api Access Token`

Renomear pra `getConversation`.

- [ ] **Step 2: Adicionar node IF**

Renomear pra `hasHumanAssignee`.
- **Condition:** `={{ $json.meta.assignee }}` is not empty AND `={{ $json.meta.assignee.role }}` equals `agent`

**True:** humano cuida → sai do fluxo.
**False:** continua pra IA.

### Task 2.10: Chamar o agente (HTTP webhook)

- [ ] **Step 1: Adicionar node HTTP Request**

- **Method:** POST
- **URL:** `={{ $json.agentUrl }}` (do `setVariables`)
- **Send Body:** Yes
- **Body Content Type:** JSON
- **Specify Body:** Using JSON
- **JSON Body:** o item inteiro do consumer (que tem `lead`, `chatwoot`, `message`, `response`, e o registro do candidato)

```
={{ $json.toJsonString() }}
```
- **Options → Timeout:** 60000ms (agente com debounce pode demorar)

Renomear pra `callAgent`.

> **Comportamento esperado:** o agente vai responder rapidamente com `{queued: true}` (Respond to Webhook do agente faz isso) e então **callback** vai chegar via outro caminho — ou (versão simplificada) bloqueamos esperando a resposta. Vamos pela versão simplificada: agente bloqueia até terminar, devolve `{messages: [...]}`. Decisão de simplificação registrada — pode ser revisitada se a latência incomodar.

### Task 2.11: Split de mensagens + delay

**Por quê:** o agente devolve um array de mensagens (split por `\n\n`), e queremos enviar uma de cada vez com delay pra parecer humano.

- [ ] **Step 1: Adicionar node Split Out**

- **Field to Split Out:** `messages` (ou `mensagem` — confirmar com o output real do agente)

Renomear pra `splitMessages`.

- [ ] **Step 2: Adicionar node Loop Over Items (`SplitInBatches`)**

- **Batch Size:** 1

Renomear pra `loopMessages`.

- [ ] **Step 3: Adicionar node Wait**

- **Resume:** After Time Interval
- **Wait Amount:** `={{ $json.response.messagesDelay }}` (vem do `setVariables`/`initData`, default 3 segundos)

Renomear pra `messageDelay`.

### Task 2.12: Enviar mensagem pro Chatwoot (`createMessage`)

- [ ] **Step 1: Adicionar node Edit Fields (`setMessageData`)**

Campos:

| Campo | Valor |
|---|---|
| `content` | `={{ $json.message }}` |
| `message_type` | `outgoing` |
| `content_type` | `text` |

- [ ] **Step 2: Adicionar node HTTP Request (`createMessage`)**

- **Method:** POST
- **URL:** `={{ $json.chatwootUrl }}/api/v1/accounts/{{ $json.chatwoot.account_id }}/conversations/{{ $json.chatwoot.conversation_id }}/messages`
- **Authentication:** Header Auth → `Chatwoot - Api Access Token`
- **Send Body:** Yes
- **Body Content Type:** JSON
- **JSON Body:** `={{ { content: $json.content, message_type: $json.message_type, content_type: $json.content_type } }}`

### Task 2.13: Loop back

- [ ] **Step 1: Conectar saída do `createMessage` de volta pra `loopMessages`**

Garante que o loop processa todas as mensagens com delay entre elas.

### Task 2.14: Error handling — publicar em `error/messages`

**Por quê:** se qualquer node falhar (Postgres timeout, Chatwoot 500, agente 504), queremos enfileirar em `error/messages` pra DLQ tentar de novo.

- [ ] **Step 1: Configurar Error Workflow**

No workflow `ellenco/post_messages`: **Settings → Error Workflow → ellenco/error_publisher** (vamos criar na Phase 4 — por enquanto deixa em branco, completa depois).

Alternativa: usar node **Error Trigger** + RabbitMQ publish dentro do mesmo workflow.

> **Decisão de simplicidade:** o template usa um webhook intermediário `purePilates/oficial/rabbit` que republica em `error/messages`. Pra Ellenco vamos fazer **igual** — então a publicação em error/messages acontece via HTTP no Workflow Producer (Phase 1) que ganha mais um path interno na Phase 4. Mantém fidelidade.

### Task 2.15: Conectar tudo e ativar

- [ ] **Step 1: Layout final**

```
post/messages → initData → setVariables → tryGetClient → hasClient
  False → newClientFields → createClient ─┐
  True ─→ needsRelink ┬─ True  → relinkChatwoot ─┐
                      └─ False ──────────────────┴─ joinClient → checkCooldownLock → isLocked
    True → STOP
    False → getConversation → hasHumanAssignee
              True → STOP
              False → callAgent → splitMessages → loopMessages → messageDelay → setMessageData → createMessage ──┐
                                                                                                                   │
                                                                                                                   ▼
                                                                                                              loopMessages (loop)
```

- [ ] **Step 2: Deixar workflow INATIVO ainda**

Não ative. Vamos ativar tudo junto na Phase 5 (cutover).

### Task 2.16: Validação isolada

- [ ] **Step 1: Pin de mensagem na queue**

Use o admin RabbitMQ pra publicar manualmente uma mensagem na `post/messages` com payload de teste:

```json
{
  "lead": {"name": "Teste", "phone": "+5511999999999"},
  "chatwoot": {"account_id": 1, "conversation_id": 999, "contact_id": 1, "inbox_id": 1, "source_id": "999"},
  "message": {"id": "test-1", "content": {"body": "olá"}, "type": "text", "fromMe": false, "timestamp": "1730000000"},
  "event": {"type": "message_created"},
  "response": {"totalQueueTime": 15, "checkQueue": 5, "messagesDelay": 3}
}
```

- [ ] **Step 2: Executar workflow uma vez no n8n manualmente**

Em `ellenco/post_messages`, clique **Execute workflow** (com o trigger ouvindo). A mensagem será consumida.

- [ ] **Step 3: Inspecionar nodes**

Verifique:
- `tryGetClient` retornou registro (ou vazio se candidato não existe).
- Branch certa foi seguida (cooldown, humano, IA).
- `callAgent` deu erro 404 esperado (o agente ainda não existe — vamos criar na Phase 3).

> Esse erro é OK. Ele confirma que tudo até o agente está funcionando. Phase 3 corrige.

---

## Phase 3: Workflow Agente (`Ellenco - Ambiente de Produção`)

### Diff vs fluxo 3 atual

| O que vem do fluxo 3 atual | Para onde | Diff |
|---|---|---|
| Webhook trigger | Continua trigger inicial | Path muda: `whatsapp-ellenco` → `ellenco/agents/production` |
| Buffer Redis manual + wait 10s | **Substituído por debounce template** | `pushToQueue` + `getQueue` + `checkTimer` switch + `wait` em loop |
| Bloco agente OpenAI 4.1 Mini | Continua | Mesmo modelo, mesmo system prompt, mesmas tools |
| Memory store Redis (chave `client:{id}:conversations:{cid}`) | Migra pra **Postgres `conversations`** | Template usa Postgres como fonte de verdade pra histórico; Redis vira cache opcional |
| Output parser estruturado (`resultado`, `motivo`) | Mantém | Específico Ellenco (decisão final aprovado/reprovado) |
| Saída pro Evolution API | **REMOVE** | Resposta vai pro consumer via webhook, consumer cuida do envio Chatwoot |
| Lookup candidato | **REMOVE** | Já feito pelo consumer; agente recebe candidato no payload |
| Salvar `conclusão` no Supabase | **REMOVE** (volta pro consumer) | Consumer faz a escrita após receber resposta do agente |
| Lock Redis cooldown | **REMOVE** | Decisão de cooldown é orquestração, fica no consumer |

**Resumo da filosofia:** o agente vira um **"cérebro puro"**. Recebe `{lead, chatwoot, message, candidato, response_config}`, faz pensamento (com debounce + memory), devolve `{messages: [...], resultado, motivo}`. Não toca em DB de orquestração, não envia mensagem, não decide humano vs IA.

### Task 3.1: Criar workflow

- [ ] **Step 1: Criar workflow vazio**

Nome: `Ellenco - Ambiente de Produção`. Tag: `ellenco`.

### Task 3.2: Webhook trigger + Respond imediato

- [ ] **Step 1: Adicionar node Webhook**

- **Method:** POST
- **Path:** `ellenco/agents/production`
- **Respond:** `Using Respond to Webhook node`

Renomear pra `post/agent`.

- [ ] **Step 2: Adicionar node Respond to Webhook**

> **Decisão de arquitetura:** vamos seguir o template **literalmente** e responder imediatamente com `{queued: true}`. O agente continua processando em background. A resposta real vai como **callback HTTP** pro endpoint do consumer.
>
> **Alternativa simples** (se quiser bloquear): use **Respond:** `When Last Node Finishes` no webhook e devolva o resultado direto. Trade-off: latência mais alta no consumer (espera 15s+ pelo debounce) vs simplicidade. **Sugestão:** começar com bloqueante (mais simples), refatorar pra callback se latência incomodar.

Vou descrever o caminho **bloqueante** abaixo (mais simples). Se quiser callback, ajustar Step 2 e o final do workflow.

Mude o webhook **Respond:** `When Last Node Finishes` (em vez de Respond to Webhook node). Pode ignorar o node Respond to Webhook nesse caso.

### Task 3.3: `initData` (parse do payload)

- [ ] **Step 1: Adicionar node Edit Fields (Set)**

- **Mode:** JSON
- **JSON Output:** `={{ $json.body }}`

Renomear pra `initData`.

### Task 3.4: Switch tipo de mensagem

- [ ] **Step 1: Adicionar node Switch**

- **Mode:** Rules
- **Rule 1:** `={{ $json.message.type }}` equals `text`
- **Rule 2:** `={{ $json.message.type }}` equals `audio`

Renomear pra `messageType`.

> **Decisão Phase 5:** rota audio fica como **TODO**. Por enquanto, mantenha a saída `audio` desconectada (ou conecte direto pra `noOp` que ignora). Phase 5 implementa transcrição.

### Task 3.5: Caminho texto direto

- [ ] **Step 1: Adicionar node Edit Fields**

Renomear pra `setText`.
- **Mode:** Manual mapping
- **Field:** `text` = `={{ $json.message.content.body }}` (Chatwoot envia em `body`)

### Task 3.6: Join texto

- [ ] **Step 1: Adicionar node Merge**

Mode: Append. Renomear pra `joinText`.

Conectar saídas de `setText` (e futuramente `setAudio`) pro merge.

### Task 3.7: Debounce — `pushToQueue`

**Por quê:** entra na fila Redis com timestamp e mid (message id). Permite agrupar mensagens consecutivas.

- [ ] **Step 1: Adicionar node Redis**

- **Operation:** Push
- **List:** Yes (Push to list)
- **Tail:** Yes (RPUSH)
- **Key:** `=client:{{ $json.candidato.id }}:conversation:{{ $json.chatwoot.conversation_id }}:messagesQueue`
  - Adapte se quiser usar `lead.phone` em vez de `candidato.id`. Sugiro `candidato.id` pra estabilidade.
- **Value:** `={{ JSON.stringify({ message: $json.text, timestamp: $now.toSeconds(), mid: $json.message.id }) }}`

Renomear pra `pushToQueue`.

### Task 3.8: Debounce — `getQueue`

- [ ] **Step 1: Adicionar node Redis**

- **Operation:** Get
- **Key:** mesma do `pushToQueue`
- **Key Type:** List
- **Get all values:** Yes

Renomear pra `getQueue`.

### Task 3.9: Debounce — `checkTimer` (switch)

> **Esse é o coração do debounce.** Lê o estado da fila Redis e decide: já processo? continuo esperando? aborto (mensagem mais nova chegou)?

- [ ] **Step 1: Adicionar node Switch**

- **Mode:** Rules
- **Rule 1 ("Nada a Fazer"):** `={{ JSON.parse($json.value[0]).mid }}` not equals `={{ $json.message.id }}`
  - Mensagem mais nova chegou ou foi processada por outra execução. Aborta.
- **Rule 2 ("Proceder"):** `={{ JSON.parse($json.value[$json.value.length-1]).timestamp }}` ≥ `={{ $now.minus({ seconds: $json.response.totalQueueTime }).toSeconds() }}`
  - Timeout ainda não estourou. Espera mais.
  - **Atenção:** essa lógica precisa ser **invertida** — se a última msg chegou nos últimos `totalQueueTime` segundos, esperamos; se já passou, processamos. Confirme com o JSON canônico em `reference/pure-pilates/Ambiente de Produção.json` node `checkTimer`.

Renomear pra `checkTimer`.

> **Recomendação:** abra `reference/pure-pilates/Ambiente de Produção.json` linhas ~290-380 e replique exatamente as 3 regras + saídas. A lógica é sutil — copiar é mais seguro que reinterpretar.

### Task 3.10: Debounce — `queueCheck` (wait + loop)

- [ ] **Step 1: Adicionar node Wait**

- **Resume:** After Time Interval
- **Wait Amount:** `={{ $json.response.checkQueue }}` segundos (default 5)

Renomear pra `queueCheck`.

- [ ] **Step 2: Loop back**

Conectar saída de `queueCheck` de volta pra `getQueue`. Cria o loop "espera 5s → relê fila → checkTimer".

### Task 3.11: `chatInput` + `deleteQueue`

- [ ] **Step 1: Adicionar node Edit Fields (`chatInput`)**

Quando `checkTimer` saída "Proceder":
- **Field:** `chatInput` = `={{ $json.value.map(v => JSON.parse(v).message).join('\n\n') }}`

Junta todas as mensagens da fila num bloco só.

- [ ] **Step 2: Adicionar node Redis (`deleteQueue`)**

- **Operation:** Delete
- **Key:** mesma chave usada em `pushToQueue`/`getQueue`

### Task 3.12: Recuperar histórico (`chatHistory`)

- [ ] **Step 1: Adicionar node Postgres (Chat Memory)**

> **Decisão:** vamos usar **Postgres Chat Memory node** apontando pra tabela `conversations` do Supabase, igual ao Pure Pilates. A tabela já tem o schema certo (`session_id, message jsonb`).

Adicionar node `Postgres Chat Memory` (sub-node do Agent).
- **Credential:** `Postgres - Ellenco`
- **Table:** `conversations`
- **Session ID:** `={{ $json.candidato.id }}` (bigint, bate com FK `session_id → candidatos_pre_aprovados.id`)
- **Context Window Length:** 12 (igual ao template)

Esse node não vai isolado — ele se conecta como input do Agent (Step 3.13).

### Task 3.13: Agente IA (gpt-4.1-mini)

- [ ] **Step 1: Adicionar node AI Agent**

Sub-nodes:
- **Chat Model:** OpenAI Chat Model
  - **Credential:** OpenAI Ellenco (cred já existente, confirme nome)
  - **Model:** `gpt-4.1-mini`
  - **Temperature:** 0.2 (mantém o do fluxo 3 atual)
  - **Frequency Penalty:** 0.7
  - **Presence Penalty:** 0.3
  - **Top P:** 0.8
- **Memory:** Postgres Chat Memory (Task 3.12)
- **Output Parser:** Structured Output Parser (próximo step)

- [ ] **Step 2: Configurar System Prompt**

> **Portar do fluxo 3 atual.** Abra `clients/Ellenco/original/3. AGENTE IA - PRINCIPAL.json` e localize o node do AI Agent. Copie o **System Message** completo (persona "Agente Virtual RH Ellenco", regras de coleta de docs, etc.) e cole aqui sem alterações.
>
> **Não invente prompt** — o atual já está validado em produção.

- [ ] **Step 3: Configurar User Message**

User message: `={{ $json.chatInput }}`

### Task 3.14: Output parser estruturado

- [ ] **Step 1: Adicionar node Structured Output Parser**

Como sub-node do Agent. Schema:
```json
{
  "type": "object",
  "properties": {
    "messages": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Lista de mensagens a enviar pro candidato (uma por bubble)"
    },
    "resultado": {
      "type": "string",
      "enum": ["em_andamento", "aprovado", "reprovado", "stand_by"],
      "description": "Estado da entrevista após essa interação"
    },
    "motivo": {
      "type": "string",
      "description": "Motivo do resultado (livre)"
    }
  },
  "required": ["messages", "resultado"]
}
```

> **Decisão:** o output do agente Ellenco precisa ser estruturado pra que o consumer consiga separar `messages` (envia pro WhatsApp) de `resultado/motivo` (vai pra `conclusão` no Supabase). Isso é diferente do template Pure que devolve só texto — adaptação Ellenco.

### Task 3.15: Devolver resposta

- [ ] **Step 1: Adicionar node Edit Fields**

- **Mode:** Manual mapping
- **Fields:**
  - `messages` = `={{ $json.output.messages }}`
  - `resultado` = `={{ $json.output.resultado }}`
  - `motivo` = `={{ $json.output.motivo }}`

Renomear pra `outputFormat`.

Como o webhook está com **Respond: When Last Node Finishes**, esse será o último node — o n8n devolve o JSON automaticamente.

### Task 3.16: Conectar e ativar

- [ ] **Step 1: Layout esperado**

```
post/agent → initData → messageType
  text → setText ──┐
                   │
                   ├→ joinText → pushToQueue → getQueue → checkTimer
                   │                                        ├ "Nada a Fazer" → STOP
                   │                                        ├ "Esperar"  → queueCheck → getQueue (loop)
                   │                                        └ "Proceder" → chatInput → deleteQueue → AI Agent → outputFormat
  audio → noOp ────┘
```

- [ ] **Step 2: Ativar workflow**

Toggle Active.

### Task 3.17: Validação isolada

- [ ] **Step 1: Disparar via curl**

```bash
curl -X POST https://n8n.ellenco.com.br/webhook/ellenco/agents/production \
  -H "Content-Type: application/json" \
  -d '{
    "lead": {"name": "Teste", "phone": "+5511999999999"},
    "chatwoot": {"conversation_id": 999, "account_id": 1},
    "message": {"id": "test-1", "type": "text", "content": {"body": "Olá, quero enviar meus documentos"}},
    "candidato": {"id": 1, "name": "Teste", "phone": "+5511999999999"},
    "response": {"totalQueueTime": 15, "checkQueue": 5, "messagesDelay": 3}
  }'
```

- [ ] **Step 2: Verificar resposta**

Esperado (após ~15s pelo debounce):
```json
{
  "messages": ["...resposta do agente..."],
  "resultado": "em_andamento",
  "motivo": "..."
}
```

- [ ] **Step 3: Verificar Postgres `conversations`**

```sql
SELECT * FROM conversations WHERE session_id = 1 ORDER BY id DESC LIMIT 5;
```
Deve ter 2 registros novos (1 user message + 1 ai message).

---

## Phase 4: Workflow DLQ (`ellenco/error_messages`)

### Diff vs fluxo 3 atual

**Esse workflow não existia.** É 100% novo. Cópia direta do template `reference/pure-pilates/error_messages.json`.

### Task 4.1: Criar workflow

- [ ] **Step 1: Criar workflow vazio**

Nome: `ellenco/error_messages`. Tag: `ellenco`.

### Task 4.2: Adicionar trigger

- [ ] **Step 1: Adicionar node RabbitMQ Trigger**

- **Credential:** `RabbitMQ - Ellenco`
- **Queue:** `error/messages`
- **Options → Parallel Messages:** 5

Renomear pra `error/messages`.

### Task 4.3: Parse + filter 24h

- [ ] **Step 1: Adicionar node Edit Fields (Set)**

- **Mode:** JSON
- **JSON Output:** `={{ $json.content.parseJson() }}`

Renomear pra `initData`.

- [ ] **Step 2: Adicionar node Filter**

- **Condition:** dateTime `={{ $json.message.timestamp.toDateTime('s') }}` is after `={{ $now.minus(24, 'hour') }}`

Renomear pra `Filter`.

### Task 4.4: Reinjetar via webhook do producer

- [ ] **Step 1: Adicionar node HTTP Request**

- **Method:** POST
- **URL:** `https://n8n.ellenco.com.br/webhook/ellenco/oficial/rabbit`
- **Send Body:** Yes
- **Body:** `={{ $json.toJsonString() }}`

Renomear pra `sendMessage`.

### Task 4.5: Adicionar webhook interno no Producer

> **Atenção:** o template tem um webhook auxiliar `purePilates/oficial/rabbit` no producer que apenas pega o JSON e republica em `messagesExchange`. Vamos adicionar o mesmo pattern no producer Ellenco.

- [ ] **Step 1: Voltar ao workflow `ellenco/rabbit_post`**

- [ ] **Step 2: Adicionar node Webhook**

- **Method:** POST
- **Path:** `ellenco/oficial/rabbit`
- **Respond:** Immediately

Renomear pra `rabbitWebhook`.

- [ ] **Step 3: Conectar `rabbitWebhook` direto ao `receiveMessages`**

(O JSON que vem do DLQ já está normalizado, não precisa passar pelo `initData2` de novo.)

### Task 4.6: Ativar e validar

- [ ] **Step 1: Ativar `ellenco/error_messages`**

- [ ] **Step 2: Publicar mensagem manualmente em `error/messages`**

Use o admin RabbitMQ pra publicar payload com `message.timestamp` recente. Verificar:
- Workflow executa.
- Filtro passa (timestamp dentro de 24h).
- HTTP request pro `rabbitWebhook` retorna 200.
- Mensagem aparece na queue `post/messages`.

- [ ] **Step 3: Testar mensagem antiga**

Publicar com `message.timestamp` de >24h atrás. Filtro deve barrar — workflow termina sem enviar.

---

## Phase 5: Cutover Fluxo 3 — virada do WhatsApp (Evolution → Chatwoot)

> **Escopo desta phase:** apenas o ecossistema **WhatsApp inbound** (Phases 1-4). Fluxos 1, 2, 3.1 e Excluir Pasta têm cutover próprio nas Phases 6-8. **Recomendação de ordem real de execução:** Phase 5 (WhatsApp) → Phase 7 (Documentos, depende do producer 3) → Phase 6 (Forms) → Phase 8 (Cleanup). Justificativa em Phase 9.

### Task 5.1: Última verificação

- [ ] **Step 1: Checklist de prontidão**

Antes de virar a chave:
- [ ] `ellenco/rabbit_post` ativo e respondendo no webhook
- [ ] `ellenco/post_messages` ativo, consumindo da queue
- [ ] `Ellenco - Ambiente de Produção` ativo
- [ ] `ellenco/error_messages` ativo
- [ ] Credenciais OK: RabbitMQ Ellenco, Chatwoot Token, Postgres Ellenco
- [ ] Schema Supabase: `id_chatwoot` populável; `conversations` aceita writes
- [ ] Inbox Chatwoot configurado com webhook apontando pro producer

### Task 5.2: Modo paralelo (opcional, recomendado)

> **Por quê:** rodar fluxo 3 antigo (Evolution) e novo (Chatwoot) em paralelo por 1-2 dias, comparando comportamento, é o jeito mais seguro. Só quando tiver confiança, desliga o antigo.

- [ ] **Step 1: Manter fluxo 3 antigo ativo**

Não desative `3. AGENTE IA - PRINCIPAL` ainda.

- [ ] **Step 2: Apontar Chatwoot pra inbox de testes**

Use um número WhatsApp diferente (linha de testes) conectado a um inbox separado no Chatwoot, mandando webhook pro producer novo. Volume de testes: você + 2-3 candidatos pilotos.

- [ ] **Step 3: Monitorar 24-48h**

Critérios de sucesso:
- Mensagens entram na queue (verificar admin RabbitMQ)
- Consumer processa sem acumular `error/messages` em volume anormal (<5%)
- Respostas chegam no Chatwoot dentro de 30s do envio do candidato
- `conversations` cresce no Postgres
- DLQ filtra ≥24h corretamente

### Task 5.3: Virada total

- [ ] **Step 1: Apontar inbox principal pro producer novo**

No Chatwoot, atualizar a webhook URL do inbox principal (Task 0.3 step 4) pra `ellenco/oficial/messages`.

- [ ] **Step 2: Desativar `3. AGENTE IA - PRINCIPAL`**

Toggle Active OFF. **Não delete** o workflow ainda — fica como rollback option por 30 dias.

- [ ] **Step 3: Anunciar pra equipe RH**

Avisar o time que de agora em diante o atendimento WhatsApp passa pelo Chatwoot. Eles devem usar a inbox do Chatwoot pra ver conversas (e atribuir a humano quando necessário).

### Task 5.4: Plano de rollback

> Se algo quebrar feio nos primeiros dias:

1. No Chatwoot: voltar webhook URL pra `whatsapp-ellenco` (paths antigos do Evolution).
   - **Atenção:** isso só funciona se o Evolution API ainda estiver conectado ao número WhatsApp. Se já tiver migrado pra Cloud API oficial via Chatwoot, **não dá pra voltar fácil** — esse é o ponto mais arriscado da migração.
2. Reativar `3. AGENTE IA - PRINCIPAL`.
3. Pausar workflows novos (deixar consumer ouvindo a queue, mas sem `Active`).

### Task 5.5: Cleanup pós-cutover (após 30 dias estável)

- [ ] **Step 1: Arquivar fluxo 3 antigo**

Exporte o JSON do `3. AGENTE IA - PRINCIPAL` pra `clients/Ellenco/original/_archived/` e delete do n8n.

- [ ] **Step 2: Documentar no `MEMORY.md` do projeto**

Adicione memória nova `template_decision_ellenco_chatwoot.md` com aprendizados que valham pra próximos clientes.

---

## Phase 6: Ecossistema Forms (Fluxos 1 + 2 → 3 workflows novos)

### Diff vs fluxos 1+2 atuais

| Aspecto | Hoje (Fluxos 1+2) | Depois (3 workflows) | Por quê |
|---|---|---|---|
| Topologia | 2 workflows: producer trivial + consumer monolítico fazendo TUDO | 3 workflows: producer + consumer (orquestrador) + agente classifier | Mesmo padrão das Phases 1-3 — agente vira "cérebro puro" reutilizável |
| Publicação | `filas-forms-ellenco` (queue direta, sem exchange) | `messagesExchange` (direct) + routing key `forms/messages` | Centraliza no exchange usado por todos os ecossistemas; abre porta pra DLQ |
| Canal de saída WhatsApp | Evolution API (`messages-api` send + `chat-api` validate) | Chatwoot API (`POST conversations` + `POST messages`) | Migração canal já decidida pra Fluxo 3 — manter consistência |
| Validação número WhatsApp | Evolution `whatsappNumbers` check | **Removida** — confiar no Cloud API rejeitar se inválido | Chatwoot/Cloud API valida no momento de enviar; check prévio vira redundante |
| Classificação aprovado/reprovado | LLM inline dentro do consumer | Workflow separado `Ellenco - Forms Classifier` | Mesma filosofia do "Ambiente de Produção" — testar/iterar prompt sem mexer na orquestração |
| Resposta Sheets de candidatos | Update em Sheets dentro do mesmo branch | Mantém — específico Ellenco | Não há equivalente no template, mas é regra de negócio. Fica no consumer. |
| Lookup vaga | Sheets `VAGAS` por ID dentro do consumer | Mantém — specifico Ellenco | Idem |
| Lock Redis cooldown rejeição | Set Redis 300s | Mantém | Mesma regra que o Fluxo 3 já honra (Task 2.8) |
| DLQ | Não tinha | `error/messages` igual Phases 4 | Ganha durabilidade |
| Erro = mensagem perdida | Sim (consumer gigante sem retry) | Não (publish em `error/messages` em qualquer falha) | DLQ template-style |

### Task 6.1: Workflow Producer `ellenco/forms_post`

> **Equivalente a:** Fluxo 1 atual (`1. PRODUCER - RABBITMQ.json`) reescrito pra usar `messagesExchange`.

- [ ] **Step 1: Criar workflow vazio**

Nome: `ellenco/forms_post`. Tag: `ellenco`.

- [ ] **Step 2: Webhook trigger**

Adicionar **Webhook** node:
- **HTTP Method:** POST
- **Path:** `ellenco/forms`
- **Respond:** Immediately
- **Authentication:** None

> **Por que mudar o path do webhook?** O atual (`/candidatos-forms`) era do Evolution-era. O Forms vai precisar ser apontado pra esse path novo. Atualização do destino do Forms acontece na Task 6.6 (cutover).

Renomear pra `formsWebhook`.

- [ ] **Step 3: Normalizar payload (`initFormData`)**

Adicionar **Edit Fields (Set)**:
- **Mode:** Manual mapping
- **Include other input fields:** No

| Campo | Tipo | Valor |
|---|---|---|
| `lead.name` | String | `={{ $json.body.nome \|\| $json.body.name }}` |
| `lead.phone` | String | `=+{{ ($json.body.celular \|\| $json.body.phone \|\| '').replace(/\D/g, '') }}` |
| `lead.email` | String | `={{ $json.body.email \|\| null }}` |
| `vaga.id` | String | `={{ $json.body.vaga_id \|\| $json.body.idVaga \|\| null }}` |
| `vaga.titulo` | String | `={{ $json.body.vaga_titulo \|\| $json.body.tituloVaga \|\| null }}` |
| `respostas` | Object | `={{ $json.body.respostas \|\| $json.body }}` |
| `event.type` | String | `forms_submitted` |
| `event.timestamp` | Number | `={{ $now.toSeconds() }}` |
| `source.app` | String | `google_forms` |
| `response.totalQueueTime` | Number | `0` |

> **Atenção:** os nomes exatos dos campos vindos do Forms (`nome` vs `name`, `celular` vs `phone`) **dependem do template do Form** que o cliente usou. Antes de implementar essa task, abra `clients/Ellenco/original/1. PRODUCER - RABBITMQ.json` e `2. PROCESSA RESPOSTAS DO FORMS.json` pra confirmar quais chaves o Fluxo 2 atual está lendo do `body`. Ajuste as expressões. **Não suponha de memória.**

Renomear pra `initFormData`.

- [ ] **Step 4: Filtro mínimo**

Adicionar **Filter** node:
- **Condition:** `={{ $json.lead.phone }}` is not empty (sem phone não dá pra responder)

Renomear pra `hasPhone`.

- [ ] **Step 5: Publicar em RabbitMQ**

Adicionar **RabbitMQ** node:
- **Credential:** `Credencial - RabbitMQ`
- **Mode:** Send Message to Exchange
- **Exchange:** `messagesExchange`
- **Exchange Type:** `direct`
- **Routing Key:** `forms/messages`
- **Message:** default (envia o JSON atual como string)

Renomear pra `publishForms`.

- [ ] **Step 6: Conectar e ativar**

Layout: `formsWebhook → initFormData → hasPhone → publishForms`. Toggle **Active**.

- [ ] **Step 7: Validar com curl**

```bash
curl -X POST https://n8n.ellenco.com.br/webhook/ellenco/forms \
  -H "Content-Type: application/json" \
  -d '{"nome":"Teste","celular":"+5511999999999","email":"a@b.com","vaga_id":"123","respostas":{"q1":"sim"}}'
```

Esperado: 200 OK. No admin RabbitMQ, queue `forms/messages` deve ter +1 mensagem.

### Task 6.2: Workflow Agente `Ellenco - Forms Classifier`

> **Equivalente a:** o **bloco do AI Agent** dentro do Fluxo 2 atual (entre extração de respostas e Switch resultado), extraído pra workflow próprio. Mesma filosofia da Phase 3 (agente WhatsApp).

- [ ] **Step 1: Criar workflow vazio**

Nome: `Ellenco - Forms Classifier`. Tag: `ellenco`.

- [ ] **Step 2: Webhook trigger + Respond**

Adicionar **Webhook** node:
- **Method:** POST
- **Path:** `ellenco/agents/forms`
- **Respond:** `When Last Node Finishes` (bloqueante; latência baixa esperada — 1 chamada LLM, sem debounce)

Renomear pra `formsAgentWebhook`.

- [ ] **Step 3: Parse payload (`initData`)**

**Edit Fields (Set)**:
- **Mode:** JSON
- **JSON Output:** `={{ $json.body }}`

Renomear pra `initData`.

- [ ] **Step 4: Montar contexto pro prompt (`buildContext`)**

**Edit Fields (Set)** com:

| Campo | Valor |
|---|---|
| `chatInput` | `={{ "Vaga: " + $json.vaga.titulo + "\n\nRequisitos: " + JSON.stringify($json.vaga.requisitos) + "\n\nRespostas do candidato: " + JSON.stringify($json.respostas) }}` |

> **Os requisitos da vaga vêm de onde?** O consumer (Task 6.3) vai buscar em Google Sheets ANTES de chamar esse agente e mandar `vaga.requisitos` no payload. Esse agente só recebe — não busca.

- [ ] **Step 5: AI Agent**

Adicionar node **AI Agent** com sub-nodes:
- **Chat Model:** OpenAI Chat Model
  - **Credential:** `OpenAi - Ellenco` (cred existente do Fluxo 2 atual)
  - **Model:** `gpt-4.1-mini` (alinhar com Phase 3 — Fluxo 2 atual usa `gpt-4o-mini`, pode ficar)
  - **Temperature:** 0.0 (classificação deve ser determinística)
- **Output Parser:** Structured Output Parser (próximo step)
- **System Prompt:**

> **Portar do Fluxo 2 atual.** Abra `clients/Ellenco/original/2. PROCESSA RESPOSTAS DO FORMS.json` e localize o node de AI Agent (provavelmente nome contém "classifica" ou "analisa"). Copie o **System Message** completo e cole aqui sem alteração. **Não invente prompt** — o atual já está validado em produção.

- **User Message:** `={{ $json.chatInput }}`

- [ ] **Step 6: Output Parser estruturado**

Sub-node **Structured Output Parser**. Schema:

```json
{
  "type": "object",
  "properties": {
    "resultado": {
      "type": "string",
      "enum": ["aprovado", "reprovado", "stand_by"],
      "description": "Decisão baseada nos requisitos da vaga vs respostas do candidato"
    },
    "motivo": {
      "type": "string",
      "description": "Justificativa curta (1-2 frases)"
    },
    "destaques": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Pontos relevantes do candidato (opcional)"
    }
  },
  "required": ["resultado", "motivo"]
}
```

> **Diferença vs Phase 3:** o output não tem `messages` — o consumer Forms (Task 6.3) que vai montar a mensagem de boas-vindas / rejeição com base em `resultado`. Decisão consciente pra desacoplar a "voz" do bot da decisão.

- [ ] **Step 7: Output formatado**

**Edit Fields (Set)** após o agente:

| Campo | Valor |
|---|---|
| `resultado` | `={{ $json.output.resultado }}` |
| `motivo` | `={{ $json.output.motivo }}` |
| `destaques` | `={{ $json.output.destaques \|\| [] }}` |

Renomear pra `outputFormat`.

- [ ] **Step 8: Conectar e ativar**

Layout: `formsAgentWebhook → initData → buildContext → AI Agent → outputFormat`.

Toggle **Active**.

- [ ] **Step 9: Validar com curl**

```bash
curl -X POST https://n8n.ellenco.com.br/webhook/ellenco/agents/forms \
  -H "Content-Type: application/json" \
  -d '{
    "vaga": {"titulo":"Auxiliar de RH","requisitos":["Ensino médio","CNH B"]},
    "respostas": {"escolaridade":"Médio","cnh":"B"}
  }'
```

Esperado: `{"resultado":"aprovado","motivo":"...","destaques":[...]}`.

### Task 6.3: Workflow Consumer `ellenco/forms_processor`

> **Equivalente a:** Fluxo 2 atual (`2. PROCESSA RESPOSTAS DO FORMS.json`) reescrito sem o agente inline e sem Evolution. Esse é o workflow mais denso da Phase 6.

- [ ] **Step 1: Criar workflow vazio**

Nome: `ellenco/forms_processor`. Tag: `ellenco`.

- [ ] **Step 2: Trigger RabbitMQ**

Adicionar **RabbitMQ Trigger**:
- **Credential:** `Credencial - RabbitMQ`
- **Queue:** `forms/messages`
- **Options → Parallel Messages:** 5
- **Options → Acknowledge Mode:** Execution Finishes

Renomear pra `forms/messages`.

- [ ] **Step 3: Parse JSON da fila (`initData`)**

**Edit Fields (Set)**:
- **Mode:** JSON
- **JSON Output:** `={{ $json.content.parseJson() }}`

Renomear pra `initData`.

- [ ] **Step 4: Variáveis de ambiente (`setVariables`)**

**Edit Fields (Set)** com (igual Task 2.4 + extras Forms):

| Campo | Valor |
|---|---|
| `chatwootUrl` | `https://chatwoot.ellenco.com.br` |
| `chatwootAccountId` | `1` |
| `chatwootInboxId` | `1` |
| `agentFormsUrl` | `https://n8n.ellenco.com.br/webhook/ellenco/agents/forms` |
| `errorWebhookUrl` | `https://n8n.ellenco.com.br/webhook/ellenco/oficial/rabbit` |
| `vagasSheetId` | `<COPIAR DO FLUXO 2 ATUAL — node "VAGAS" ou similar>` |
| `candidatosSheetId` | `<COPIAR DO FLUXO 2 ATUAL — node de update final>` |

> **Como achar os IDs:** abrir `clients/Ellenco/original/2. PROCESSA RESPOSTAS DO FORMS.json` no editor, buscar `documentId` ou `sheetId`. Vai aparecer 2 IDs distintos — um pra VAGAS, um pra Candidatos.

- [ ] **Step 5: Buscar requisitos da vaga (`getVaga`)**

**Google Sheets** node:
- **Credential:** `Credencial - Google Sheets`
- **Operation:** Get Row(s)
- **Document ID:** `={{ $json.vagasSheetId }}`
- **Sheet:** "VAGAS" (confirmar nome real no Fluxo 2 atual)
- **Filter By:** `id` = `={{ $json.vaga.id }}`

> **Output esperado:** linha com colunas tipo `id`, `titulo`, `requisitos` (talvez separados por vírgula). Adapte conforme schema real da planilha.

- [ ] **Step 6: Mesclar vaga ao payload (`mergeVaga`)**

**Edit Fields (Set)**:
- Manter campos do `initData` E adicionar:
  - `vaga.requisitos` = `={{ $json.requisitos.split(',').map(r => r.trim()) }}` (ou estrutura que o Sheets retornar)
  - `vaga.titulo` = `={{ $json.titulo }}`

> **Truque n8n:** se `getVaga` perder o resto do payload original, use **Merge** node em modo `Combine` — chave `position` 0 — pra unir o item original do `initData` com a linha do Sheets.

- [ ] **Step 7: Chamar agente classifier (`callClassifier`)**

**HTTP Request**:
- **Method:** POST
- **URL:** `={{ $json.agentFormsUrl }}`
- **Send Body:** Yes
- **Body Content Type:** JSON
- **JSON Body:** `={{ $json.toJsonString() }}`
- **Options → Timeout:** 30000ms

- [ ] **Step 8: Switch por resultado**

**Switch** node:
- **Mode:** Rules
- **Rule 1:** `={{ $json.resultado }}` equals `aprovado`
- **Rule 2:** `={{ $json.resultado }}` equals `reprovado`
- **Rule 3:** `={{ $json.resultado }}` equals `stand_by`

Renomear pra `resultado`.

#### Branch APROVADO

- [ ] **Step 9a: Criar contato no Chatwoot (`createContact`)**

**HTTP Request**:
- **Method:** POST
- **URL:** `={{ $json.chatwootUrl }}/api/v1/accounts/{{ $json.chatwootAccountId }}/contacts`
- **Authentication:** Header Auth → `Chatwoot - Api Access Token`
- **Body JSON:**
```
={{ { name: $json.lead.name, phone_number: $json.lead.phone, email: $json.lead.email, inbox_id: $json.chatwootInboxId } }}
```

> **Atenção 409 Conflict:** se o contato já existir (phone único no Chatwoot), API responde 409. Configure o node com **On Error: Continue (using error output)** e use o branch de erro pra fazer **GET /api/v1/accounts/{id}/contacts/search?q={phone}** e recuperar o `id` existente.

- [ ] **Step 9b: Criar conversa no Chatwoot (`createConversation`)**

**HTTP Request**:
- **Method:** POST
- **URL:** `={{ $json.chatwootUrl }}/api/v1/accounts/{{ $json.chatwootAccountId }}/conversations`
- **Authentication:** Header Auth → `Chatwoot - Api Access Token`
- **Body JSON:**
```
={{ { source_id: $json.lead.phone.replace(/\D/g,''), inbox_id: $json.chatwootInboxId, contact_id: $json.contactId } }}
```

> Anota o `id` retornado — é o `conversation_id` (= `id_chatwoot` que o Fluxo 3 vai precisar mais tarde).

- [ ] **Step 9c: Criar pasta no Drive (`createFolder`)**

**Google Drive** node:
- **Credential:** `Credemcial - Google Drive` (sic, é o nome atual no n8n Ellenco)
- **Operation:** Create Folder
- **Name:** `={{ $('mergeVaga').item.json.lead.name + ' - ' + $('mergeVaga').item.json.lead.phone }}`
- **Parent Folder:** ID da pasta-mãe (copiar do Fluxo 2 atual node `pastaCandidatos` ou similar)

> **Por que esse padrão de nome?** Fluxo 2 atual usa `NOME - CELULAR`. Mantém pra o Fluxo 3.1 (Phase 7) achar a pasta pelo `id_pasta` do Supabase.

Anota o `id` da pasta criada.

- [ ] **Step 9d: Inserir candidato no Supabase (`insertCandidato`)**

**Postgres** node:
- **Credential:** `Postgres - Ellenco`
- **Operation:** Insert
- **Schema:** public
- **Table:** `candidatos_pre_aprovados`
- **Columns:** `name`, `phone`, `email`, `id_chatwoot`, `id_pasta`, `vaga_id`, `vaga_titulo`, `motivo_aprovacao`, `destaques`
- **Mapping:** referenciar campos:
  - `name` → `={{ $('mergeVaga').item.json.lead.name }}`
  - `phone` → `={{ $('mergeVaga').item.json.lead.phone }}` (formato E.164 — observar diferença com a base atual que tem JID; ver Apêndice E)
  - `email` → `={{ $('mergeVaga').item.json.lead.email }}`
  - `id_chatwoot` → `={{ $('createConversation').item.json.id }}`
  - `id_pasta` → `={{ $('createFolder').item.json.id }}`
  - `vaga_id` → `={{ $('mergeVaga').item.json.vaga.id }}`
  - `vaga_titulo` → `={{ $('mergeVaga').item.json.vaga.titulo }}`
  - `motivo_aprovacao` → `={{ $('callClassifier').item.json.motivo }}`
  - `destaques` → `={{ JSON.stringify($('callClassifier').item.json.destaques) }}`

> **Schema check:** rode `SELECT column_name FROM information_schema.columns WHERE table_name = 'candidatos_pre_aprovados';` e remova/adapte campos que não existirem. **Não rode DDL agora** — se faltar coluna importante, registra como debt e usa só o que tem.

- [ ] **Step 9e: Enviar primeira mensagem WhatsApp (`sendWelcome`)**

**HTTP Request**:
- **Method:** POST
- **URL:** `={{ $('mergeVaga').item.json.chatwootUrl }}/api/v1/accounts/{{ $('mergeVaga').item.json.chatwootAccountId }}/conversations/{{ $('createConversation').item.json.id }}/messages`
- **Authentication:** Header Auth → `Chatwoot - Api Access Token`
- **Body JSON:**
```
={{ { content: "<mensagem de boas-vindas — copiar do Fluxo 2 atual node sendMessage branch aprovado>", message_type: "outgoing", content_type: "text" } }}
```

> **Mensagem real:** **NÃO invente.** Abrir Fluxo 2 atual e copiar o template de mensagem de boas-vindas literalmente. Se tiver placeholder tipo `{{nome}}`, traduzir pra expressão n8n: `${$('mergeVaga').item.json.lead.name}`.

> **Importante:** essa primeira mensagem é o que faz o candidato responder no WhatsApp. A partir da resposta dele, **o Fluxo 3 (Phase 1-3) assume** — Chatwoot recebe a resposta dele e o producer WhatsApp puxa pra `post/messages`. O loop fecha.

- [ ] **Step 9f: Atualizar Sheets candidatos (`updateSheetCandidatos`)**

**Google Sheets** node:
- **Credential:** `Credencial - Google Sheets`
- **Operation:** Append Row
- **Document ID:** `={{ $('mergeVaga').item.json.candidatosSheetId }}`
- **Sheet:** confirmar nome (provavelmente "Candidatos" ou similar)
- **Mapping:** copiar campos do node equivalente no Fluxo 2 atual

#### Branch REPROVADO

- [ ] **Step 10a: Decisão — criar conversation ou só enviar mensagem?**

> **Decisão Ellenco:** mesmo reprovado, ainda queremos enviar a mensagem de rejeição via WhatsApp pra fechar o loop com o candidato. **Mas não criamos contato/conversation no Chatwoot pra reprovado** — Cloud API permite enviar mensagem template/texto pra um número novo via inbox. Se o Cloud API requer contato preexistente, **criamos contato (mas não conversation)** ou criamos ambos e marcamos a conversation como `resolved` logo após.

> **Recomendação prática:** criar contato + conversation iguais ao branch APROVADO (Steps 9a, 9b), enviar mensagem de rejeição (próximo step), e marcar conversation como `resolved` imediatamente. Isso mantém histórico no Chatwoot e fecha o ticket.

Replicar Steps 9a, 9b nesse branch (mesma config).

- [ ] **Step 10b: Enviar mensagem de rejeição (`sendRejection`)**

**HTTP Request** igual Step 9e mas com `content` = mensagem de rejeição copiada do Fluxo 2 atual branch reprovado.

- [ ] **Step 10c: Marcar conversation como resolvida (`resolveConversation`)**

**HTTP Request**:
- **Method:** POST
- **URL:** `={{ $json.chatwootUrl }}/api/v1/accounts/{{ $json.chatwootAccountId }}/conversations/{{ $('createConversation').item.json.id }}/toggle_status`
- **Authentication:** Header Auth → `Chatwoot - Api Access Token`
- **Body JSON:** `={{ { status: "resolved" } }}`

- [ ] **Step 10d: Lock Redis cooldown (`setCooldown`)**

**Redis** node:
- **Credential:** `Redis - Ellenco` (ou nome existente — verificar)
- **Operation:** Set
- **Key:** `={{ "cooldown:" + $json.lead.phone.replace(/\D/g,'') }}`
- **Value:** `reprovado`
- **Expire:** Yes
- **TTL:** 300 (5min, igual atual)

> **Por que esse formato de chave?** Match com a chave que o consumer WhatsApp (Task 2.8) vai checar. **Tem que bater.** Se o Fluxo 3 atual usa formato diferente (ex.: só o phone sem prefixo), padronize **agora** — escolha um formato e use em ambos.

- [ ] **Step 10e: Atualizar Sheets reprovados**

**Google Sheets** Append Row em sheet "Reprovados" (confirmar nome no Fluxo 2 atual).

#### Branch STAND_BY

> **Decisão Ellenco:** caso intermediário — agente em dúvida. Hoje o Fluxo 2 atual provavelmente trata como "humano avalia depois". Replicar comportamento: enviar pra Sheets de "stand_by" e parar (sem mensagem WhatsApp).

- [ ] **Step 11: Append em Sheets stand_by**

**Google Sheets** Append Row em sheet correspondente. Sem outro side effect.

#### Convergência

- [ ] **Step 12: Conectar todos os branches a um Set final (`opcional`)**

Pode unir os 3 branches num **Merge → NoOp** só pra ter 1 ponto de saída visual no canvas. Ajuda no debug.

### Task 6.4: Configurar Error Workflow

- [ ] **Step 1: No `ellenco/forms_processor`, abrir Settings**

Em **Settings → Error Workflow → Select**, escolher: o producer principal (`ellenco/rabbit_post`) — porque ele tem o webhook auxiliar `/ellenco/oficial/rabbit` (Task 4.5) que republica em `messagesExchange`.

> **Alternativa explícita:** dentro do `forms_processor`, capturar erros via node **Error Trigger** + **HTTP Request** pra `errorWebhookUrl` (já no `setVariables`). Mais visível, menos elegante. Escolha a forma que ficar mais legível pro time.

### Task 6.5: Validação isolada

- [ ] **Step 1: Publicar mensagem manualmente em `forms/messages`**

No admin RabbitMQ, **Queues → forms/messages → Publish message**:

```json
{
  "lead": {"name":"Teste Aprovado","phone":"+5511999999999","email":"a@b.com"},
  "vaga": {"id":"<id-real-de-uma-vaga-da-sheet>","titulo":""},
  "respostas": {"q1":"sim"},
  "event": {"type":"forms_submitted","timestamp":1730000000},
  "source": {"app":"google_forms"},
  "response": {"totalQueueTime":0}
}
```

- [ ] **Step 2: Executar workflow uma vez no n8n**

Verificar:
- `getVaga` retornou linha da Sheets
- `callClassifier` retornou `aprovado` ou `reprovado`
- Branch certa executou
- Mensagem chegou no WhatsApp do número de teste
- Linha apareceu em Sheets candidatos
- Row apareceu em `candidatos_pre_aprovados` (Postgres)
- `id_chatwoot` populado com ID real da conversation

- [ ] **Step 3: Smoke test branch reprovado**

Repetir Step 1 com `respostas` claramente fora de spec pra forçar reprovação. Verificar Redis lock criado: `redis-cli GET cooldown:5511999999999` → deve ter TTL ~300s.

### Task 6.6: Cutover Forms

- [ ] **Step 1: Apontar Google Forms / Apps Script pro novo webhook**

> **Onde mudar:** o Forms do Ellenco usa um Apps Script `onFormSubmit` que faz POST no n8n. Ou um Zap/Make. **Antes de tocar**, descubra qual é (perguntar pro time se não estiver documentado).

- Mudar URL do POST de: `https://n8n.ellenco.com.br/webhook/candidatos-forms`
- Para: `https://n8n.ellenco.com.br/webhook/ellenco/forms`

- [ ] **Step 2: Desativar Fluxo 1 e 2 antigos**

No n8n, toggle **Active OFF** em `1. PRODUCER - RABBITMQ` e `2. PROCESSA RESPOSTAS DO FORMS`.

> **Não delete ainda.** Mantenha como rollback option por 30 dias.

- [ ] **Step 3: Validar com submissão real**

Pedir pro time enviar 1 candidatura de teste no Form. Acompanhar:
- Webhook do `ellenco/forms_post` recebe (Executions n8n)
- Mensagem entra na queue `forms/messages`
- Consumer processa
- Candidato chega no WhatsApp ou recebe mensagem de rejeição

- [ ] **Step 4: Monitorar 24h**

Volume de erros em `error/messages` deve ficar baixo (<5%). Se algo quebrar, voltar URL do Forms pra o webhook antigo (rollback rápido).

---

## Phase 7: Ecossistema Documentos (Fluxo 3.1)

### Diff vs Fluxo 3.1 atual

| Aspecto | Hoje | Depois | Por quê |
|---|---|---|---|
| Trigger | Webhook custom `/documentos` (alimentado pelo Evolution recebendo anexo via instância "ellenco") | **Branch no producer principal `ellenco/rabbit_post`** que detecta anexo e publica em `documents/messages` | Chatwoot já entrega o anexo no mesmo webhook que entrega texto — separar via routing key, não via webhook duplicado |
| Source dos headers (`id_client`, `id_pasta`, `nome`) | Headers HTTP custom enviados pela origem | **Lookup em `candidatos_pre_aprovados` pelo `id_chatwoot`** (igual Task 2.5) | Chatwoot não envia esses headers; consumer recupera via DB |
| Download do binário | Já vinha em `body` do webhook (Evolution embute) | HTTP GET na URL do anexo (`data_url` do payload Chatwoot) | Diferença de payload entre os 2 canais |
| Conversão PDF→PNG (`convert.ellenco.com.br`) | Mantém | Mantém | API interna funciona, sem motivo pra trocar |
| Análise IA (Gemini PDF, OpenAI Vision imagem) | Mantém | Mantém | Lógica funciona |
| Renomear arquivo (CTPS / CNH-RG / Comp.Residência) | Mantém | Mantém | Regra negócio Ellenco |
| Update `documents` Supabase | Mantém | Mantém | Mantém |
| Resposta ao candidato | Não respondia (assíncrono pelo Evolution) | **Sim — envia mensagem de confirmação no Chatwoot** ("Documento X recebido com sucesso") | Melhor UX, agora que temos canal sincronizado |

> **Decisão chave:** o Fluxo 3.1 atual reage a um webhook que **já tem o binário e os headers prontos**. Pra adaptar ao Chatwoot, o consumer precisa **recuperar contexto** (id_pasta do candidato) ANTES de processar o anexo. Mais lookups, mas tudo via DB que já temos.

### Task 7.1: Adicionar branch no producer `ellenco/rabbit_post`

> Edita o workflow da Phase 1. Não cria workflow novo.

- [ ] **Step 1: Abrir `ellenco/rabbit_post`**

- [ ] **Step 2: Adicionar Switch após `eventType`**

Inserir **Switch** entre `eventType` e `receiveMessages` (Task 1.7). Renomear pra `routeMessage`.
- **Mode:** Rules
- **Rule 1 (text):** `={{ $json.message.type }}` equals `text`
- **Rule 2 (attachment):** `={{ $json.message.type }}` not equals `text` AND `={{ $json.message.type }}` is not empty

Conectar:
- Saída `text` → `receiveMessages` (já existente, publica em `post/messages`)
- Saída `attachment` → novo node `receiveDocuments` (próximo step)

> **Por que `type !== 'text'`?** O `initData2` setou `message.type` a partir de `attachments[0].file_type` se houvesse anexo, senão `'text'`. Então `type ∈ {image, audio, video, file, document}` indica anexo. Match negativo é mais robusto que listar todos os tipos.

- [ ] **Step 3: Adicionar segundo RabbitMQ publish (`receiveDocuments`)**

Adicionar novo **RabbitMQ** node ao lado do `receiveMessages`:
- **Credential:** mesma
- **Mode:** Send Message to Exchange
- **Exchange:** `messagesExchange`
- **Exchange Type:** `direct`
- **Routing Key:** `documents/messages`
- **Message:** default

Renomear pra `receiveDocuments`.

- [ ] **Step 4: Salvar e testar**

Mandar foto de teste via WhatsApp pro inbox Chatwoot. Verificar:
- `routeMessage` saiu pelo branch attachment
- Queue `documents/messages` ganhou +1 mensagem
- Queue `post/messages` NÃO ganhou (não duplicou)

### Task 7.2: Workflow Consumer `ellenco/documents_consumer`

- [ ] **Step 1: Criar workflow vazio**

Nome: `ellenco/documents_consumer`. Tag: `ellenco`.

- [ ] **Step 2: Trigger RabbitMQ**

**RabbitMQ Trigger**:
- **Queue:** `documents/messages`
- **Parallel Messages:** 3 (anexos são pesados — não saturar)
- **Acknowledge Mode:** Execution Finishes

Renomear pra `documents/messages`.

- [ ] **Step 3: Parse JSON (`initData`)**

Igual Task 2.3.

- [ ] **Step 4: setVariables**

Igual Task 2.4 + adicionais:

| Campo extra | Valor |
|---|---|
| `pdfConverterUrl` | `https://convert.ellenco.com.br/api/v1/convert/pdf/img` |
| `defaultFolderId` | `<copiar do Fluxo 3.1 atual — pasta-mãe quando candidato não tem pasta>` |

- [ ] **Step 5: Lookup do candidato (`tryGetClient`)**

Mesmo Postgres query da Task 2.5 (mesmo padrão de phone normalization + id_chatwoot).

- [ ] **Step 6: Branch — candidato existe?**

**IF**: `={{ $json.id }}` is not empty.
- True → segue
- False → publish em `error/messages` ou ignorar com log

> **Decisão:** se anexo chega de número que não está em `candidatos_pre_aprovados`, é caso anômalo (pessoa mandando documento sem ter feito Form). Atualmente Fluxo 3.1 espera `id_client` no header — não trata esse caso. Manter comportamento: descartar com log.

- [ ] **Step 7: Download do binário (`downloadAttachment`)**

**HTTP Request**:
- **Method:** GET
- **URL:** `={{ $('initData').item.json.message.content.data_url }}` (Chatwoot embeds `data_url` no payload do anexo)
- **Authentication:** Header Auth → `Chatwoot - Api Access Token` (alguns inboxes exigem)
- **Response → Response Format:** File / Binary
- **Response → Put Output in Field:** `data`

> **Atenção:** `data_url` do Chatwoot pode ser uma URL pública (CDN) ou interna. Testar primeiro qual o formato. Se for pública, autenticação vira opcional.

- [ ] **Step 8: Upload no Drive (`uploadFile`)**

**Google Drive** node:
- **Operation:** Upload
- **Binary Property:** `data`
- **Parent Folder:** `={{ $('tryGetClient').item.json.id_pasta }}` (id da pasta criada no Fluxo 2/Phase 6)
- **File Name:** `={{ $('initData').item.json.message.content.name \|\| 'documento_' + $now.toMillis() }}`

- [ ] **Step 9: Reabrir share + download (igual Fluxo 3.1 atual)**

Replicar nodes do Fluxo 3.1 atual:
- **Share** o arquivo (anyone with link, viewer)
- **Download** novamente (porque o LLM exige base64 limpo)
- **Convert to base64** se necessário

> **Estes são copy-paste do Fluxo 3.1 atual.** Abre o JSON original e replica nodes 1:1. Só ajusta o `parentFolder` pra usar `tryGetClient.id_pasta`.

- [ ] **Step 10: Switch tipo arquivo (PDF vs imagem)**

**Switch**:
- Rule 1: `={{ $json.fileType \|\| $json.mimeType }}` contains `pdf`
- Rule 2: `={{ $json.fileType \|\| $json.mimeType }}` contains `image`

#### Branch PDF

- [ ] **Step 11a: Converter PDF→PNG via API interna**

**HTTP Request** apontando pra `pdfConverterUrl`. Replicar do Fluxo 3.1 atual.

- [ ] **Step 11b: Análise Gemini**

**Google Gemini Chat Model** node + **AI Agent** com prompt de extração estruturada (copiar do Fluxo 3.1 atual). Output deve ser JSON com `{tipo: "carteira_de_trabalho|cnh_rg|comprovante_de_residencia|outros", dadosExtraidos: {...}}`.

#### Branch Imagem

- [ ] **Step 12: Análise OpenAI Vision (`gpt-4-mini` ou `gpt-4o-mini`)**

**OpenAI Chat Model** com Vision. Mesmo prompt de extração.

#### Convergência

- [ ] **Step 13: Switch por tipo de documento**

**Switch** com 4 saídas: `carteira_de_trabalho`, `cnh_rg`, `comprovante_de_residencia`, `outros`.

- [ ] **Step 14: Renomear arquivo (`renameFile`)**

**Google Drive** → Update file → name = `<Tipo Humano>` (ex.: "CNH/RG", "Carteira de Trabalho", "Comprovante de Residência").

> **Mapping exato:** copiar dos nodes equivalentes do Fluxo 3.1 atual.

- [ ] **Step 15: Update Supabase `documents`**

**Postgres** Update:
- **Table:** `documents`
- **Match Columns:** `client_id` = `={{ $('tryGetClient').item.json.id }}`
- **Values:** dependendo do tipo:
  - se `carteira_de_trabalho`: setar coluna `ctps` ou similar com dados extraídos
  - se `cnh_rg`: setar `cnh_rg`
  - se `comprovante_de_residencia`: setar `comp_residencia`

> **Schema check:** `SELECT column_name FROM information_schema.columns WHERE table_name = 'documents';` antes de configurar — confirmar nomes exatos.

- [ ] **Step 16: Confirmar pro candidato (`sendConfirmation`)**

**HTTP Request** pro Chatwoot `messages` endpoint:
- Body: `{ content: "<documento> recebido com sucesso ✅", message_type: "outgoing", content_type: "text" }`

> **Antes:** Fluxo 3.1 atual não confirmava porque o Evolution era assíncrono e não tinha contexto fácil de canal. Agora que estamos no Chatwoot/Cloud API, confirmar é trivial e melhora MUITO a UX.

- [ ] **Step 17: Conectar e deixar inativo**

Não ativar ainda — Phase 7 cutover ativa junto com Phase 5.

### Task 7.3: Cutover Documentos

- [ ] **Step 1: Pré-condição** — Phase 5 (cutover Fluxo 3) já feito. O producer `ellenco/rabbit_post` já está roteando texto pro `post/messages`; agora ativa documentos também.

- [ ] **Step 2: Ativar `ellenco/documents_consumer`**

Toggle Active.

- [ ] **Step 3: Desativar Fluxo 3.1 antigo**

`3.1 SALVAR DOCUMENTOS DOS CANDIDATOS` → toggle OFF. Não deletar (rollback 30 dias).

- [ ] **Step 4: Smoke test**

Mandar foto e PDF de documento real (carteira ou CNH) pro WhatsApp do número de teste. Verificar:
- Branch attachment do producer disparou
- `documents/messages` consumida
- Drive uploadou
- Análise IA rodou
- Arquivo renomeado certo
- Linha em `documents` atualizada
- Mensagem de confirmação chegou no WhatsApp

- [ ] **Step 5: Monitorar**

Mesmo critério: <5% em DLQ nas primeiras 24h.

---

## Phase 8: Ecossistema Cleanup (Excluir Pasta no Drive)

### Diff vs fluxo atual

| Aspecto | Hoje | Depois | Por quê |
|---|---|---|---|
| Trigger | Webhook `/supabase` direto disparado por Supabase delete event | **Webhook continua, mas agora publica em `cleanup/folders`**. Consumer separado faz delete. | Padrão template "tudo passa por broker". DLQ se Drive estiver fora. |
| Caminho | webhook → Drive deleteFolder | webhook → publish → consumer → Drive deleteFolder | +1 hop, mas ganha retry/DLQ |
| Latência | ~500ms | ~1-2s | Aceitável (não é caminho crítico) |
| Risco se Drive cair | Falha silenciosa, pasta órfã | DLQ retém 24h, time pode reprocessar | Trade-off vence |

> **Trade-off honesto:** pra UM webhook que faz UMA chamada de delete, RabbitMQ no meio é overkill comparado ao Fluxo 3 (que tem debounce, agente, 16 nodes). Vale a consistência arquitetural ("padrão template em todos os fluxos") + DLQ. Se o Luiz mudar de ideia depois, é trivial reverter pro webhook direto.

### Task 8.1: Workflow Producer `ellenco/cleanup_post`

- [ ] **Step 1: Criar workflow vazio**

Nome: `ellenco/cleanup_post`. Tag: `ellenco`.

- [ ] **Step 2: Webhook**

- **Method:** POST
- **Path:** `ellenco/cleanup/folder`
- **Respond:** Immediately

Renomear pra `cleanupWebhook`.

- [ ] **Step 3: Normalizar payload**

**Edit Fields (Set)**:
- `id_pasta` = `={{ $json.body.old_record?.id_pasta \|\| $json.body.record?.id_pasta }}`
- `id_candidato` = `={{ $json.body.old_record?.id }}`
- `event.type` = `cleanup_folder`
- `event.timestamp` = `={{ $now.toSeconds() }}`

Renomear pra `initCleanup`.

- [ ] **Step 4: Filtro**

**Filter**: `={{ $json.id_pasta }}` is not empty (sem id_pasta não tem o que deletar).

- [ ] **Step 5: Publish RabbitMQ**

**RabbitMQ**:
- Mode: Send Message to Exchange
- Exchange: `messagesExchange`
- Routing Key: `cleanup/folders`

Renomear pra `publishCleanup`.

- [ ] **Step 6: Conectar e ativar**

Layout: `cleanupWebhook → initCleanup → Filter → publishCleanup`. Toggle Active.

### Task 8.2: Workflow Consumer `ellenco/cleanup_consumer`

- [ ] **Step 1: Criar workflow vazio**

Nome: `ellenco/cleanup_consumer`. Tag: `ellenco`.

- [ ] **Step 2: Trigger RabbitMQ**

**RabbitMQ Trigger**:
- Queue: `cleanup/folders`
- Parallel: 3
- Ack Mode: Execution Finishes

Renomear pra `cleanup/folders`.

- [ ] **Step 3: Parse JSON**

`initData` igual outras phases.

- [ ] **Step 4: Drive deleteFolder**

**Google Drive**:
- Credential: `Credemcial - Google Drive`
- Operation: Delete
- Resource: Folder
- Folder ID: `={{ $json.id_pasta }}`
- **On Error:** Continue (using error output) — pasta pode ter sido deletada manualmente

Renomear pra `deleteFolder`.

- [ ] **Step 5: Conectar e ativar**

Layout: `cleanup/folders → initData → deleteFolder`. Toggle Active.

### Task 8.3: Cutover Cleanup

- [ ] **Step 1: Atualizar webhook URL no Supabase**

No Supabase do Ellenco, em **Database → Webhooks**, achar o webhook que dispara delete em `candidatos_pre_aprovados`.
- URL atual: `https://n8n.ellenco.com.br/webhook/supabase` (ou similar)
- URL nova: `https://n8n.ellenco.com.br/webhook/ellenco/cleanup/folder`

- [ ] **Step 2: Desativar `EXCLUIR PASTA NO DRIVE` antigo**

Toggle Active OFF.

- [ ] **Step 3: Smoke test**

Deletar 1 row de teste em `candidatos_pre_aprovados` (com `id_pasta` apontando pra uma pasta de teste no Drive). Verificar:
- Webhook do `cleanup_post` recebeu (Executions)
- `cleanup/folders` ganhou e perdeu mensagem
- Pasta sumiu do Drive

> **Pasta de teste:** crie uma pasta vazia "teste-cleanup-2026-05" no Drive antes do smoke test. Não use uma pasta de candidato real.

---

## Phase 9: Estimativa de tempo de execução manual

> **Objetivo desta seção:** dar uma referência **realista** pra você responder o gestor sobre prazo. Os números são pra **execução do plano por uma pessoa que conhece n8n** (você), com o ambiente já preparado (acesso ao broker, Chatwoot, Supabase). **Não** inclui descoberta, debate de design ou desenvolvimento de prompts novos — esse trabalho está absorvido neste plano.

### Breakdown por phase

| Phase | Conteúdo | Estimativa pessimista | Otimista | Notas |
|---|---|---|---|---|
| 0 | Pré-requisitos (queues, exchange, credenciais, schema check) | 2h | 1h | Phase 0 está praticamente feita; resta criar as 3 queues novas (Task 0.2) |
| 1 | Producer `ellenco/rabbit_post` | 2h | 1.5h | 18 campos do `initData2` é o mais lento |
| 2 | Consumer `ellenco/post_messages` | 4h | 3h | 16 nodes, branches, integração Chatwoot |
| 3 | Agente `Ellenco - Ambiente de Produção` | 4h | 3h | Debounce + memory + agente são detalhados |
| 4 | DLQ `ellenco/error_messages` | 1h | 30min | Cópia direta do template |
| 5 | Cutover Fluxo 3 | 1h | 30min | + janela de monitoramento de 24h (não conta como esforço ativo) |
| 6 | Forms ecosystem (3 workflows: producer + consumer + agente classifier) | 6h | 4h | Consumer Forms é denso (Sheets + Chatwoot + Drive + Postgres + branches) |
| 7 | Documentos (modificar producer + consumer novo) | 4h | 3h | Replicar lógica IA do 3.1 atual + adaptar trigger |
| 8 | Cleanup (producer + consumer) | 1h | 30min | Trivial |
| **Total esforço ativo** | | **25h** | **17h** | |

### Tradução em dias úteis

| Cenário | Hipótese | Dias úteis |
|---|---|---|
| Otimista | Você dedica 6h/dia full focus, sem interrupções, ambiente cooperando | **~3 dias** (17h ÷ 6h) |
| Realista | Você dedica 4h/dia (resto vai pra reuniões/outras tarefas) e tem ~30% de retrabalho/debug | **~7-8 dias** úteis (~1.5 sprint) |
| Pessimista | 3h/dia + ambientes oscilando (Chatwoot caindo, Cloud API negando, etc) | **~12 dias** úteis (~2.5 semanas) |

### Recomendação de ordem real de execução

> **A ordem das Phases no plano é didática (mais fácil → mais difícil), não cronológica.** Pra cutover real, a ordem ótima é:

1. **Phase 0** (1 dia) — destrava tudo
2. **Phases 1-4** (3-5 dias) — Fluxo 3 inteiro pronto, ainda inativo
3. **Phase 5** (cutover Fluxo 3) — vira WhatsApp inbound. **Marco crítico:** Evolution sai de jogo aqui.
4. **Phase 7** (Documentos) — só pode rodar depois que producer principal estiver vivo. ~2-3 dias.
5. **Phase 6** (Forms) — pode começar a preparar em paralelo com Phase 7. Cutover quando Phase 7 estiver estável. ~2-3 dias preparo + 1 dia cutover.
6. **Phase 8** (Cleanup) — última, ~0.5 dia.

**Total realista:** ~8-12 dias úteis distribuídos. Se rodar 100% focado, encurta pra ~4-5 dias. **Resposta sugerida pro gestor:** entrega em **2-3 semanas** com folga pra debug e estabilização.

### Riscos que podem aumentar prazo

- **Cloud API approval/rate limits:** se a inbox WhatsApp do Chatwoot ainda não estiver totalmente aprovada (template messages, business verification), enviar mensagem proativa (Phase 6 — primeira mensagem ao candidato pré-aprovado) pode esbarrar em janela de 24h. **Mitigação:** começar smoke tests cedo.
- **Schema Supabase incompleto:** se o `documents` ou `candidatos_pre_aprovados` não tiverem colunas que o Fluxo 2 atual escreve, descobrir só na Phase 6/7. **Mitigação:** Task 0.5 já checa isso parcialmente — expandir pra Phase 6/7 antes de começar.
- **Prompt do classifier divergente:** se o Fluxo 2 atual tem prompt complexo que requer ajuste pra trabalhar com output estruturado, gasta 1-2h iterando. Já contado nos 4h da Phase 6.
- **Convert.ellenco.com.br fora do ar:** quebra Phase 7 branch PDF. **Mitigação:** verificar saúde do serviço na Phase 0, ter fallback documentado (Gemini multi-modal direto?).
- **Volume de retrabalho de UI:** clicar 200+ vezes em forms n8n cansa e gera erros. **Mitigação:** usar atalhos n8n (duplicar nodes, copiar configs entre workflows) — pode acelerar 30%.

---

## Apêndice A: Credenciais necessárias (uso por phase)

| Credencial (nome no n8n Ellenco) | Usada em | Phases |
|---|---|---|
| `Credencial - RabbitMQ` | Producer / Consumer / DLQ — todos os workflows que publicam ou consomem da exchange | 0, 1, 2, 4, 6, 7, 8 |
| `Chatwoot - Api Access Token` (criar — Task 0.4) | createMessage, createConversation, createContact, toggle_status, get conversation | 2, 6, 7 |
| `Postgres - Ellenco` (renomear de `Supabase - Ellenco` se necessário) | Lookup, insert, update em `candidatos_pre_aprovados`, `documents`, memory `conversations` | 0, 2, 3, 6, 7 |
| `Redis - Ellenco` (ou nome existente) | Lock cooldown rejeição, debounce queue | 2, 3, 6 |
| `OpenAi - Ellenco` | Agente WhatsApp + Forms Classifier + Vision (imagens) | 3, 6, 7 |
| `Credencial Gemini - Ellenco` | Análise de PDF (Phase 7) | 7 |
| `Credencial - Google Sheets` | Buscar vagas + escrever candidatos/reprovados/standby | 6 |
| `Credemcial - Google Drive` (sic, mantém typo se existir hoje) | Criar pasta candidato, upload doc, rename, delete folder | 6, 7, 8 |

> Os nomes "Credemcial" (typo) são os reais do Ellenco hoje. Renomear é opcional — se renomear, atualizar **todas** as referências nos workflows novos antes de ativar.

## Apêndice B: Estrutura final de arquivos no repo

```
clients/Ellenco/
├── original/                  ← workflows atuais (preservar)
│   ├── 1. PRODUCER - RABBITMQ.json
│   ├── 2. PROCESSA RESPOSTAS DO FORMS.json
│   ├── 3. AGENTE IA - PRINCIPAL.json
│   ├── 3.1 SALVAR DOCUMENTOS DOS CANDIDATOS.json
│   └── EXCLUIR PASTA NO DRIVE.json
├── adapted/                   ← workflows novos exportados após cada Phase
│   ├── ellenco-rabbit_post.json              (Phase 1, atualizado em Phase 7.1)
│   ├── ellenco-post_messages.json            (Phase 2)
│   ├── Ellenco - Ambiente de Produção.json   (Phase 3)
│   ├── ellenco-error_messages.json           (Phase 4)
│   ├── ellenco-forms_post.json               (Phase 6.1)
│   ├── Ellenco - Forms Classifier.json       (Phase 6.2)
│   ├── ellenco-forms_processor.json          (Phase 6.3)
│   ├── ellenco-documents_consumer.json       (Phase 7.2)
│   ├── ellenco-cleanup_post.json             (Phase 8.1)
│   └── ellenco-cleanup_consumer.json         (Phase 8.2)
└── plans/
    └── 2026-05-05-fluxo-3-template-pure.md  ← este arquivo
```

> **Após cada Phase ficar pronta**, exporte o JSON do workflow do n8n e salve em `adapted/`. Não altere os JSONs em `original/`.

## Apêndice C: Fora de escopo deste plano

Tudo abaixo fica pra plano(s) seguinte(s):
- **Transcrição de áudio** no agente WhatsApp (Phase 3 — rota `audio` do `messageType` switch). Implementar quando o produto precisar.
- **Ambiente de Testes Ellenco** (chat trigger nativo do n8n, gêmeo do `Ambiente de Testes - N8N` do Pure Pilates): criar depois pra iterar prompts sem mexer em produção.
- **Refatoração callback assíncrono** (consumer recebe callback via webhook em vez de bloquear no `callAgent`): considerar só se latência > 30s.
- **Multi-vagas por candidato:** schema atual não suporta (cada phone aparece em 1 row). Se a Ellenco quiser que o mesmo candidato se candidate a múltiplas vagas, precisa de uma tabela `candidaturas` separada — proposta de DDL fica fora deste plano.
- **Limpeza da queue legada `filas-forms-ellenco`:** após Phase 6 estável, deletar a queue. Plano de cleanup separado (junto com Task 5.5 e equivalentes).
- **Dashboards de observabilidade** (RabbitMQ usage, taxa de erro DLQ, latência por phase): nice to have, não bloqueia.
- **Migração das `conclusão`/`conclusao` rows com encoding inconsistente:** descobrir + DDL é trabalho separado.

## Apêndice D: Convenções de nomes

| Item | Convenção | Exemplos |
|---|---|---|
| Workflows (consumer/producer/DLQ) | `ellenco/<nome>` (slash) | `ellenco/rabbit_post`, `ellenco/post_messages`, `ellenco/forms_post`, `ellenco/forms_processor`, `ellenco/error_messages`, `ellenco/documents_consumer`, `ellenco/cleanup_post`, `ellenco/cleanup_consumer` |
| Workflows agente (env produção) | `Ellenco - <Nome>` (espaço hífen espaço, igual template Pure) | `Ellenco - Ambiente de Produção`, `Ellenco - Forms Classifier` |
| Webhook paths | `ellenco/<categoria>/<acao>` | `ellenco/oficial/messages`, `ellenco/oficial/rabbit`, `ellenco/forms`, `ellenco/agents/production`, `ellenco/agents/forms`, `ellenco/cleanup/folder` |
| Queues | `<acao>/<entidade>` (igual template) | `post/messages`, `error/messages`, `forms/messages`, `documents/messages`, `cleanup/folders` |
| Routing keys | igual nome da queue | (mesmas acima) |
| Credenciais | `<Tipo> - Ellenco` (preferencial) | `Postgres - Ellenco`, `Chatwoot - Api Access Token` |
| Tags n8n | `ellenco` em TODOS os 9 workflows novos | |
| Chaves Redis | `<contexto>:<chave>` — formato bate entre fluxos | `cooldown:5511999999999`, `client:1234:conversation:5678:messagesQueue` |

## Apêndice E: Phone format — JID vs E.164

> **Descoberta na Phase 2 (Task 2.5).** Documentado aqui porque afeta Phase 6 também.

- **Coluna `phone` em `candidatos_pre_aprovados`:** está hoje em formato JID (`5515996823928@s.whatsapp.net`) — herança do Evolution API.
- **Chatwoot envia:** E.164 (`+5515996823928`).
- **Estratégia adotada:** normalizar nos lookups via `REGEXP_REPLACE(phone, '\D', '', 'g')` na coluna E `.replace(/\D/g, '')` no parâmetro JS.
- **Decisão:** **não** fazer migração de dados (UPDATE em todas as 954 rows trocando JID por E.164). Razões:
  1. Escrita nova (Phase 6 insertCandidato) já usa E.164 — base vai se "auto-corrigindo" com candidatos novos.
  2. Migração risk-free é fácil (`UPDATE candidatos_pre_aprovados SET phone = '+' || REGEXP_REPLACE(phone, '\D', '', 'g')`) mas pode bater em algum trigger ou view não documentada.
  3. Lookups dos consumers já normalizam — não há funcional impedindo de manter como está.
- **Se quiserem migrar mais tarde:** rodar a query acima fora do horário comercial, fazer backup antes, e remover a normalização dos lookups (otimização de query).
