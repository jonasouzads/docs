# Proposta — reorganização do contexto do Claude Code no WizeBot

**Data:** 2026-09-10 · **Claude Code:** 2.1.259 · **Status:** proposta, nada aplicado
**Base:** `code.claude.com/docs/en/memory` e `code.claude.com/docs/en/best-practices` (lidas em 2026-09-10), complementadas por `docs/en/permissions` para a seção 5.

> Este documento é somente uma proposta. Nenhuma memória foi apagada, movida ou editada; nenhum `CLAUDE.md`, rule ou skill foi criado; nenhum setting foi alterado; nenhum commit foi feito.

---

## Estado de execução

| Fase | Estado |
|---|---|
| Verificação de histórico git de `.claude/` nos 6 repos | **feita** em 2026-09-10 — nunca commitado em nenhum repo, nenhuma ref, nenhum objeto |
| Fase 1 — `.gitignore` dos 6 repos | **aplicada** |
| Fase 1 — poda da allowlist da raiz | **aplicada** — 497 → 55 entradas, zero credenciais |
| Fase 2 — regras `ask`/`deny` + hook `PreToolUse` | **aplicada** em `~/.claude/settings.json`, 17/17 casos de teste conforme esperado |
| Rotação das duas senhas de Postgres | **pendente com o Jonas** |
| Fase 3 — os seis `CLAUDE.md` | **aplicada** — criados e commitados; seção 4 abaixo já traz o texto final |
| Fase 4 — rules com `paths` e as duas skills | **aplicada** — 9 rules e 2 skills, validadas |
| Fase 5 — poda do índice, `autoMemoryDirectory`, pente no allowlist de usuário | **aplicada** |

**As cinco fases da proposta foram executadas.** O que a fase 5 mudou, medido:

| | Antes | Depois |
|---|---|---|
| `MEMORY.md` em bytes | 19.223 (75,1% do teto) | **16.768** (65,5%) |
| `MEMORY.md` em tokens por sessão | 8,5k | **7,5k** |
| Folga até o corte de 25 KB | 6.377 bytes | **8.832 bytes** |
| `permissions.allow` do usuário | 55 entradas | **14** |
| Auto memory numa sessão dentro de `backend/` | nenhuma | **o índice da raiz, via `autoMemoryDirectory`** |

A economia do índice veio de encurtar para uma linha as 24 entradas promovidas a
`CLAUDE.md` — cada uma passou a dizer só o assunto e onde a regra vive, com o arquivo de
tópico intacto —, mais quatro encurtamentos, a fusão das duas memórias do teto de colunas
do Axiom e a poda da memória de handoff.

**Duas correções de rota na execução da fase 5.** A classificação da fase 3 tinha marcado
a memória de handoff como descartável; lida na íntegra, ela ainda carregava um **deploy
pendente em duas fases**, o fato de que a pausa de automação não expira sozinha se a fila
cair, e o motivo de o PUT de metadata ser merge raso. Foi reescrita enxuta em vez de
apagada. E a janela do Axiom carregava um alarme que não é sobre teto de coluna — a
limpeza de execuções de fluxo travada na cabeça da fila desde 2026-03-24 —, que virou
memória própria em vez de ser enterrado na fusão.

### O que a fase 4 criou

```text
frontend/.claude/
├── rules/design-system.md      paths: components/**/*.tsx, app/**/*.tsx, app/globals.css
├── rules/ui-navegacao.md       paths: app/**/*.tsx
├── rules/canais.md             paths: app/settings/**, components/settings/**
└── skills/design-system-check/SKILL.md

backend/.claude/rules/
├── jobs-e-filas.md             paths: src/**/*.worker.ts, *.handler.ts, *queue*.ts, *.processor.ts
├── mensageria.md               paths: src/modules/{messaging,flows,agents}/**/*.ts
├── node-stats.md               paths: src/modules/flows/**/*.ts
├── notificacoes.md             paths: src/modules/notifications/**/*.ts
├── rotas.md                    paths: src/**/*.controller.ts
└── arquivos-publicos.md        paths: public/**

.claude/skills/ops-db-producao/SKILL.md   (na raiz, local — cobre scripts/)
```

Um arquivo por tema, não por memória: `design-system.md` absorveu tokens, variantes,
ghost, escala de cinza, Tabler e o `TagPicker`; `ui-navegacao.md` absorveu breadcrumb e
campos fabricados; `jobs-e-filas.md` absorveu coordenação por payload e o `jobId` do BullMQ.

**Validação.** Os globs foram conferidos contra os arquivos reais de cada repo, com sonda
positiva e negativa. O carregamento foi verificado por comportamento, não por inspeção:
uma sessão que lê um arquivo casado responde um fato que só existe dentro da rule, e uma
sessão que lê arquivo não casado responde "não sei". Vale também para sessão aberta na
**raiz** lendo arquivo de repo aninhado. `/context` **não** lista rules carregadas sob
demanda — só `CLAUDE.md`, auto memory e skills; por isso o teste é comportamental.

Duas divergências deliberadas em relação ao que a seção 5.3/5.4 propunha, decididas durante a execução e detalhadas nas próprias seções: o `deny` de Prisma foi **estreitado** para os subcomandos que aplicam algo, e o caminho de DDL do hook usa **apenas** código de saída 2.

---

## Sumário dos achados

| # | Achado | Impacto |
|---|--------|---------|
| 1 | A raiz `/Users/jonas/WizeBot` **não é repositório git**; 7 subdiretórios são | Explica por completo os diretórios de auto memory separados e vazios |
| 2 | `MEMORY.md` está em **114 linhas / 18,8 KB** — 57% do teto de linhas, **75% do teto de bytes** | O corte real é o de bytes: restam ~6,4 KB, ~28 entradas no ritmo atual |
| 3 | **Não existe um único `CLAUDE.md`** em nenhum dos 6 repos | 100% do contexto persistente hoje é auto memory local, não versionada, não compartilhada |
| 4 | O modo de permissão padrão é **auto** (nenhum `defaultMode` configurado em lugar nenhum) | Um classificador aprova a maioria das ações; não há nenhuma regra `ask` ou `deny` para conter push e DDL |
| 5 | `WizeBot/.claude/settings.local.json` tem **497 entradas em `permissions.allow`**, incluindo `CREATE UNIQUE INDEX CONCURRENTLY` e `DROP INDEX CONCURRENTLY` **no banco de produção** | Contradiz frontalmente a memória `feedback_no_prod_ddl_without_ask` |
| 6 | Essa mesma allowlist contém **duas senhas de Postgres em texto puro** — a do banco de dev e a do de **produção**. Os identificadores `snllmiwhqckhchcljvms` e `fuoxpjrywculbupmmuub` que aparecem ao lado são *project refs* do Supabase: públicos, não secretos | `.claude/` não está em nenhum `.gitignore`; hoje salva pela raiz não ser repo — mas o de `backend/` está dentro de um repo |
| 7 | O índice `MEMORY.md` está **desatualizado** em pelo menos um ponto | A linha do áudio 131053 diz "falta o log da branch"; o arquivo diz `CAUSA RESOLVIDA … corrigido em aa787d4` |

---

## 1. Mapa de repositórios git

`git -C /Users/jonas/WizeBot rev-parse --show-toplevel` → `fatal: not a git repository`.
A raiz é uma **pasta guarda-chuva**, não um repo.

| Diretório | É repo? | Branch | Origin |
|---|---|---|---|
| `backend/` | sim | `main` | `git@github.com:wizebot-com-br/backend.git` |
| `frontend/` | sim | `main` | `git@github.com:wizebot-com-br/frontend.git` |
| `wizebot-admin/` | sim | `main` | `github.com/jonasouzads/wizebot-admin.git` |
| `wizebot.com.br/` | sim | `main` | `github.com/jonasouzads/wizebot.com.br.git` |
| `server-wizebot/` | sim | `main` | `github.com/jonasouzads/server-wizebot.git` |
| `docs/` | sim | `main` | `github.com/jonasouzads/docs.git` |
| `scripts/` | **não** | — | pasta solta dentro da raiz |

### Por que os diretórios de auto memory divergem

A documentação é explícita:

> *"Each project gets its own memory directory at `~/.claude/projects/<project>/memory/`. The `<project>` path is derived from the git repository, so all worktrees and subdirectories within the same repo share one auto memory directory. **Outside a git repo, the project root is used instead.**"*

Aplicando ao seu caso:

- Sessão aberta em `/Users/jonas/WizeBot` → não há repo → a chave é o **próprio caminho** → `-Users-jonas-WizeBot` → **86 memórias**.
- Sessão aberta em `/Users/jonas/WizeBot/backend` → **há** repo, e o topo dele é `backend/` → chave `-Users-jonas-WizeBot-backend` → **0 memórias**.

Estado medido:

| Diretório de auto memory | Arquivos de memória |
|---|---|
| `~/.claude/projects/-Users-jonas-WizeBot/memory/` | **86** (+ `MEMORY.md`) |
| `~/.claude/projects/-Users-jonas-WizeBot-backend/memory/` | 0 (nem existe) |
| `~/.claude/projects/-Users-jonas-WizeBot-Prompts/memory/` | 0 |
| `~/.claude/projects/-Users-jonas-WizeBot-ScriptEmerson/memory/` | 0 |

Em 2026-09-10, depois desta medição, `Prompts/`, `ScriptEmerson/` e `documentation-cloud-api/` foram movidos para fora de `/Users/jonas/WizeBot` por não fazerem parte do projeto — as duas últimas linhas da tabela acima ficaram sem objeto, e os repos do WizeBot passaram de 7 para **6**.

Não é bug nem corrupção: são projetos distintos aos olhos do Claude Code. **Abrir a sessão dentro de `backend/` hoje significa trabalhar sem nenhuma das 86 memórias.** Tratado na seção 6.

Uma consequência secundária, e favorável: como a chave da raiz é o caminho e não um repo, **renomear ou mover a pasta `/Users/jonas/WizeBot` órfã as 86 memórias de uma vez**. Vale saber antes de mexer na pasta.

---

## 2. `MEMORY.md` contra o limite de carregamento

O limite documentado:

> *"The first 200 lines of `MEMORY.md`, or the first 25KB, whichever comes first, are loaded at the start of every conversation. Content beyond that threshold is not loaded at session start."*

Medição de `~/.claude/projects/-Users-jonas-WizeBot/memory/MEMORY.md`:

| Métrica | Valor | Teto | Consumo | Folga |
|---|---|---|---|---|
| Linhas | **114** | 200 | 57,0% | 86 linhas |
| Bytes | **19.223** (18,8 KB) | 25.600 (25 KB) | **75,1%** | **6.377 bytes** |
| Entradas (`- …`) | 86 | — | — | — |

**O corte que chega primeiro é o de bytes, não o de linhas.** Ele será atingido com folga de linhas sobrando: as entradas têm em média **225 bytes** cada (bem acima de uma linha "um resumo por memória"), então a folga de 6.377 bytes comporta cerca de **28 entradas novas** — e não as 86 linhas que a contagem de linhas sugere.

O que acontece ao passar: a escrita ainda funciona, mas *"everything past the limit is dropped on the next load"* — o excedente simplesmente deixa de ser carregado, em silêncio, a partir da sessão seguinte. As memórias mais recentes ficam no fim do arquivo, então **são as recentes que somem primeiro**.

> **Atualização (fase 5 executada):** o índice foi podado e está em **16.768 bytes / 7,5k tokens**, com folga de 8.832 bytes. A medição abaixo é a de antes da poda, preservada porque é ela que justifica o trabalho.

Duas entradas passam de 400 bytes (`project_chat_ux_fase1_seguranca_operacional` e `project_mapa_latencia_endpoints_fase0_1_2`) e várias passam de 300. Encurtar as 10 maiores para ~150 bytes devolve cerca de 2 KB sem perder nenhuma memória — o detalhe já vive no arquivo de tópico, que é lido sob demanda.

---

## 3. Classificação das 86 memórias

> O diretório tem 87 arquivos: `MEMORY.md` (o índice) + **86 memórias**. O índice lista 86 entradas, então a contagem fecha.

**Categorias:** **A** = regra estável → `CLAUDE.md` · **B** = procedimento → skill · **C** = regra de escopo estreito → rule com `paths` · **D** = preferência/decisão/estado → continua em auto memory · **E** = obsoleto ou derivável → poda.

**Distribuição:** A = 24 · B = 2 · C = 17 · D = 42 · E = 1.

| # | Memória | Cat | Destino proposto | Motivo |
|---|---|:--:|---|---|
| 1 | `feedback_backfill_prod_saturou_pausado` | A | CLAUDE.md raiz | Regra de segurança operacional permanente; só o relato do incidente fica na memória |
| 2 | `feedback_breadcrumb_nao_voltar` | C | rule `frontend/ui-navegacao` | Convenção de navegação que só existe no frontend |
| 3 | `feedback_button_secondary_white_outline_gray` | C | rule `frontend/design-system` | Papel de variante por contexto de fundo; só ao tocar UI |
| 4 | `feedback_canal_novo_tela_propria` | C | rule `frontend/canais` (`app/settings/**`) | Layout da seção Canais; escopo estreito |
| 5 | `feedback_commit_por_bloco_obrigatorio` | A | CLAUDE.md raiz | Processo válido nos 6 repos; já custou uma implementação inteira |
| 6 | `feedback_continuous_execution` | A | CLAUDE.md raiz | Modo de trabalho estável — preservar a exceção de ação destrutiva/outward-facing |
| 7 | `feedback_coord_via_job_payload` | C | rule `backend/jobs-e-filas` | Padrão de coordenação assíncrona; só backend |
| 8 | `feedback_date_only_brt_parsing` | A | CLAUDE.md raiz | Cruza backend + 2 frontends; erro é off-by-one silencioso |
| 9 | `feedback_default_schema_vs_ui` | A | CLAUDE.md raiz | Retrocompatibilidade de campo novo; vale para qualquer feature |
| 10 | `feedback_design_system_first` | **B** | skill `design-system-check` (frontend) | É checklist de vários passos antes de UI, não uma regra de uma linha |
| 11 | `feedback_design_tokens_primary_vs_button` | C | rule `frontend/design-system` | Semântica de token travada; carrega só em arquivo de UI |
| 12 | `feedback_ghost_variant` | C | fundir na rule `frontend/design-system` | Detalhe de uma variante; sozinho não justifica arquivo |
| 13 | `feedback_gray_scale_balanced` | C | fundir na rule `frontend/design-system` | Só importa ao mexer em `globals.css` |
| 14 | `feedback_nao_renomear_o_que_o_usuario_ve` | A | CLAUDE.md raiz (**IMPORTANT**) | Invariante de produção; a mais cara de esquecer |
| 15 | `feedback_no_comments_public` | C | rule `backend/arquivos-publicos` (`public/**`) | Regra de um diretório só |
| 16 | `feedback_no_defensive_code_for_config` | A | CLAUDE.md raiz | Define a resposta certa quando a causa é env/config; vale em qualquer repo |
| 17 | `feedback_no_dev_status_in_ui_or_logs` | A | CLAUDE.md raiz | Cruza os 2 frontends e os logs do backend |
| 18 | `feedback_no_error_messages_to_customer` | C | rule `backend/mensageria` | Só o caminho de saída ao cliente final |
| 19 | `feedback_no_fabricated_ui_fields` | C | rule `frontend/ui-navegacao` | Vale ao implementar mockup; escopo de UI |
| 20 | `feedback_no_polling_credits` | D | fica | Preferência de um componente; não paga contexto em toda sessão |
| 21 | `feedback_no_prod_ddl_without_ask` | A | CLAUDE.md raiz **+ hook** | Crítica demais para ficar só em contexto advisory — ver seção 5 |
| 22 | `feedback_stop_and_report_divergence` | A | CLAUDE.md raiz | Método para refactor guiado por plano |
| 23 | `feedback_tabler_icons_padrao` | C | rule `frontend/design-system` | Biblioteca de ícone; só ao tocar UI |
| 24 | `project_account_created_signup_paths` | D | fica | Mapa dos 3 caminhos de cadastro; consulta pontual |
| 25 | `project_activation_toggle_listagens` | D | fica | Pacote entregue com pendências deliberadas |
| 26 | `project_admin_subscriptions_crash_neutro` | A | CLAUDE.md `wizebot-admin/` (só a regra de deploy) | Bug corrigido; sobrevive "Vercel deploya `origin/main`" |
| 27 | `project_agente_indisponivel_sem_fallback` | D | fica | Regra de produto com pendência aberta do toast |
| 28 | `project_agente_silencio_bufferizado` | A | CLAUDE.md `backend/` (2 gotchas) | `logger.log` não chega ao Axiom e `export {} from` quebra o `nest build` |
| 29 | `project_api_dois_envelopes_data` | A | CLAUDE.md `backend/` | Convenção de contrato; erra silencioso com `tsc` limpo |
| 30 | `project_audio_131053_investigacao` | D | fica — **corrigir a linha do índice** | Arquivo diz causa resolvida (`aa787d4`); índice ainda diz que falta o log |
| 31 | `project_auth_audit_2026_07` | A | CLAUDE.md `frontend/` (só as 3 allowlists) | 16,8 KB de auditoria; o que se erra sem instrução são as três listas |
| 32 | `project_axiom_custo_por_tenant_sem_dados` | D | fica | Referência de onde o custo mora; consulta ocasional |
| 33 | `project_axiom_log_schema_allowlist` | D | fica — **fundir com #61** | A pendência de vacuum aqui já foi executada na janela de 13/08 |
| 34 | `project_bff_token_never_in_browser` | A | CLAUDE.md `wizebot-admin/` | Invariante de segurança; reintroduzir o endpoint é regressão grave |
| 35 | `project_billing_admin_two_inadimplencia_sources` | A | CLAUDE.md `wizebot-admin/` | "Não unificar" é decisão travada e contraintuitiva |
| 36 | `project_billing_fail_policy` | A | CLAUDE.md `backend/` | Política fail-open/fail-closed por domínio; não derivável do código |
| 37 | `project_blog_webhook_assinatura_incompativel` | D | fica | Corrigido; sobra o ponteiro do spec no README |
| 38 | `project_bulk_close_assigned_conversations` | D | fica | Feature em implementação com decisões travadas |
| 39 | `project_campaign_counts_single_source` | A | CLAUDE.md `backend/` | Fonte única + a armadilha do export divergente |
| 40 | `project_campanha_variavel_vazia_placeholder` | D | fica | Em produção com 3 pendências nomeadas |
| 41 | `project_chat_lista_fase1_e_medicao` | D | fica — **encurtar no índice** | Estado + medição; entrada acima de 300 bytes |
| 42 | `project_chat_ux_fase1_seguranca_operacional` | D | fica — **encurtar no índice** | Maior entrada do índice; a regra permanente dela já saiu para #14 |
| 43 | `project_consolidacao_rajada_fluxos` | D | fica | Investigação recente, decisão pendente |
| 44 | `project_conta_morta_enforcement_e_cb_travado` | D | fica | Anomalia do disjuntor ainda não explicada |
| 45 | `project_contrato_multicanal_m2` | D | fica | Estado de entrega multicanal |
| 46 | `project_docs_openapi_playground` | D | fica | Pushado, não publicado; pendência com o Jonas |
| 47 | `project_failed_flow_msg_metadata_wipe` | C | rule `backend/node-stats` | Bug conhecido que enviesa toda agregação de node-stats |
| 48 | `project_feature_guard_rollout` | D | fica | Virada observe→enforce é decisão operacional pendente |
| 49 | `project_filtro_contatos_auditoria` | A | CLAUDE.md `backend/` (só o `ValidationPipe`) | Filtro desconhecido some com HTTP 200 — causa de 4 achados |
| 50 | `project_flow_duplicate_id_collision` | D | fica | Corrigido; o saneamento on-load ainda condiciona o editor |
| 51 | `project_flow_editor_foundation` | D | fica | Roadmap com decisões a não "consertar" |
| 52 | `project_flow_id_migration_status` | D | fica | Fase 5 pendente |
| 53 | `project_flow_save_400_ref_orfa` | D | fica | Bug aberto em produção, 47 fluxos |
| 54 | `project_flows_listagem_metricas_lote` | D | fica | Em produção; acerto de cache não verificado |
| 55 | `project_flows_providers_dual_module` | A | CLAUDE.md `backend/` | Armadilha de DI que quebra o boot do worker; uma linha, alto valor |
| 56 | `project_global_users_admin_endpoint` | D | fica | Fases seguintes pendentes |
| 57 | `project_handoff_state_nunca_commitado` | **E** | poda, após extrair | Resolvido em 06/08; a lição virou #5 e "raiz não é repo" vai ao CLAUDE.md raiz |
| 58 | `project_import_contacts_account_required` | D | fica | Decisão de produto com regra 0/1/2+ contas |
| 59 | `project_inbox_filter_sidebar` | D | fica | Arquitetura + itens deferidos |
| 60 | `project_invite_global_identity_phase1` | D | fica | Fase 1 com 4 decisões travadas |
| 61 | `project_janela_axiom_2026_08_13_executada` | D | fica — **fundir com #33** | Duas memórias sobre o mesmo teto de colunas em fases diferentes |
| 62 | `project_janela_ddl_2026_08_12_executada` | D | fica | Janela executada; resta paridade do `schema.prisma` |
| 63 | `project_local_db_is_dev_prod_via_axiom` | A | CLAUDE.md raiz | Qual `.env` é dev e qual é prod; errar aqui gera afirmação falsa sobre produção |
| 64 | `project_mapa_latencia_endpoints_fase0_1_2` | D | fica — **encurtar no índice** | Achados de isolamento aguardam decisão; entrada grande |
| 65 | `project_mcp_fase2_frontend` | D | fica | Sem push nem deploy |
| 66 | `project_media_upload_mediaid_migration` | D | fica | Migração ativa em enforce/100% |
| 67 | `project_meta_cobranca_servico_out2026` | D | fica | Exposição financeira com data-limite (01/10/2026) |
| 68 | `project_midia_unknown_fluxos` | D | fica | Em produção; segunda frente aberta |
| 69 | `project_notif_dispatch_vs_direct_mail` | C | rule `backend/notificacoes` | Regra de um módulo só |
| 70 | `project_posthog_error_tracking_round1` | D | fica (extrair 1 linha) | "Não recriar `types/posthog-js.d.ts`" vai para a rule do frontend |
| 71 | `project_redis_cache_executecommand_returns_null` | A | CLAUDE.md `backend/` | Sem a instrução escreve-se `catch` inalcançável — código morto |
| 72 | `project_resolucao_conversa_identidade_canal` | D | fica | Em produção 09/09 |
| 73 | `project_site_blog_sanity` | D | fica | Estado do blog; repo próprio |
| 74 | `project_tag_picker_global` | C | rule `frontend/design-system` | "Usar sempre `<TagPicker>`" + migração pendente |
| 75 | `project_telegram_f6_visibilidade` | D | fica | F6.3 e latência pendentes |
| 76 | `project_telegram_multicanal_f5` | D | fica | Teste real pendente |
| 77 | `project_tenant_selector_investigation` | D | fica | Investigação que condiciona design futuro |
| 78 | `project_tenant_switch_isolation_audit` | D | fica | Corrigido em parte; achados (c) pré-existentes |
| 79 | `project_whatsapp_disconnect_lifecycle` | D | fica | Auditoria com blast radius mapeado |
| 80 | `project_whatsapp_flows_fase0` | D | fica (extrair 1 linha) | "Módulo novo não pode se chamar `flows`" vai ao CLAUDE.md do backend |
| 81 | `reference_bullmq_jobid_dois_pontos` | C | rule `backend/jobs-e-filas` | Gotcha de biblioteca; só onde há `queue.add` |
| 82 | `reference_design_system_docs` | C | cabeçalho da rule `frontend/design-system` | É o ponteiro de "onde mora"; cabe como preâmbulo |
| 83 | `reference_log_sanitizer_mascara_ids_meta` | A | CLAUDE.md `backend/` | Destrói ID da Meta em todo log; provavelmente já quebrado em prod |
| 84 | `reference_prefixo_e_namespace_whatsapp` | C | rule `backend/rotas` (`**/*.controller.ts`) | Só ao criar ou mover rota |
| 85 | `reference_prisma7_p2002_sem_meta_target` | A | CLAUDE.md `backend/` | Guarda escrita contra `meta.target` vira 500 com teste verde |
| 86 | `reference_prod_db_ops_gotchas` | **B** | skill `ops-db-producao` | Procedimento de vários passos: timeout do role, porta 5432, nunca SQL Editor, resolução do `pg` |

### Efeito da classificação sobre o índice

Nada é apagado por essa reorganização — uma memória promovida a `CLAUDE.md` **continua existindo**, e a documentação diz que o Claude *"skips anything your CLAUDE.md files already say"*, ou seja, deixa de reescrever o que já está lá. Na prática o índice para de crescer por esse lado. As reduções diretas são as marcadas acima: 1 poda (#57), 2 fusões (#33+#61) e 4 encurtamentos (#41, #42, #64, #30) — juntas devolvem cerca de 1,5 KB dos 6,4 KB de folga.

---

## 4. `CLAUDE.md` propostos

Critério aplicado a cada linha, o da doc de boas práticas: *"Would removing this cause Claude to make mistakes?"* Nada que se descubra lendo o código entrou — sem árvore de diretórios, sem lista de dependências, sem descrição de arquitetura.

**Um detalhe de carregamento que muda o desenho:** `CLAUDE.md` é lido *"from your current working directory and every directory above it"*. Como todos os repos ficam sob `/Users/jonas/WizeBot`, um `CLAUDE.md` na raiz **é carregado mesmo quando a sessão abre dentro de `backend/`** — e o do repo entra depois, com prioridade maior por estar mais perto. A raiz não é repo, então esse arquivo **não é versionado**: é o lugar certo para o que é seu e cruza projetos. Os dos repos são versionados e compartilháveis.

### 4.1 `/Users/jonas/WizeBot/CLAUDE.md` — 46 linhas (texto final, como gravado)

```markdown
# WizeBot — contexto de topo

A raiz `/Users/jonas/WizeBot` NÃO é repositório git. Cada subprojeto é o seu:
`backend/`, `frontend/`, `wizebot-admin/`, `wizebot.com.br/`, `server-wizebot/`
e `docs/`. Todo comando git precisa de `-C <subprojeto>`; commit "na raiz" não
existe.

## Regras permanentes

- **IMPORTANT: a plataforma está em produção, com clientes na caixa de entrada.**
  Não renomeie, mova nem remova rótulo, coluna ou elemento que o usuário já vê,
  a menos que o prompt peça. Achou um rótulo errado no caminho: relate e siga
  com o comportamento corrigido, mantendo o texto atual.
- **Commit ao fim de cada bloco**, no repo correspondente, com suíte verde.
  Nada termina só no working tree.
- **Nunca aplique DDL ou migration no banco de produção.** Gere o `migration.sql`,
  rode `prisma generate` e reporte que falta aplicar. `git push` também é decisão
  do Jonas — peça antes.
- Operação pesada de dados em produção (backfill, varredura em lote) só em janela
  combinada, com freio de latência e lockfile. O backfill de mídia já saturou o
  Postgres em horário de pico e foi morto por intervenção externa.
- **Causa raiz em env/config**: diga exatamente o que mudar (qual arquivo, qual
  variável, qual valor). Não adicione validação defensiva, log extra nem throw
  preventivo no código por causa disso.
- **Nada de linguagem de processo interno na UI ou em log de produto**:
  "em desenvolvimento", "em breve", "stub", "próximo commit". Ação sem backend
  fica `disabled`, sem explicação de processo. TODO só em comentário de código.
- **Campo de config novo**: ausência no schema preserva o comportamento ATUAL;
  o default novo vale só para itens criados a partir de agora no UI.
- **Datas date-only (`YYYY-MM-DD`) são dias de calendário BRT.** Backend:
  `parseAsaasDate` de `@/common/utils/timezone`. Frontend: `Intl.DateTimeFormat`
  com `timeZone: 'America/Sao_Paulo'`. Nunca `new Date(string)` cru.
- **Refactor guiado por plano**: releia com os próprios olhos cada ponto que vai
  mudar; ao encontrar divergência plano-vs-código, PARE e relate em vez de
  improvisar. Classificação que você não confirmou, confesse como não confirmada.
- Durante implementação, vá de ponta a ponta sem pedir confirmação a cada passo.
  Isso não vale para ação destrutiva, em produção ou irreversível: essas pergunte.

## Qual banco é qual

- `backend/.env` → Supabase **DEV** (`snllmiwhqckhchcljvms`), tenant "Jonas".
  Consulta feita aqui NUNCA pode ser relatada como "verificado em produção".
- `scripts/.env` → Supabase **PROD** (`fuoxpjrywculbupmmuub`), `DIRECT_URL`.
- Axiom, dataset `wizebot`: recebe também o backend rodando local. Filtre sempre
  `env == "production"`. 2026-08-13 é virada de telemetria — não compare séries
  que cruzem essa data.
```

### 4.2 `backend/CLAUDE.md` — 48 linhas (texto final, como gravado)

```markdown
# WizeBot backend (NestJS 11 · Prisma 7 · BullMQ)

## Processos

São 8 entrypoints: a API (`dist/main`) e 7 workers (`worker`, `worker-infra`,
`worker-campaigns`, `worker-flows`, `worker-messaging`, `worker-ai`,
`worker-telegram`). Serviço novo consumido por flows precisa ser provido em
`flows.module.ts` **e** em `worker-flows.module.ts` — senão o `worker-flows`
quebra no boot por DI, e só lá.

## Contratos e armadilhas que erram em silêncio

- **Um envelope só.** O `ResponseTransformInterceptor` já devolve
  `{ success, data, timestamp }`. Controller que retorna `{ data: … }` produz
  `{data:{data}}`: a tela renderiza vazia sem erro, com `tsc` limpo. Corrija no
  controller — nunca desembrulhe duas vezes no frontend.
- **`ValidationPipe` global roda `whitelist: true` com
  `forbidNonWhitelisted: false`** (`main.ts`). Filtro ou campo desconhecido no DTO
  desaparece e a resposta é 200. É a causa por trás de vários "o filtro não aplica".
- **`RedisCacheService.executeCommand` retorna `null` e nunca lança.** Um
  `try/catch` em volta dele tem o `catch` inalcançável para falha de Redis. Teste
  `result === null` e decida fail-open vs fail-closed explicitamente.
- **Prisma 7 com driver adapter `pg`: o `P2002` não tem `meta.target`.** As colunas
  vêm em `meta.driverAdapterError.cause.constraint.fields`. Guarda escrita contra
  `meta.target` vira 500 em produção com o teste verde.
- **O sanitizador de log mascara qualquer string de 10 a 15 dígitos como telefone**,
  independentemente da chave. IDs numéricos da Meta (WABA, Flow, template) chegam
  destruídos no Axiom. Prefixe com `meta:` antes de logar.
- **`logger.log` não chega ao Axiom** — use o nível que o transport publica.
  E `export {} from '…'` quebra o `nest build`: reexporte com `export * from`.
- **BullMQ recusa `jobId` customizado com dois-pontos**, salvo quando há
  exatamente 2 (3 segmentos). O erro é `Error` cru, e fila mockada não valida.

## Políticas de produto que o código não revela

- **Enforcement de estado do tenant é por domínio, de propósito:** campanha
  (saída em massa) = **fail-closed**; caminho compartilhado de mensageria e
  inbound = **fail-open**. Bloqueio confirmado sempre descarta o job. O estado é
  lido SEMPRE no consume, nunca no enqueue.
- **Contagem de campanha tem fonte única:** `CampaignStatsService.deriveDisplayCounts`.
  O export do dashboard (`dashboard-export.service.ts`, no repo do frontend) é um
  caminho divergente conhecido — não crie uma terceira fonte.

## Nomes

Módulo novo **não pode se chamar `flows`** — o nome já é do motor de fluxos e a
colisão é silenciosa. Prefixo global da API é `api/v1` (`API_PREFIX`), com
`/admin/queues` como única exceção.
```

### 4.3 `frontend/CLAUDE.md` — 36 linhas (texto final, como gravado)

```markdown
# WizeBot frontend (Next 15 · React 19 · Tailwind 4 · Zustand)

## Antes de qualquer UI

Este projeto tem design system próprio, documentado em `docs/` e com primitivos
já customizados em `components/ui/`. Rode a skill `design-system-check` antes de
criar componente — as regras de cor e variante estão em `.claude/rules/design-system.md`
e carregam sozinhas ao abrir um `.tsx`.

Dois tokens distintos, decisão travada: **`--button` (azul) = ação do usuário**,
**`--primary` (verde) = estado e identidade**. Não os troque em CTA.

## Rota pública nova entra em TRÊS listas

Existem três allowlists de rota pública, não sincronizadas entre si:

1. `middleware.ts` → `publicRoutes` (server-side)
2. `components/layout/MainLayout.tsx` → `PUBLIC_ROUTES`
3. `contexts/AuthContext.tsx` → `publicAuthRoutes` (aparece 2×)

Faltar em qualquer uma produz o sintoma "só funciona logado" — em (2) via
redirect pós-hidratação, em (3) quando a sessão está stale. Toda rota pública
nova precisa das três, e de redeploy.

## Convenções

- Lista de conversas do chat é **Zustand** (`setFilter`), não React Query.
- Navegação de volta usa `Breadcrumb`, nunca link ou botão "Voltar".
- Ícones: `@tabler/icons-react` (o codebase ainda mistura Flaticon e lucide;
  o alvo é Tabler).
- Ao implementar mockup, não renderize campo ou botão sem fonte de dados e rota
  reais. Sem fonte: omita e avise.
- **Não recrie `types/posthog-js.d.ts`** — o stub manual escondia opções reais
  do SDK e foi deletado de propósito.
- Canal novo ganha item próprio na seção "Canais" das configurações, com pasta
  de componentes própria — nunca uma seção dentro da tela de outro canal.
```

### 4.4 `wizebot-admin/CLAUDE.md` — 25 linhas (texto final, como gravado)

```markdown
# WizeBot admin (Next 16 · porta 3002)

## O token do backend nunca chega ao browser

O JWT fica em cookie httpOnly e é injetado server-side pelo BFF
`app/api/proxy/[...path]/route.ts`. O API client chama `/api/proxy/*`
same-origin e nunca lê o token. Não reintroduza endpoint que devolva o JWT ao
cliente nem leitura do cookie no browser: o antigo `/api/auth/token` anulava o
httpOnly e foi removido por isso. Refresh: cliente recebe 401 → `getSession()`.

## Deploy

A Vercel deploya `origin/main`. Commit local não pushado **não está no ar** —
já houve caça a um bug de produção cuja causa real era essa defasagem. Antes de
diagnosticar comportamento em produção, confira `git status` e o que está no remoto.

## Inadimplência tem duas fontes, de propósito

Na tela `/admin/subscriptions` convivem, por decisão tomada com o Jonas:

- card "Inadimplentes" e KPI → `subscriptions.status = 'past_due'`
- card "Em atraso (R$)", coluna "Situação financeira" e aba "Inadimplentes"
  → `payment_history`

Os dois números **podem divergir e isso é esperado**. Não unifique.
```

### 4.5 `docs/CLAUDE.md` — 9 linhas (texto final, como gravado)

```markdown
# Docs (Mintlify)

- `docs.json` é a navegação. Página fora dele não é publicada — é assim que os
  relatórios soltos deste diretório convivem com o site.
- `x-mint.href` é obrigatório em operação de API; sem ele a URL sai acentuada.
- O schema oficial do `docs.json` é a fonte confiável para a estrutura de
  navegação — não deduza pelo que já está escrito.
- `openapi.json` é gerado do comportamento real dos controllers do backend,
  não do que a doc promete.
```

### 4.6 `wizebot.com.br/CLAUDE.md` — 9 linhas (texto final, como gravado)

```markdown
# Site institucional (Next · Sanity)

- O blog usa **Sanity** (projectId `u70jrx5c`, dataset `production`), não
  Contentful — as dependências de Contentful foram removidas por serem código morto.
- `NEXT_PUBLIC_SANITY_API_VERSION` do ambiente da Vercel **vence o default do
  código**. Ao mudar `apiVersion`, mude nos dois lugares ou nada acontece.
- `scripts/publish-post.mjs` valida o `PUBLISHED_FILTER` antes de gravar.
- O spec de assinatura do webhook do Sanity é `t=` + base64url (não `ts=` + hex);
  está versionado no README.
```

### 4.7 Sem `CLAUDE.md` proposto

`server-wizebot/` e `scripts/` não têm nenhuma das 86 memórias apontando para eles.
Criar arquivo vazio ali só gastaria contexto. `scripts/` é coberto pela skill
`ops-db-producao` e pela seção de bancos do CLAUDE.md raiz.

---

## 5. Permissões: estado atual e proposta

### 5.1 O que está configurado hoje

| Camada | Arquivo | Estado |
|---|---|---|
| Managed policy | `/Library/Application Support/ClaudeCode/` | não existe |
| Usuário | `~/.claude/settings.json` | `permissions.allow`: 55 · **sem `defaultMode`, sem `ask`, sem `deny`** |
| Projeto (raiz) | `WizeBot/.claude/settings.local.json` | `permissions.allow`: **497** · **sem `ask`, sem `deny`** |
| Projeto (backend) | `backend/.claude/settings.local.json` | só `enabledMcpjsonServers`; nenhuma permissão |

**Modo padrão: `auto`.** Nenhum `defaultMode` está definido em nenhuma camada, e em plano Pro/Max/Team auto é *"the built-in starting permission mode for interactive terminal and VS Code sessions"* — um classificador revisa as ações no seu lugar. Não há hoje **nenhuma** regra `ask` ou `deny` em nenhum arquivo: a única contenção é o classificador.

**Três problemas concretos na allowlist de 497 entradas:**

1. Estão pré-aprovados, como comandos exatos, **DDL contra o banco de produção**:
   `CREATE UNIQUE INDEX CONCURRENTLY … ON public.messages`, `DROP INDEX CONCURRENTLY …`, `DROP INDEX IF EXISTS …`. Isso contradiz frontalmente `feedback_no_prod_ddl_without_ask`.
2. Essas entradas carregam **duas senhas de Postgres em texto puro** — a do banco de dev e a do de **produção**. A distinção importa: `snllmiwhqckhchcljvms` e `fuoxpjrywculbupmmuub` são *project refs* do Supabase, identificadores públicos que aparecem na própria URL do projeto; o segredo é a senha embutida nas connection strings e nos `PGPASSWORD=`. `.claude/` não aparece em nenhum `.gitignore` dos 6 repos. A da raiz escapa de ser commitada só porque a raiz não é repo; a de `backend/` está dentro de um repo versionado.
3. `Bash(git stash *)` está aprovado — descarta trabalho não commitado sem perguntar.

### 5.2 O que as regras de permissão conseguem expressar

**Para `git push`, `ask` resolve bem.** A doc é explícita: *"Deny and ask rules apply when any subcommand matches them, including a command nested inside a subshell, a command substitution, or a control-flow body… An ask rule like `Bash(git clean *)` still prompts you for `cd /tmp && git clean -f` or `echo "$(git clean -f)"`, **even in auto mode**."* E a precedência é `deny → ask → allow`, com *"a matching ask rule prompts even when a more specific allow rule also matches"* — ou seja, a regra vence as 497 entradas da allowlist sem precisar limpá-las primeiro.

**Para DDL, as regras sozinhas não bastam**, por duas razões independentes:

- *Elas não são fronteira em volta do programa.* A doc traz a tabela: `Bash(git push *)` para `git push origin main` mas **não** para `git -C . push origin main`, `git -c push.default=current push`, nem `git 'push'`. O mesmo vale para `psql` invocado como `/usr/local/bin/psql` ou dentro de `sh -c '…'`. Literalmente: *"a deny or ask rule covers the invocation Claude usually produces and isn't a security boundary around the program."*
- *`deny` não admite exceção.* *"A broad deny rule like `Bash(aws *)` blocks every matching call, including calls that also match a narrower allow rule… so a deny rule can't carry allowlist exceptions."* Negar `Bash(psql *)` mataria junto todas as consultas de leitura em produção, que são rotina aqui. E DDL não chega só por `psql`: chega por `prisma migrate deploy`, `prisma db execute`, `supabase db push`, ou por um `.mjs` em `scripts/` que abre conexão própria — nenhum deles tem forma de comando estável para casar por prefixo.

**Conclusão:** `ask` cobre o push; o DDL exige **hook `PreToolUse`**, que é o único mecanismo que *"inspect[s] the full command text with your own logic before it runs"*. O hook também é o que sobrevive à allowlist existente: *"A hook that exits with code 2 stops the tool call before permission rules are evaluated, so the block applies even when an allow rule would otherwise let the call proceed."*

### 5.3 Configuração proposta

Em `~/.claude/settings.json` (camada de usuário, para valer em todos os projetos):

```json
{
  "permissions": {
    "ask": [
      "Bash(git push)",
      "Bash(git push:*)",
      "Bash(gh pr merge:*)",
      "Bash(git stash:*)"
    ],
    "deny": [
      "Bash(npx prisma migrate deploy)",
      "Bash(npx prisma migrate deploy:*)",
      "Bash(npx prisma migrate dev)",
      "Bash(npx prisma migrate dev:*)",
      "Bash(npx prisma migrate reset)",
      "Bash(npx prisma migrate reset:*)",
      "Bash(npx prisma migrate resolve:*)",
      "Bash(npx prisma db push)",
      "Bash(npx prisma db push:*)",
      "Bash(npx prisma db execute:*)",
      "Bash(prisma migrate deploy:*)",
      "Bash(prisma db push:*)",
      "Bash(prisma db execute:*)",
      "Bash(supabase db push:*)",
      "Bash(supabase db reset:*)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/guard-ddl-e-push.py",
            "timeout": 10,
            "statusMessage": "Verificando DDL e push..."
          }
        ]
      }
    ]
  }
}
```

`Bash(git push)` e `Bash(git push:*)` aparecem os dois de propósito: *"A rule with no `*` matches one exact command"*, então a segunda não cobre um `git push` sem argumentos. As entradas em `deny` são as formas que o Claude de fato escreve — baratas de adicionar e resolvem o caso comum antes do hook.

**Divergência assumida na execução:** a versão original desta seção negava `Bash(npx prisma migrate:*)` inteiro, o que derrubaria junto `npx prisma migrate status`, que é somente leitura e está na allowlist há tempos. Como `deny` não admite exceção, a lista foi estreitada para os subcomandos que de fato **aplicam** algo (`deploy`, `dev`, `reset`, `resolve`, `db push`, `db execute`). `prisma generate` e `migrate status` seguem liberados. Cada subcomando aparece na forma exata e na forma com `:*`, porque uma regra sem `*` casa só o comando exato.

### 5.4 O hook

Instalado e executável em `~/.claude/hooks/guard-ddl-e-push.py`. O arquivo é a fonte da verdade; o que segue é o desenho.

**Um mecanismo por caminho, nunca os dois no mesmo:**

| Caminho | Mecanismo | Por quê |
|---|---|---|
| DDL / migration | **exit 2, razão no stderr, sem JSON** | Exit 2 *"blocks the tool call regardless of JSON output"* e, pela doc de permissões, *"stops the tool call before permission rules are evaluated, so the block applies even when an allow rule would otherwise let the call proceed"*. É a única forma que sobrevive a uma regra `allow`, presente ou futura. Sem JSON impresso, a razão chega ao Claude pelo stderr — o canal documentado nesse caso. |
| `git push` | **exit 0 + JSON `permissionDecision: "ask"`** | Exit 2 só sabe bloquear; não sabe pedir confirmação. `ask` é valor válido de `permissionDecision` e, com exit 0, *"If output parses and validates, JSON decision fields take effect"*. |

Misturar os dois no mesmo caminho seria redundante e ambíguo: com exit 2 o JSON serviria apenas para trocar a origem da mensagem, e a semântica de bloqueio já está decidida pelo código de saída.

**Como o guarda evita falso positivo.** A detecção não é busca de palavras SQL no texto do comando. São duas camadas:

- **(a) Por token de programa** — `prisma migrate deploy|dev|reset|resolve`, `prisma db push|execute` e `supabase db push|reset` são reconhecidos como programa e subcomando, então uma busca textual por essas palavras não dispara nada.
- **(b) Por cabeça de comando** — palavras-chave de DDL só contam quando um cliente de banco está em **posição de comando** (primeiro token do comando ou de um segmento depois de `|`, `&&`, `;`). Mencionar o cliente dentro de um heredoc, de uma string ou de um texto não executa nada.

A camada (b) nasceu de um falso positivo real durante a instalação: a primeira versão exigia apenas que o nome do cliente aparecesse *em algum lugar* do comando, e o guarda bloqueou a escrita deste próprio documento, porque o heredoc falava de banco e de DDL na mesma linha. O critério passou a ser posição de comando, e o caso virou teste de regressão.

O script ainda entra em `sh -c '…'` recursivamente e, em `psql -f arquivo.sql`, **lê o arquivo** e aplica a mesma varredura — o DDL não precisa estar no comando. Para `git push`, ele lista o subcomando de cada invocação de `git` pulando as opções globais (`-C`, `-c`, `--git-dir`), o que faz `git log --grep push` passar e `git -C /repo push` ser interceptado.

**Bateria de testes (22 casos, 22 conforme o esperado):**

| Caso | Esperado | Obtido |
|---|---|---|
| DDL direto via `psql` | deny | deny |
| DDL com caminho absoluto do binário | deny | deny |
| DDL dentro de shell aninhado (`sh -c`) | deny | deny |
| `prisma migrate deploy` | deny | deny |
| `psql -f` com DDL dentro do arquivo | deny | deny |
| `supabase db push` | deny | deny |
| DDL canalizado de `echo` para o cliente de banco | deny | deny |
| `PGPASSWORD=… psql -c "CREATE TABLE…"` | deny | deny |
| `git push` | ask | ask |
| `git push --force-with-lease origin main` | ask | ask |
| `git push` dentro de subshell | ask | ask |
| `git push` via `sh -c` | ask | ask |
| `SELECT` em produção | passa | passa |
| `prisma generate` | passa | passa |
| `npx prisma migrate status` | passa | passa |
| `git commit -m "fix: corrige push duplicado"` | passa | passa |
| `git log --grep push` | passa | passa |
| `grep -rn "create table" src/` | passa | passa |
| `psql -f` com `SELECT` dentro do arquivo | passa | passa |
| heredoc de documentação citando banco e DDL | passa | passa |
| `grep` procurando DDL no código | passa | passa |
| `echo "DROP INDEX…" > rascunho.sql` | passa | passa |

**Limites que a proposta não esconde:**

- Um hook que estoura o `timeout` **não bloqueia**: *"A timed-out `command`… hook doesn't block the tool call… don't count on a stalled hook to act as a gate."* Daí o `timeout: 10` e um script sem I/O externo.
- O hook lê o texto do comando. Um script `.mjs` em `scripts/` que abra conexão e execute DDL por dentro passa — o texto do Bash é só `node scripts/foo.mjs`. Fechar isso de verdade exigiria revisar o script antes de rodar, ou sandbox. Para o risco real aqui (Claude digitando DDL), o hook cobre.
- O hook roda em toda chamada de Bash. Os regexes são de linha única e sem backtracking patológico; ainda assim, é custo fixo por comando.

### 5.5 Recomendações que acompanham

1. **Podar as 497 entradas.** Boa parte é `grep`/`find`/`ls` de sessões antigas que já são read-only por padrão. Depois da poda, as entradas com senha somem junto.
2. **Rotacionar as duas senhas de Postgres** que estão hoje em texto puro no `settings.local.json` — a de produção inclusive.
3. **Adicionar `.claude/settings.local.json` ao `.gitignore` dos 6 repos.** Nenhum tem hoje, e a de `backend/` está dentro de um repo versionado.
4. Considerar `Read(//Users/jonas/WizeBot/**/.env)` e `Read(//Users/jonas/WizeBot/scripts/.env)` em `deny`, já que o `.env` de produção mora ali.

---

## 6. Unificar a auto memory entre raiz e repos

O problema, medido na seção 1: as 86 memórias existem só para sessões abertas exatamente em `/Users/jonas/WizeBot`. Abrir em `backend/` dá um Claude sem nenhuma delas.

### Alternativa A — `autoMemoryDirectory` nos settings de cada repo

A doc: *"To store auto memory in a different location, set `autoMemoryDirectory` in your `settings.json`. It is read from any settings scope: user, project, local, policy, or `--settings`."* O valor precisa ser absoluto ou começar com `~/`.

Em cada repo, `.claude/settings.local.json`:

```json
{ "autoMemoryDirectory": "~/.claude/projects/-Users-jonas-WizeBot/memory" }
```

| Prós | Contras |
|---|---|
| Funciona independentemente de onde a sessão abre, inclusive por IDE que abre na pasta do repo | Sete arquivos para manter em sincronia |
| Preserva o `.claude/settings.local.json` próprio de cada repo | O caminho é da sua máquina: se for commitado, quebra para qualquer outra pessoa — **precisa** ficar em `settings.local.json` **e** no `.gitignore` (hoje não está) |
| Nada precisa mudar de hábito | Junta backend, frontend e admin num índice só: o `MEMORY.md` que já está a 75% do teto de bytes passa a crescer mais rápido |
| — | Sujeito à *"workspace trust rule as hooks in settings files"* quando definido em settings de projeto |

### Alternativa B — abrir sempre na raiz

| Prós | Contras |
|---|---|
| Zero configuração, zero sincronia | Depende de disciplina; um `code backend/` desfaz |
| Todo git já exige `-C <repo>`, então o hábito atual já é esse | Mover ou renomear `/Users/jonas/WizeBot` **órfã as 86 memórias**, porque fora de repo a chave é o caminho |
| Um índice só, no tamanho que já tem | Perde a separação natural por repo, se um dia ela for desejada |

### Recomendação: as duas, em papéis diferentes

**Abrir na raiz continua sendo o caminho principal** — é o que faz os `additionalDirectories` e o `CLAUDE.md` raiz funcionarem juntos, e é o hábito que já está instalado.

**`autoMemoryDirectory` entra como rede de proteção**, nos 7 `.claude/settings.local.json`, apontando para o mesmo diretório. Assim, a sessão aberta por engano dentro de `backend/` não perde as memórias nem começa a escrever um segundo índice paralelo que ninguém vai ler. Como `CLAUDE.md` já é carregado a partir de todos os diretórios acima do cwd, a raiz continua entregando as regras de topo nesse cenário — o que falta hoje é exatamente a auto memory, e é isso que a Alternativa A cobre.

Duas condições para adotar:

1. `.claude/settings.local.json` precisa entrar no `.gitignore` dos 6 repos **antes** — senão um caminho de máquina local vai para o remoto.
2. Com todos os repos escrevendo no mesmo índice, a folga de 6,4 KB encolhe mais rápido. A poda da seção 3 deixa de ser opcional e vira pré-requisito.

**Alternativa descartada:** `CLAUDE_CODE_PROJECT_DIR_NAME` ao lado de `CLAUDE_CONFIG_DIR` (v2.1.234+) resolveria de uma vez, mas obriga a exportar duas variáveis em todo shell e IDE, e passa a valer para qualquer repositório aberto com aquele config dir — inclusive os 19 outros projetos em `~/.claude/projects/`. Custo alto para o ganho.

---

## 7. O que este documento não fez

Conforme as restrições do pedido:

- nenhuma das 86 memórias foi apagada, movida ou editada;
- nenhum `CLAUDE.md`, rule ou skill foi criado — os textos da seção 4 são rascunhos dentro deste documento;
- nenhum arquivo de settings foi alterado; nenhum hook foi instalado;
- nenhum commit foi feito. Este arquivo está **untracked** em `docs/`, que é repo próprio (`github.com/jonasouzads/docs`). Como não está em `docs.json`, não é publicado pelo Mintlify — mesma situação dos outros relatórios que já vivem nesse diretório.

### Ordem sugerida, se for aprovado

1. `.gitignore` dos 6 repos + rotação das duas senhas (seção 5.5) — é o único item com risco vivo.
2. Regras `ask`/`deny` + hook (seção 5.3 e 5.4).
3. `CLAUDE.md` raiz e dos repos (seção 4); as memórias promovidas continuam no lugar.
4. Rules com `paths` e as 2 skills (seção 3).
5. Poda, fusões e encurtamentos do índice (seção 3) + `autoMemoryDirectory` (seção 6).
