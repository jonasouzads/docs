# Pacote A do chat de atendimento — linha de base antes do deploy (2026-11-04)

Medição somente leitura, feita **antes** de publicar o pacote A do frontend (branch
`fix/chat-pacote-a`). Serve de comparação para as medições de 24 horas e de 7 dias
depois do deploy. Nada foi alterado em produção.

Fontes: Axiom, dataset `wizebot`, e PostHog, projeto 299809.

## Janela e filtros

* **Período:** 7 dias completos em horário de Brasília, de **28/10/2026 00:00 a
  04/11/2026 00:00 BRT** (`2026-10-28T03:00:00Z` a `2026-11-04T03:00:00Z`). Não cruza a
  virada de telemetria de 13/08.
* **Filtro do Axiom:** `event == "http.request"`, `env == "production"` e
  `railwayService == "Api"`, que separa a API dos workers e exclui a telemetria local.
* **Hora útil:** segunda a sexta, das 08h às 19h BRT, o que dá 5 dias × 11 h =
  **55 horas úteis**. Atenção: 02/11 (Finados) é feriado e foi contado como dia útil.
  As medições depois do deploy devem usar a mesma definição.
* **Duração:** campo `duration` em ms. Os percentis do Axiom são aproximados por
  histograma.
* **Denominador:** a razão de sucesso é o volume de cada rota dividido pelo volume de
  `messages-optimized` (abertura e paginação do histórico de conversas).

## Linha de base

| Rota | Método e caminho | Volume 7 d | Volume por hora útil | p50 ms | p95 ms | Tempo total s | Razão / mensagens |
|---|---|---:|---:|---:|---:|---:|---:|
| Estatísticas de mídia do contato | `GET /contacts/:contactId/media/stats` | 45.857 | 478,6 | 55 | 208 | 4.478 | 0,699 |
| Agentes | `GET /agents` | 45.919 | 480,3 | 22 | 143 | 1.874 | 0,700 |
| Metadados da conversa | `GET /whatsapp/conversations/:conversationId/metadata` | 51.794 | 545,7 | 36 | 77 | 2.161 | 0,789 |
| Responsáveis do CRM | `GET /crm/assignees` | 70.060 | 726,7 | 27 | 196 | 4.102 | 1,068 |
| Etiquetas do contato | `GET /contacts/:contactId/tags` | 48.817 | 504,3 | 28 | 62 | 1.604 | 0,744 |
| Estado de sessão | `GET /messages/session-status/:conversationId` | 57.696 | 588,2 | 19 | 40 | 1.287 | 0,879 |
| Linha do tempo de auditoria | `GET /audit-logs/chat-timeline` | 45.667 | 476,7 | 28 | 62 | 1.521 | 0,696 |
| Notas da conversa | `GET /whatsapp/conversations/:conversationId/notes` | 45.670 | 476,8 | 34 | 71 | 1.839 | 0,696 |
| **Mensagens otimizadas (denominador)** | `GET /whatsapp/conversations-optimized/:conversationId/messages-optimized` | **65.608** | **733,2** | 58 | 106 | 4.101 | 1,000 |
| Modelos | `GET /templates` | 7.561 | 85,3 | 30 | 71 | 275 | 0,115 |
| Estatísticas de modelos | `GET /templates/stats` | 6.002 | 65,8 | 20 | 45 | 147 | 0,091 |
| Resolver conversa | `POST /whatsapp/conversations/:conversationId/resolve` | 2.124 | 19,5 | 158 | 488 | 522 | 0,032 |
| Reabrir conversa | `POST /whatsapp/conversations/:conversationId/reopen` | 24 | 0,4 | 378 | 531 | 7 | 0,000 |

Resultado salvo no Axiom: <https://app.axiom.co/wizebot-t2eb/query?qid=uAX5Z4Ikv01-to04ly>

### Exceções React #31 no `/chat` (PostHog)

Contagem por dia BRT:

| Dia BRT | Exceções | Pessoas | Sessões |
|---|---:|---:|---:|
| 28/10 | 0 | 0 | 0 |
| 29/10 | 3 | 1 | 1 |
| 30/10 | 5 | 1 | 1 |
| 31/10 | 4 | 1 | 1 |
| 01/11 | 7 | 1 | 1 |
| 02/11 | 0 | 0 | 0 |
| 03/11 | 3 | 1 | 1 |
| **Total** | **22** | — | — |

## O que se espera depois do deploy

O que cada rota deve fazer, pelo desenho do pacote:

| Rota | Movimento esperado na razão / mensagens | Por quê |
|---|---|---|
| Estatísticas de mídia, agentes | **cai bastante** (hoje ≈ 0,70) | Passam a sair só com o painel de detalhes aberto. |
| Metadados | **cai** (hoje 0,79) | Só com o painel aberto ou com a barra de entrada bloqueada no modo IA. Painel e barra dividem uma busca só. |
| Responsáveis do CRM | **cai bastante** (hoje 1,07) | Uma entrada de cache por empresa, com prazo de 60 s, para todas as telas do chat. Deixam de existir as buscas diretas por troca de conversa. |
| Modelos, estatísticas de modelos | **cai** | O modal de modelo deixa de buscar ao montar o compositor. As estatísticas deixam de sair no chat. |
| Resolver | **cai** | O menu de contexto deixa de chamar a API duas vezes por clique. |
| Etiquetas do contato, estado de sessão, linha do tempo, notas | **estável** | Ficaram como estavam, por decisão. |
| React #31 no `/chat` | **zero** nas abas com o bundle novo | Corrigido no bloco 1 e na regra única de texto do bloco 5. |

**Ressalvas de leitura:**

* **`/agents`, `/templates`, `/templates/stats` e `/crm/assignees` também são usadas fora
  do chat.** `/crm/assignees` serve o CRM (`useCrm` e `kanban-store`, que o pacote não
  tocou). `/agents` serve a tela de agentes, e os modelos servem configurações e
  campanhas. A queda dessas rotas fica diluída pelo uso fora do chat.
* **`messages-optimized` inclui paginação e o pré-carregamento no hover.** O pacote não
  muda nenhum dos dois, então o denominador continua comparável.
* **Abas abertas antes do deploy seguem com o bundle antigo até recarregar.** Isso vale
  também para o React #31. No PostHog, o recorte "aba com bundle novo" é
  `properties.$initialization_time >= <hora do deploy>`. No Axiom não há como separar
  o bundle, então a janela de 24 horas ainda carrega parte do comportamento antigo.
* **Réplica lenta depois do deploy:** uma das réplicas da API costuma ficar 2 a 4 vezes
  mais lenta. Antes de ler o p95 como regressão, quebre por `railwayReplica`.

## Como repetir a medição

Troque `<INICIO_Z>` e `<FIM_Z>` pela janela desejada, sempre em dias completos BRT
(00:00 BRT = 03:00Z):

* **24 horas depois:** do 00:00 BRT seguinte ao deploy até 24 h depois. Para comparar
  volume por hora útil, use um dia útil.
* **7 dias depois:** os 7 dias completos seguintes ao deploy. Recalcule o número de
  horas úteis da janela (dias úteis × 11) e anote os feriados.

### Axiom — rotas

```kusto
['wizebot']
| where _time >= datetime(<INICIO_Z>) and _time < datetime(<FIM_Z>)
| where event == "http.request" and env == "production" and railwayService == "Api"
| where (method == "GET" and path in (
    "/api/v1/contacts/:contactId/media/stats",
    "/api/v1/agents",
    "/api/v1/whatsapp/conversations/:conversationId/metadata",
    "/api/v1/crm/assignees",
    "/api/v1/contacts/:contactId/tags",
    "/api/v1/messages/session-status/:conversationId",
    "/api/v1/audit-logs/chat-timeline",
    "/api/v1/whatsapp/conversations/:conversationId/notes",
    "/api/v1/whatsapp/conversations-optimized/:conversationId/messages-optimized",
    "/api/v1/templates",
    "/api/v1/templates/stats"))
  or (method == "POST" and path in (
    "/api/v1/whatsapp/conversations/:conversationId/resolve",
    "/api/v1/whatsapp/conversations/:conversationId/reopen"))
| extend t = _time - 3h
| extend util = dayofweek(t) >= 1 and dayofweek(t) <= 5 and hourofday(t) >= 8 and hourofday(t) < 19
| summarize n = count(), n_util = countif(util), p50 = percentile(duration, 50),
            p95 = percentile(duration, 95), total_s = sum(duration) / 1000 by method, path
| order by n desc
```

Na chamada do MCP, `startTime` e `endTime` devem cobrir a mesma janela. Como fazer as
contas:

* **Volume por hora útil:** `n_util` dividido pelas horas úteis da janela.
* **Razão:** `n` da rota dividido por `n` de `messages-optimized`.
* **`dayofweek`:** domingo = 0. A expressão foi conferida em 03/11, uma terça, que deu 2.

### Axiom — p95 por réplica (pós-deploy)

```kusto
['wizebot']
| where _time >= datetime(<INICIO_Z>) and _time < datetime(<FIM_Z>)
| where event == "http.request" and env == "production" and railwayService == "Api"
| where path in ("/api/v1/whatsapp/conversations-optimized/:conversationId/messages-optimized",
                 "/api/v1/messages/session-status/:conversationId")
| summarize n = count(), p95 = percentile(duration, 95) by path, railwayReplica
```

### PostHog — React #31 por dia BRT

As datas literais do HogQL já são lidas em BRT, o fuso do projeto.

```sql
SELECT toDate(timestamp) AS dia_brt,
       count() AS excecoes,
       uniq(person_id) AS pessoas,
       uniq($session_id) AS sessoes
FROM events
WHERE timestamp >= toDateTime('<INICIO_BRT>')
  AND timestamp < toDateTime('<FIM_BRT>')
  AND event = '$exception'
  AND properties.$pathname = '/chat'
  AND toString(properties.$exception_message) LIKE '%error #31%'
GROUP BY dia_brt
ORDER BY dia_brt
```

Para ver só as abas com o bundle novo, acrescente
`AND toFloat(properties.$initialization_time) >= <epoch ms do deploy>`, depois de
conferir no `read-data-schema` como a propriedade está gravada.
