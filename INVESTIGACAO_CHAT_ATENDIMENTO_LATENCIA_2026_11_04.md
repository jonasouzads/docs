# Chat de atendimento: latência, lentidão percebida e usabilidade — investigação (2026-11-04)

Investigação somente leitura. Nenhum código foi alterado, nenhuma DDL, nenhum push e
nenhum acesso ao banco de produção. Fontes:

* **Axiom**, dataset `wizebot`, `env == "production"`. Para rota HTTP, também
  `railwayService == "Api"`. Janela: **14 dias completos em BRT, de 21/10 00:00 a
  04/11 00:00** (21/10 03:00Z → 04/11 03:00Z).
* **PostHog**, projeto 299809, mesma janela, `$pathname = '/chat'`.
* **Código** em produção: frontend `415db87` e backend `9cec9290`, os dois no
  `origin/main` de 04/11.

> **Divergência do pedido.** O filtro `railwayService != "unknown"` não filtra nada,
> porque esse valor não existe no dataset. Usei `railwayService == "Api"`, que separa
> a API dos 7 workers. A janela não cruza a virada de telemetria de 13/08. O Axiom
> guarda dados a partir de 05/10 21:47Z.
>
> **Sobre a precisão.** Os percentis do Axiom são aproximados por histograma. Por
> isso aparecem valores "quebrados" como 399,68 ms, que aqui vão arredondados. Todos
> os rankings usam o tempo total (volume × latência), nunca a média.

---

## Resumo

1. **`POST …/mark-read` virou a rota que mais consome tempo da API: 31,6 mil s em
   14 dias.** O p50 é **400 ms**, constante desde que o dado existe (05/10) e igual
   em todas as réplicas. O volume tem causa confirmada no código: com a conversa
   aberta, cada mensagem recebida dispara um `mark-read`, sem agrupar. A latência
   **não tem causa confirmada**. A rota não chama a Meta, e as hipóteses estão
   no §3.1.
2. **Trocar de conversa dispara de 9 a 10 requisições.** Seis delas vêm do painel
   de detalhes do contato, que fica montado e escondido mesmo quando está fechado.
   Juntas, essas seis somam **≈ 28 mil s em 14 dias** (§3.2). As contagens do Axiom
   batem com isso: `media/stats`, `contacts/:id/tags`, `metadata`, `agents` e
   `session-status` têm entre 99 mil e 122 mil chamadas cada, o mesmo patamar de
   `messages-optimized`.
3. **Os contadores (`filter-counts` e `unread-count`) seguem como 2º e 4º
   consumidores**, com 45 mil s somados. Saem a cerca de **1 GET por minuto por aba
   ativa**, por uma invalidação temporizada de 60 s. A materialização dos contadores
   (F1, em produção desde 08/09) **continua em sombra**: nenhum leitor usa a bandeira.
4. **Experiência medida no PostHog:**
   * No desktop, o **INP p75 é 280 ms** e o p95 é 592 ms, ambos na faixa "precisa
     melhorar". Houve 1.033 interações acima de 500 ms.
   * As interações mais lentas são **cliques em linhas da lista de conversas**, com
     p75 entre 270 e 350 ms (160–200 ms de JS mais ~110 ms de apresentação). É a
     parte da troca de conversa que roda na thread principal.
   * O **CLS p75 está na faixa ruim nos dois aparelhos**: 0,25 no desktop e 0,43 no
     mobile. Não há atribuição do elemento que se desloca.
5. **A queda ao responder uma mensagem interativa (React #31) continua em
   produção.** Desde 06/10 foram 51 ocorrências, e a última foi hoje (04/11). O
   botão "Tentar novamente" relança o erro, e é ele que recebe rage click no mobile.
6. **Telemetria cega em três pontos centrais:** o tempo até a lista ficar utilizável,
   o tempo de troca de conversa (só a parte de JS aparece, pelo INP) e o envio de
   ponta a ponta, do clique ao tique. O chat não tem nenhum evento próprio no
   PostHog, e a lista e o thread estão marcados com `ph-no-capture`.

O §7 propõe a ordem dos pacotes, do maior ganho por esforço ao menor.

---

## 1. Fase zero — inventário de telemetria

### 1.1 Backend (Axiom)

* **`http.request`** sai do `HttpRequestLoggerMiddleware`
  (`src/common/middleware/http-request-logger.middleware.ts`) para todas as rotas,
  exceto `/health`, assets e docs.
  * Vai por `emitTelemetry`, então chega ao Axiom mesmo com o `LOG_LEVEL=warn`.
  * Campos: `path` normalizado **sem query string**, `duration` (ms; o campo **não** é
    `durationMs`), `statusCode`, `tenantId`, `userId` (populado em ~38% das linhas
    numa amostra de 30 min), `userAgent` e `railwayReplica`.
  * As rotas do chat não são amostradas.
* **`perf.slow_*`** (`trackPerformance`) só aparece **acima do limiar**: 500 ms para
  query, 10 s para chamada externa, 120 ms para os contadores `inbox.counts.*`. Não há
  distribuição de duração.
* **`logger.log`/`info` não chega ao Axiom.** Todo `[PERF]`, `message.enqueued`,
  `messaging.window.open` e `[Assignment]` do chat se perde.
* **`job.completed`** traz `duration` e `wait_time_ms` por fila. **`channel.message.sent`**
  traz `durationMs` da chamada à Meta, mas não a origem (chat, campanha ou fluxo).
* **Falhas que aparecem como 2xx:**
  * `send-chat` e `upload-media` respondem 202 com `success:false`;
  * `filter-counts` e `unread-count` respondem 200 com zeros;
  * `mark-read`, `metadata` e `session-status` respondem `success:false`.
  * O `statusCode` do `http.request` não enxerga nenhum desses casos.

### 1.2 Frontend (PostHog)

Eventos no `/chat` em 14 dias:

| Evento | Eventos | Pessoas | Sessões |
|---|---:|---:|---:|
| `$web_vitals` | 23.236 | 149 | 4.610 |
| `$pageview` | 8.371 | 156 | 3.536 |
| `$autocapture` | 7.180 | 146 | 1.697 |
| `$pageleave` | 4.020 | 174 | 2.011 |
| `$exception` | 104 | 15 | 54 |
| `$rageclick` | 39 | 22 | 33 |
| `app_version_*` | 155 | — | — |

* **Não há nenhum `$dead_click`, nenhum evento de produto do chat** (conversa aberta,
  mensagem enviada, resolvida) **e nenhum `performance.mark`**. O registro fechado
  `PRODUCT_EVENTS` (`lib/analytics/identity.ts`) não tem eventos do chat.
* **A lista e o thread usam `ph-no-capture`** (`ChatList.tsx:490`,
  `MessageArea.tsx:699`). Por isso o autocapture de clique é quase nulo nessas áreas,
  e as gravações escondem esse trecho.
* **Web vitals:**
  * INP tem atribuição: `inputDelay`, `processingDuration`, `presentationDelay` e o
    alvo do clique.
  * **CLS não tem atribuição**: o campo `largestShiftTarget` vem vazio em 100% das
    amostras.
* **Gravações:** existem. A seleção do §4.5 usou os metadados (`session_replay_events`).

### 1.3 Mensurabilidade por superfície

| Superfície | Estado | O que existe / o que falta |
|---|---|---|
| Carga inicial (contadores, lista, filtros, abas) | **Parcial** | Duração de cada rota (Axiom) e FCP/LCP da página. Falta o tempo até a lista ficar utilizável e a cascata real no cliente. |
| Abertura e troca de conversa | **Parcial** | Duração de cada rota e INP do clique na linha (só JS). Falta o tempo do clique até o histórico renderizado. |
| Envio pelo compositor | **Parcial** | `send-chat` (HTTP), `job.completed` da fila `msg-realtime`, `channel.message.sent.durationMs`. Falta ponta a ponta, origem do envio, publicação no Pusher e abandono de envio. |
| Tempo real | **Cega** | Não há evento de mensagem recebida por tenant, nem latência de publicação no Pusher, nem medida do refetch causado por evento. |
| Mobile | **Parcial** | Web vitals e pageviews por `$device_type`. Falta atribuição do CLS e eventos de navegação lista↔conversa. |

---

## 2. Fase um — latência de backend (Axiom, 14 dias)

### 2.1 Rotas usadas pelo chat, ranqueadas por tempo total

`statusCode` incluído (todas as respostas). A coluna "Sup." indica a superfície:
**C** = carga, **T** = troca de conversa, **E** = envio, **R** = tempo real/contadores.

| # | Rota | Sup. | Volume | p50 ms | p95 ms | p99 ms | Tempo total (s) | 429 |
|---:|---|---|---:|---:|---:|---:|---:|---:|
| 1 | `POST whatsapp/conversations/:id/mark-read` | T/R | 92.319 | 400 | 560 | 606 | **31.615** | 169 |
| 2 | `GET conversations-optimized/filter-counts` | C/R | 229.126 | 63 | 280 | 957 | **24.246** | 0 |
| 3 | `GET conversations-optimized/unread-count` | C/R | 351.994 | 34 | 152 | 451 | **20.809** | 0 |
| 4 | `GET conversations-optimized` (lista) | C/R | 147.126 | 74 | 216 | 528 | **14.455** | 1.491 |
| 5 | `POST messages/upload-media` | E | 8.664 | 507 | 3.073 | 10.493 | 9.739 | 0 |
| 6 | `GET contacts/:id/media/stats` | T | 98.896 | 56 | 208 | 538 | 9.489 | 502 |
| 7 | `GET …/:id/messages-optimized` | T | 140.539 | 57 | 106 | 149 | 8.643 | 294 |
| 8 | `GET crm/assignees` | C/T | 150.217 | 27 | 185 | 377 | 8.413 | 1.045 |
| 9 | `GET tags` | C | 159.009 | 31 | 100 | 185 | 7.059 | 2.189 |
| 10 | `GET whatsapp/conversations/:id/metadata` | T | 111.467 | 37 | 77 | 115 | 4.665 | 697 |
| 11 | `POST pusher/auth` | R | 253.346 | 15 | 39 | 56 | 4.664 | 0 |
| 12 | `GET whatsapp/conversations/:id/notes` | T | 98.551 | 35 | 69 | 104 | 3.943 | 502 |
| 13 | `GET agents` | T | 98.931 | 22 | 137 | 291 | 3.926 | 347 |
| 14 | `GET contacts/:id/tags` | T | 105.858 | 29 | 59 | 85 | 3.499 | 0 |
| 15 | `GET audit-logs/chat-timeline` | T | 98.539 | 29 | 63 | 96 | 3.287 | 506 |
| 16 | `GET messages/session-status/:id` | T | 122.317 | 19 | 42 | 67 | 2.783 | 356 |
| 17 | `POST messages/send-chat` | E | 48.824 | 49 | 79 | 174 | 2.746 | 0 |
| 18 | `POST chat/typing` | E | 227.287 | 8 | 19 | 29 | 2.294 | 0 |
| 19 | `POST whatsapp/media/resolved-urls` | T | 56.388 | 27 | 61 | 100 | 1.841 | 0 |
| 20 | `POST …/:id/resolve` | — | 5.032 | 158 | 498 | 560 | 1.249 | 0 |
| 21 | `POST ai/rewrite` | E | 1.110 | 1.143 | 2.069 | 2.623 | 1.232 | 0 |
| 22 | `POST messages/react` | E | 1.210 | **995** | 1.391 | 1.599 | 1.228 | 0 |
| 23 | `POST media/upload` (cabeçalho de modelo) | E | 306 | 738 | **25.079** | 32.473 | 1.198 | 0 |
| 24 | `POST templates/:id/send` | E | 608 | 1.477 | 2.196 | 2.522 | 932 | 0 |
| 25 | `POST …/:id/mark-unread` | — | 3.177 | 177 | 507 | 571 | 896 | 0 |

Fora do chat, as maiores rotas da API no mesmo período são `whatsapp/webhook` (20.855 s,
2,7 M chamadas, p95 17 ms: é volume, não lentidão) e `webhook-triggers` (3.464 s).

**Mudanças desde a medição de setembro** (25/08–08/09, memória do projeto):

* `mark-read` não aparecia no top 7 e agora é o 1º.
* Os 429 da listagem caíram de **110.630 para 1.491**, efeito das correções de
  setembro e outubro.
* O p95 do `filter-counts` caiu de 392 para 280 ms.

### 2.2 Concentração por tenant

Os tenants vão pelo prefixo de 8 caracteres.

* **`15cd8418`** é o maior em quase todas as rotas do chat:
  * `mark-read`: 14.849 chamadas, 5.252 s;
  * `filter-counts`: 24.442 chamadas;
  * lista: 20.565 chamadas;
  * `unread-count`: 27.919 chamadas;
  * `upload-media`: p95 de 8,4 s.
  * Opera **um** número (pelos `channelAccountId` distintos em `channel.message.sent`).
* **`filter-counts` com p95 acima de 1 s** em tenants específicos: `f30dd097` (2,15 s),
  `11788d25` (1,42 s) e `fe354a62` (0,82 s). Os mais caros entre as amostras lentas
  são `inbox.counts.open` (p95 1,84 s) e `inbox.counts.unread` (p95 1,31 s).
* **`media/stats`** tem p95 de **3,75 s** em `2b774cba`.
* **Tenants com vários números:**
  * `3f6e646a` (9 contas), `c001cdfa` e `10d67771` (6 cada), `c674bd0a` e `b69bf8e7`
    (3 cada).
  * Desses, só `c001cdfa` aparece entre os grandes consumidores (`unread-count`,
    929 s).
  * **Ter vários números não é, hoje, o que concentra o custo do chat.** O custo vem de
    volume de conversas por número.

### 2.3 Padrões de chamada por usuário (7 dias, por usuário e hora)

| Rota | p50/h | p90/h | p99/h | máx/h |
|---|---:|---:|---:|---:|
| `unread-count` | 16 | 46 | 87 | 328 |
| `filter-counts` | 12 | 43 | 77 | 168 |
| lista | 5 | 28 | 113 | **1.517** |
| `tags` | 6 | 27 | 80 | **950** (2.445 na janela de 14 d) |
| `messages-optimized` | 7 | 36 | 106 | 330 |

* **Contadores por tempo:** um usuário típico, observado das 08h às 18h BRT de 03/11,
  fez 1 `unread-count` e 1 `filter-counts` por minuto ativo, no máximo 2. É uma
  cadência fixa, não reação a evento.
* **Laço de um único `userId`:** `241bd083` atua em **12 tenants** e sozinho gerou:
  * **30.483 `/tags`** (19% de todo o `/tags`);
  * **17.511 listagens** (12% do total).
  * Houve horas com **2.445 `/tags` em 60 minutos seguidos**, 40 por minuto.
  * Isso é código em laço, não clique. A causa não foi identificada (pergunta Q2).
* **Rajadas de lista em clientes:** `c55e23a9` (1.241/h) e `7034f714` (1.018/h), em
  27–31 minutos ativos da hora.

### 2.4 Envio (trecho assíncrono)

| Etapa | p50 | p95 |
|---|---:|---:|
| `send-chat` (HTTP, até enfileirar) | 49 ms | 79 ms |
| espera na fila `msg-realtime` | 1 ms | 7 ms |
| job `msg-realtime` completo | 1,12 s | 1,60 s |
| dentro do job: chamada à Meta (`channel.message.sent.durationMs`, todas as filas) | 0,80 s | 1,21 s |

* O tique de "enviado" visto pelo atendente depende da resposta HTTP. A troca pela
  mensagem real depende do Pusher depois do job, ≈ 1,1–1,6 s depois do clique, mais a
  publicação no Pusher, que não é medida.
* Para comparação, a fila de massa `msg-bulk` tem espera p95 de **8 minutos**. É a fila
  de campanha e não afeta o chat.

### 2.5 Outros sinais de backend

* `whatsapp-webhook:media-download-primary`: p50 de **7,4 s**, p95 de 20 s em 492
  amostras lentas. Esse é o tempo para mídia recebida aparecer no chat, nas amostras
  acima do limiar.
* `realtime.payload.trimmed`: ~2.100 por dia, mas o aparo é pequeno (~1.076 → 906
  bytes, etapa `strip`, áudio e modelo). É benigno.

---

## 3. Fase três — código que explica as medições

Legenda. **Evidência:** **M** = medido (cita a seção). **C** = confirmado lendo o
código. **H** = hipótese, sem evidência. **Severidade:** Alta / Média / Baixa.

### 3.1 `mark-read`: volume explicado, latência sem causa confirmada — Alta (custo de banco), T/R

* **Volume (M §2.1, C):** com a conversa aberta, cada mensagem recebida agenda um
  `markAsRead` 300 ms depois, **um por mensagem, sem agrupar**
  (`frontend hooks/useChatRealtime.ts:385-387`). Abrir conversa com não lidas soma
  mais um (`conversation-store.ts:638`).
* **Corpo da rota (C):** `verifyConversationAccess` (Redis), `messages.findMany`,
  `messages.updateMany`, `conversations.update`. Com não lidas, também um `await` no
  Pusher (`backend whatsapp-conversation-actions.service.ts:295-353`).
  * Não chama a Meta.
  * O `findMany` serve só para contar e listar ids, e repete o filtro do `updateMany`.
* **Latência de 400 ms constante (M §2.1, H):**
  * O `mark-unread`, que também faz `await` no Pusher, tem p50 de 177 ms. Portanto o
    Pusher sozinho não explica o piso.
  * A diferença que sobra está no par `findMany` + `updateMany` em `messages`
    (`conversation_id`, `direction='inbound'`, `read_at IS NULL`), ou num trigger
    aplicado no banco fora das migrations versionadas (não há nenhum nas migrations).
  * **Não confirmado.** Confirmar exige `EXPLAIN` no banco (pergunta Q1).
* **Defeito relacionado (C):** o `del` de `unread:count:${tenantId}:all`
  (`actions.service.ts:334` e `whatsapp-conversation.service.ts:50`) mira uma chave
  que não existe. A chave real é `unread:count:<t>:<escopo>`
  (`conversations-optimized.service.ts:495`). O badge depende só do TTL de 60 s e do
  ping.
* O atendente **não espera** essa rota, porque `markAsRead` é otimista. O impacto é
  ocupação de conexão e pool, não latência vista na tela.

### 3.2 Painel de detalhes montado e escondido busca dados em toda troca — Alta, T

* `ContactDetails` fecha com `!isOpen && 'hidden'` (`ContactDetails.tsx:74`), sem
  desmontar (C). A cada troca de conversa, com o painel **fechado**, saem (C):
  * `contacts/:id/tags`;
  * `contacts/:id/media/stats`;
  * `conversations/:id/metadata`;
  * `agents?status=active`;
  * `crm/assignees`, por fetch direto. Duplica a da sidebar, que passa pelo React
    Query.
  * `session-status`.
* **Medição (M §2.1):** essas rotas somam ≈ 9.489 + 3.499 + 4.665 + 3.926 + 2.783 s,
  mais parte dos 8.413 s de `crm/assignees`. Total de ≈ 28 mil s em 14 dias, quase o
  mesmo que o `mark-read`.
  * O volume de cada uma (99 mil–122 mil) acompanha o de `messages-optimized`
    (140 mil, que inclui paginação).
  * Também respondem por parte dos 429 (502 em `media/stats`, 697 em `metadata`).
* No `MessageArea`, `chat-timeline` (limit 200) e `notes` são buscados a cada troca. A
  guarda guarda só o último id, então ir de A para B e voltar a A busca de novo (C).
* O compositor monta o `TemplateSendModal` sempre, e isso dispara
  `templates?limit=1000` e `templates/stats` na primeira montagem
  (`useTemplates.ts:406-413`) (C).

### 3.3 Contadores por relógio, materialização em sombra — Alta, C/R

* **`filter-counts` (C):** qualquer mudança de referência do dicionário de conversas
  (toda `new-message`) agenda uma invalidação no fim de uma janela de 60 s
  (`useInboxFilterCounts.ts:58-75`).
  * Com atividade contínua, dá ~1 GET por minuto por aba. É o padrão medido no §2.3.
  * No backend são 6 consultas em `Promise.all`, uma com `JOIN` e `COUNT(DISTINCT)`.
    Cada miss ocupa até 6 conexões do pool, de 20 por processo (H).
* **`unread-count` (C):** throttle de 60 s no ping `unread-count-changed`, mais
  reconciliação a cada 5 min e no foco.
* **Bandeira de contadores materializados (C):** `countersFlag` é calculada em
  `getUserContext`, e **nenhum leitor a consome** (`inbox-counters-flag.ts:20`). A
  tabela `inbox_counters` é mantida pelo reconciliador e não é lida.
* **Sidebar no mobile (C):** fica com `hidden md:flex` mas montada, e faz
  `filter-counts` e `assignees` mesmo invisível.
* `refetchOnWindowFocus` fica no padrão (true), com staleTime de 60 s. Voltar à aba
  refaz contadores, tags e assignees (H, comportamento padrão da biblioteca).

### 3.4 Clique na linha da lista custa ~300 ms de thread principal — Alta, T

* **(M §4.2)** O INP das linhas tem p75 de 270–350 ms, dos quais 160–200 ms são
  `processingDuration`.
* **(C)**
  * A lista não é virtualizada (`ChatList.tsx:532`).
  * `useVisibleChats` filtra e ordena **o dicionário inteiro**, que só cresce, e tem
    `selectedChatId` nas dependências (`selectors.ts:86,104`). Assim, toda seleção
    reordena tudo.
  * O histórico também não é virtualizado: `groupedMessages` re-agrupa mensagens,
    logs e notas a cada mensagem nova (`MessageArea.tsx:799-824`).
* **(C)** O layout forçado por tecla (`useMessageInput.ts:428-429`: `height='auto'`
  seguido de leitura de `scrollHeight`) continua. Ele combina com o `inputDelay` de
  ~170 ms medido em teclas no `body` e no campo de busca (M §4.2).
* A decisão de virtualizar (P4/P6 da investigação de outubro) segue pendente.

### 3.5 `fetchSingleChat` recarrega a página 1 inteira — Média, R

* **(C)** Não existe rota de leitura de uma conversa. `new-message` de conversa
  desconhecida, atribuição a mim de conversa ausente e **toda**
  `new-conversation-queued` disparam `GET` da lista com 50 conversas
  (`useChatRealtime.ts:350,570,775`), com coalescência de 500 ms.
* `conversation-updated` com `reopened`, `unpinned` ou `metadata-updated` faz
  `silentRefreshChats`, que é a página 1 com o filtro ativo (`:659,695,705`).
* **(H)** Esse mesmo evento acabou de subir a versão da lista no Redis, e a chave de
  cache inclui o `userId`. Então cada aba de cada atendente roda a consulta completa ao
  mesmo tempo.
* **(M parcial)** 1.491 respostas 429 na lista em 14 dias. Não dá para separar no
  servidor a parte causada por evento, porque o `http.request` não tem query string
  nem marca de origem.

### 3.6 Laço de `/tags` e da lista num usuário multiempresa — Média, C

* **(M §2.3)** Há 40 `/tags` por minuto sustentados por uma hora, além de rajadas de
  lista.
* **(H)** A causa não foi encontrada no código. Candidatos:
  * a cascata `contact:tags-updated` → `contact:tags-changed` → invalidação de
    `['tags']` (`useChatRealtime.ts:830`), sob automação que mexe em etiquetas em
    massa;
  * várias abas abertas em empresas diferentes.
* Ver Q2.

### 3.7 Queda ao responder mensagem interativa (React #31) — Alta, T/E

* **(M §4.3, C)** Em `handleReply`, `content = message.content.body`
  (`MessageArea.tsx:370-371`). Na mensagem interativa, `body` é `{text}`. O
  `ReplyPreview` devolve `content` cru no `default` (`ReplyPreview.tsx:16-21`).
* O `replyingTo` fica no store, e por isso o "Tentar novamente" do
  `ChatErrorBoundary` relança o erro. Só recarregar a página sai do estado.
* Diagnosticado em 02/10 e não corrigido.

### 3.8 Resolver pelo menu de contexto chama a API duas vezes — Média, —

* **(C)** `ConversationContextMenu.executeAction` chama `markAsResolved` ou
  `reopenConversation` (`:151-156`). Depois `onAction` chama `store.resolveChat` ou
  `reopenChat` (`ChatList.tsx:331-336`), que chamam a API de novo
  (`conversation-store.ts:897-910`).
* **(C)** No backend, a resolução pelo painel regrava sem trava
  (`actions.service.ts:969-980`). Depois disso:
  * `triggerClosingFlow` enfileira um `execute-flow` sem `jobId`. O `startExecution`
    tem `findActiveExecution` e advisory lock, então o pós-atendimento em dobro é
    **improvável** (H, não provado sob corrida).
  * O **resumo por IA** (`generateSummaryOnResolve` + `materializeSummaryNote`) e o
    gatilho de aprendizado contínuo rodam **duas vezes** (H no efeito: possível nota
    de resumo duplicada e custo de LLM em dobro).
* O resolver pelo cabeçalho engole o erro: `resolveChat` captura e não relança, e o
  toast de erro de `ChatHeader.tsx:66-71` nunca aparece (C).

### 3.9 Corrida na carga inicial: o filtro do atendente pode ser descartado — Média, C

* **(C)** `loadChats` retorna cedo com `if (isLoading) return;`, e o `forceReload` não
  fura essa trava (`conversation-store.ts:384`).
* O filtro padrão por perfil (`'mine'` para atendente) é aplicado quando as permissões
  chegam (`useChatUrlSync.ts:120-131`). Se a carga inicial ainda estiver em voo, a
  carga de "Minhas" é descartada.
* A resposta antiga grava `nextCursor` e `hasMoreChats` sem conferir o filtro, porque
  não há guarda de sequência. A restauração de filtros salvos (`ChatList.tsx:163-177`)
  tem a mesma exposição.
* **(H)** No efeito, "Minhas" sai derivada dos primeiros 50 abertos e pagina com o
  cursor de outro filtro. Não há medição que confirme.
* A busca (`searchChats`) também não tem abort nem sequência. Resposta fora de ordem
  sobrescreve o resultado (C no código, H no efeito).
* O Pusher só conecta depois da lista (`useChatEffects.ts:45-49`), uma cascata
  sequencial (C).

### 3.10 Custo fixo duplicado por requisição — Baixa a Média, todas

* **(C)** `JwtAuthGuard` é global (`app.module.ts:275-277`) e também está declarado na
  classe dos controllers do chat (por exemplo `whatsapp-conversations.controller.ts:31`).
  O guard não verifica se `req.user` já existe, então a verificação de JWT, blacklist
  e cache roda duas vezes. **O impacto não foi medido**: o Redis do caminho quente não
  tem telemetria.
* **(C)** `send-chat` roda `validateAccess` duas vezes (`messaging.controller.ts:91` e
  `messaging.service.ts:153`). Cada execução faz um `findFirst` com 3 includes e um
  SELECT da janela.
* **(C)** A lista (`getConversationsOptimized`) faz 3 `await` independentes em série
  (pins, sequências, campanhas: `service :1170,1189,1214`). Com resultado vazio, ainda
  faz `count` e `distinct` do tenant inteiro só para um `logger.debug`
  (`:1145-1152`).

### 3.11 Chamadas síncronas à Meta dentro do request — Média, E

* `messages/react` tem p50 de 995 ms (M), por causa de `sendAndReport` síncrono
  (`message-reaction.service.ts:165`) (C). A reação é otimista no cliente, mas a rota
  fica presa ~1 s.
* `block`/`unblock` também (`syncBlockWithMeta`) (C).
* `templates/:id/send` tem p50 de 1,48 s (M).
* `media/upload` do cabeçalho de modelo tem p95 de 25 s (M).
* `upload-media` sobe ao R2 de forma síncrona e sem medição (C), com p95 de 3,1 s e p99
  de 10,5 s (M).

### 3.12 Estados de carregamento, erro e vazio

* **Existem e funcionam (C):** esqueleto, erro com "Tentar novamente" e vazio na lista;
  spinner, erro inicial, erro de paginação e vazio nas mensagens.
* **Enganosos ou ausentes (C):**
  * `filter-counts` com erro mostra **zeros**;
  * notas e logs com erro mostram timeline vazia;
  * AssigneeSection com erro aplica `metadata(null)`;
  * áudio que falha não mostra toast;
  * upload não tem progresso;
  * resolver ou reabrir não confirma sucesso.

### 3.13 Mobile — Média

* **(M §4.4)** O CLS p75 é 0,43, na faixa ruim. É 19% dos pageviews do `/chat` (1.634)
  e 40 pessoas.
* **(C)** Os pontos de corte são inconsistentes:
  * `md` (768) para alternar lista e conversa;
  * `sm` (640) para o painel de detalhes sobreposto;
  * `innerWidth<640` para a rolagem.
  * Entre 640 e 767 px, o painel de 340 px divide a tela com a conversa (H no visual).
* **(C)** O "voltar" do cabeçalho mostra a lista, mas **mantém `selectedChatId`**.
  Mensagem que chega nessa conversa enquanto o atendente olha a lista é **marcada como
  lida sozinha**, sem badge e sem som (`ChatLayout.tsx:50-52`,
  `useChatRealtime.ts:298,385-387`).
* **(C)** A URL só usa `replaceState`. O botão voltar do sistema sai do `/chat`.
* **(C)** Conversa que deixa de casar com o filtro zera a seleção, mas a lista não volta
  a aparecer. O atendente cai no `ChatWelcome`, que não tem botão de voltar.

---

## 4. Fase dois — experiência real (PostHog, 14 dias, `/chat`)

### 4.1 Web vitals por aparelho

| Aparelho | INP n | INP p75 | INP p95 | INP > 500 ms | LCP p75 | FCP p75 | CLS p75 |
|---|---:|---:|---:|---:|---:|---:|---:|
| Desktop | 13.778 | **280 ms** | 592 ms | 1.033 | 2,58 s | 0,60 s | **0,25** |
| Mobile | 1.561 | 208 ms | 456 ms | 65 | 1,83 s | 0,72 s | **0,43** |

* **Tempo até a lista ficar utilizável:** **não mensurável.** Não há marca no cliente.
  FCP e LCP são os únicos proxies, e o LCP do desktop já está em "precisa melhorar".
* **Tempo de troca de conversa:** só a parte síncrona aparece, pelo INP do clique na
  linha (§4.2). A parte de rede e de render do histórico está cega.
* **Abandono de envio:** **não mensurável.** Não há evento de envio.

### 4.2 Interações mais lentas (desktop, alvos com 40+ amostras)

| Alvo do clique (resumo) | n | p75 | Processamento p75 | Apresentação p75 |
|---|---:|---:|---:|---:|
| linha da lista (cabeçalho da linha) | 1.579 | 304 ms | 185 ms | 109 ms |
| linha da lista (nome/hora) | 1.675 | 272 ms | 158 ms | 106 ms |
| linha selecionada (`bg-chat-surface-selected`) | 1.221 | 312 ms | 186 ms | 110 ms |
| prévia da última mensagem na linha | 1.277 | 272 ms | 160 ms | 108 ms |
| linha com etiqueta | 335 | 352 ms | 202 ms | 130 ms |
| tecla no `body` | 74 | 480 ms | 28 ms (**input delay 175 ms**) | 185 ms |
| tecla na busca da lista | 59 | 344 ms | 85 ms (**input delay 169 ms**) | 65 ms |
| tecla no compositor | 866 | 184 ms | 52 ms | 82 ms |

* **Idade da aba:** o p50 do INP sobe de 152 ms (0–15 min) para 200 ms (3 h ou mais). O
  p75 vai de 272 para 304 ms.
* Numa sessão gravada de 4 h (§4.5), o INP da página subiu de forma contínua, de 480 ms
  para 1.848 ms.
* Ressalva: o SDK gira `session_id` e `window_id` quando a aba fica ociosa, então a
  idade real é subestimada.

### 4.3 Erros de frontend no `/chat`

| Erro | Ocorrências | Pessoas |
|---|---:|---:|
| React #31 "object with keys {text}" (§3.7) | 29 (51 desde 06/10; última em 04/11) | 5 |
| `NotAllowedError` em `play()` (som de notificação sem gesto do usuário) | 28 | 1 |
| `Script error.` (origem cruzada, sem detalhe) | 22 | 3 |
| `ChunkLoadError` (deploy novo com aba antiga) | 14 | 3 |
| React #185 (profundidade máxima de atualização: laço de setState) | 4 | 2 |
| React #418 (hidratação) | 2 | 2 |
| `removeChild` em nó que não é filho | 1 | 1 |

O React #185 não tem componente identificado. Aparece em 23/10 e 30/10 e entra como
pergunta Q4.

### 4.4 Rage clicks e diferença entre desktop e mobile

* São 39 rage clicks em 14 dias. Os únicos alvos com repetição:
  * "Todas as conversas" (5, uma pessoa);
  * links sem texto (5, 4 pessoas);
  * **"Tentar novamente" no mobile (4, 2 pessoas)**: o botão do `ChatErrorBoundary`,
    compatível com o laço do §3.7.
* Outros alvos de rage click são nomes de contatos. Não foram reproduzidos aqui.
* O número baixo **não indica ausência de atrito**: lista e thread estão em
  `ph-no-capture`, e o clique nelas não é capturado. Dead clicks não estão sendo
  capturados.
* **Desktop × mobile:** o mobile tem INP melhor (208 contra 280 ms; listas menores na
  tela) e CLS pior (0,43 contra 0,25). O LCP do mobile é menor.

### 4.5 Sessões gravadas mais lentas

Selecionei 12 sessões gravadas de 28/10 a 03/11 com 3 ou mais interações acima de
500 ms. Descrevo cinco pelos metadados da gravação e pela sequência de eventos:

| Sessão (início BRT) | Duração / ativa | Cliques / teclas | O que os eventos mostram |
|---|---|---|---|
| 02/11 18:17 | 4h41 / 78 min | 2 / 0 | 27 de 28 amostras de INP acima de 500 ms. O INP sobe de forma contínua de 480 para 1.848 ms ao longo da noite. Picos de processamento de 800–1.160 ms e de apresentação até 1.250 ms. É uma aba aberta a tarde inteira, ficando mais lenta a cada hora. |
| 28/10 08:03 | 3h35 / 78 min | 1 / 1 | INP pior de 1.184 ms, 8 amostras ruins em 34. |
| 29/10 09:42 | 4h50 / 15 min | 3 / **8.709** | Digitação intensa. É o padrão de layout forçado por tecla (§3.4). |
| 02/11 14:52 | 2h13 / 40 min | 0 / 0 | INP pior de 2.432 ms. |
| 02/11 11:36 | 56 min / 3 min | 0 / 1.482 | 7 interações ruins em 10. |

* Os cliques e teclas da gravação ficam escondidos onde há `ph-no-capture` e
  `maskTextSelector:'*'`, então a gravação mostra pouco do conteúdo, por desenho.
* **Não rodei o Replay Vision:** ele consome créditos, grava observações persistentes
  no PostHog e envia as gravações a um modelo externo. Ver Q3.

---

## 5. Fase quatro — atritos de usabilidade

Os itens abaixo só apontam o atrito. Nada foi renomeado nem alterado.

**Ações escondidas ou com cliques demais**
* O menu da linha ("Mais ações") só aparece no hover; no toque, só com pressão longa de
  500 ms (`ChatListItem.tsx:130-139,357-367`).
* As ações do balão usam `opacity-0 group-hover:opacity-100` (`MessageActions.tsx:143-146`).
  No toque, a reação fica indisponível (`MessageWrapper.tsx:364`).
* Fixar, arquivar e marcar como não lida existem só no menu de contexto.
* "Encaminhar" e "Apagar" existem no componente mas nunca aparecem, porque o
  `MessageArea` não passa `onForward`/`onDelete`.

**Feedback ausente**
* Resolver, reabrir, arquivar e fixar não mostram confirmação de sucesso (só bloquear e
  desbloquear mostram).
* Erro ao resolver pelo cabeçalho é silencioso (§3.8).
* Áudio que falha não mostra toast.
* Upload não tem progresso, e o botão de enviar fica bloqueado durante qualquer upload
  (`SendButtons.tsx:33,54`).
* A mensagem de texto enviada fica com um tique até o Pusher trocar a bolha. Sem o
  Pusher, ela fica `temp-` para sempre: sem entregue/lida e sem poder ser respondida.
* A troca da bolha temporária pela real é por heurística de conteúdo e tempo, sem
  `tempId` enviado ao backend (`message-store.ts:203-219`). Duas mensagens iguais
  seguidas podem casar com a bolha errada (H).
* O modelo que falha mostra no toast a mensagem crua do servidor
  (`MessageInput.tsx:286-292`).

**Diferenças em relação ao WhatsApp**
* Enter envia também no celular (`MessageInput.tsx:111-126`). No WhatsApp mobile, Enter
  quebra linha.
* Não há separador nem salto para "não lidas": a abertura sempre rola ao fim. A pílula
  "Novas mensagens" não diz quantas são.
* Pular para a mensagem citada falha em silêncio se ela não estiver carregada
  (`MessageArea.tsx:390-402`).
* O voltar do sistema sai do `/chat` em vez de voltar da conversa para a lista (§3.13).
* O Ctrl/Cmd+R do navegador é capturado para a correção por IA. Foi uma decisão
  registrada no código (`useMessageInput.ts:714-719`); fica só o apontamento de que
  substitui o recarregar do navegador.

**Rótulos fora de um glossário único** (texto atual, citado exato)
* Abas: `Todos`, `Minhas`, `Abertos`, `Resolvidas` (`ChatFilters.tsx:77-81`).
  * Os rótulos misturam gêneros.
  * A aba "Abertos" é a fila sem atendente, mas na sidebar o mesmo conjunto se chama
    `Sem atendente`.
  * A sidebar usa `Abertas` para o conjunto da aba "Todos" (`StatusFilterList.tsx:45-66`).
  * A memória do projeto registra a aba `'open'` como "Sem atendente" desde a Fase 3A.
    O código atual mostra "Abertos". **Divergência a esclarecer** (Q5).
* O vazio da aba "Abertos" diz `Nenhuma conversa aberta` (`ChatListEmpty.tsx:33-35`).
* Menu `Marcar como resolvida` × cabeçalho `Resolver conversa`; e
  `Limpar histórico de bate-papo` (`ConversationContextMenu.tsx:268,274`).
* O placeholder da busca, `Buscar ou começar uma nova conversa` (`ChatListHeader.tsx:259`),
  promete iniciar conversa, mas digitar só busca (H).
* `ChatWelcome` mostra `Suas mensagens pessoais são protegidas.` (`ChatWelcome.tsx:164`),
  frase do WhatsApp pessoal num atendimento comercial.
* A seção da sidebar se chama `Tags` (`InboxFilterSidebar.tsx:140`), enquanto o resto do
  produto fala em etiquetas.

---

## 6. Instrumentação faltante

Ordem por valor para as próximas decisões:

1. **Frontend — tempo até a lista utilizável:** marca no `/chat` do início da navegação
   até a primeira renderização com dados, enviada como evento de produto com aparelho e
   faixa de tamanho da lista. Sem conteúdo nem identificador.
2. **Frontend — troca de conversa:** do clique na linha até o histórico renderizado,
   com origem cache/rede.
3. **Envio de ponta a ponta:** `tempId` (ou `clientMessageId`) do cliente até o
   `new-message`, medindo clique → HTTP → job → Pusher. No backend, `source` em
   `channel.message.sent` (chat, campanha, fluxo, IA).
4. **`mark-read`:** `trackPerformance` em cada etapa (acesso, `findMany`, `updateMany`,
   `conversations.update`, Pusher) e uma série de duração abaixo do limiar, amostrada.
5. **Publicação no Pusher:** duração e falha estruturada (`event`) no `pusher.service`.
   Hoje a falha é só `logger.error` com texto livre.
6. **Origem das listagens:** cabeçalho ou parâmetro que marque `fetchSingleChat`,
   `silentRefresh` e ação do usuário, refletido no `http.request` (que não guarda query
   string).
7. **Mensagem recebida por tenant:** evento de entrada com `tenantId` e
   `channelAccountId`. O `/whatsapp/webhook` chega sem `tenantId`.
8. **CLS com atribuição:** hoje o `largestShiftTarget` vem vazio.
9. **Eventos de produto do chat** no registro `PRODUCT_EVENTS`: conversa aberta,
   mensagem enviada e envio falho, só com enum e booleano. Hoje o autocapture está
   desligado por `ph-no-capture`.
10. **Falhas como 2xx:** um contador por rota de `success:false`, para que o
    `statusCode` deixe de esconder falha.
11. **Redis do caminho quente** (throttler, JWT, `user:ctx`, versões): duração amostrada,
    para medir o custo do guard duplicado (§3.10).

---

## 7. Ordem proposta dos pacotes de correção

Do maior ganho por esforço ao menor. Nenhum foi iniciado.

| # | Pacote | Ganho esperado | Esforço | Base |
|---|---|---|---|---|
| 1 | **Painel de detalhes não buscar fechado** + `crm/assignees` pelo React Query + `chat-timeline`/`notes` com cache por conversa | Até ≈ 28 mil s/14 d de API e ~6 requisições a menos por troca. Também menos 429 | Pequeno, só frontend | §3.2 |
| 2 | **Corrigir o React #31** ao responder interativa e o **resolver duplicado** do menu de contexto | Fim da queda do atendimento. Fim do resumo de IA e dos efeitos em dobro | Pequeno, só frontend | §3.7, §3.8 |
| 3 | **`mark-read` agrupado por conversa** (no máximo 1 em voo e debounce na conversa aberta) + remover o `findMany` redundante + corrigir a chave morta do `unread:count` | Parte grande dos 31,6 mil s; a medir depois do item 4 do §6 | Pequeno a médio, frontend + backend | §3.1 |
| 4 | **Instrumentação** do §6, itens 1–6 | Torna medíveis a carga, a troca e o envio. Pré-requisito para decidir 5–7 | Médio | §1.3 |
| 5 | **Contadores por evento em vez de relógio** e decisão de ligar a leitura da materialização (F1 já em produção, em sombra) | Grande parte dos 45 mil s de `filter-counts` + `unread-count` | Médio, depende de decisão | §3.3 |
| 6 | **Rota `GET /conversations/:id`** para `fetchSingleChat`, e eventos que tragam o estado em vez de pedir recarga | Fim da recarga de 50 conversas por evento e do efeito manada entre abas | Médio, backend + frontend | §3.5 |
| 7 | **Corrida da carga inicial e da busca** (guarda de sequência; `forceReload` furando a trava) | Lista "Minhas" correta para atendente | Pequeno, correção de comportamento | §3.9 |
| 8 | **Custo fixo duplicado:** guard JWT duplo, `validateAccess` duplo, `Promise.all` na lista, `count` de debug | Ganho por requisição em todas as rotas, a medir | Pequeno a médio | §3.10 |
| 9 | **Mobile:** voltar limpar a seleção, `pushState` para o voltar do sistema, pontos de corte unificados | Fim da leitura automática indevida no celular | Pequeno a médio | §3.13 |
| 10 | **Virtualização** da lista e do histórico (decisão pendente desde outubro; perde Ctrl+F) | Atinge o INP de 280–350 ms do clique na linha e a piora com a idade da aba | Grande | §3.4, §4.2 |
| 11 | **Meta fora do request** em reação e bloqueio | ~1 s a menos de rota presa por reação | Médio | §3.11 |

---

## 8. Perguntas em aberto

* **Q1.** Qual é a origem do piso de 400 ms do `mark-read`? Exige `EXPLAIN` do
  `updateMany` em `messages` e a lista de triggers em `messages` e `conversations` no
  banco de produção. Não foi feito porque o acesso ao banco estava fora do escopo.
* **Q2.** Quem é o `userId` `241bd083`, que opera 12 empresas e faz até 2.445 `/tags`
  por hora? É conta de suporte, revenda ou automação? Que tela fica aberta nessas horas?
* **Q3.** Autoriza rodar o Replay Vision (créditos, observação persistente, envio a
  modelo externo) nas 12 sessões selecionadas, para descrever o que o atendente viu?
* **Q4.** Onde nasce o React #185 (laço de atualização) visto em 23/10 e 30/10? Não há
  `componentStack` no evento.
* **Q5.** O rótulo atual da aba `'open'` é "Abertos" no código de produção, mas a memória
  do projeto o registra como "Sem atendente". Qual dos dois é o rótulo vigente decidido?
* **Q6.** O pós-atendimento em dobro sob o resolver duplicado está de fato bloqueado
  pelo `findActiveExecution` quando os dois jobs correm ao mesmo tempo? E a nota de
  resumo de IA sai duplicada? Confirmável com o evento `flow.closing_flow.*` e as notas
  `ai_summary` por conversa.
