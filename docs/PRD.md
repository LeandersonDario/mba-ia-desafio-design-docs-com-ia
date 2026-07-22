# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| Autor | Marcos (Product Manager) |
| Revisores | Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| Status | Em revisão |
| Data | Reunião de quinta-feira, 09:00 (conforme cabeçalho da `TRANSCRICAO.md`; data de calendário não registrada) |
| Documentos relacionados | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) |

## 1. Resumo e contexto

Este documento define os requisitos de produto do **Sistema de Webhooks de Notificação de Pedidos** para o OMS existente. A feature permite que clientes B2B cadastrem endpoints HTTPS próprios e recebam, em menos de 10 segundos ([09:02] Marcos), uma notificação assinada a cada mudança de status dos seus pedidos — eliminando a necessidade de consultar repetidamente o `GET /orders` para descobrir se algo mudou ([09:00] Marcos).

A demanda nasceu de um pedido formal de três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** ([09:00] Marcos) — e tem compromisso comercial para **fim de novembro** ([09:45] Marcos), com estimativa de execução de **3 sprints** ([09:46] Larissa), incluindo a revisão de segurança da Sofia ([09:46] Sofia). A solução técnica (padrão Outbox no MySQL, worker separado, retry + DLQ, HMAC) está descrita na [RFC](RFC.md) e nos [ADRs](adrs/); este PRD trata do problema, do escopo e dos critérios de sucesso do ponto de vista de produto.

## 2. Problema e motivação

- **Integração por polling é lenta e cara para o cliente.** Hoje os três clientes "ficam batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles" ([09:00] Marcos). O cliente paga o custo de consultar continuamente sem garantia de frescor da informação.
- **Expectativa de tempo real não atendida.** Para os clientes, "tempo real" significa qualquer notificação **abaixo de 10 segundos**; o essencial é "que não fique pendurado e eles tenham que ficar atualizando manualmente" ([09:02] Marcos). O modelo atual não oferece isso.
- **Risco comercial concreto (churn).** A Atlas Comercial "chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente" ([09:00] Marcos). A feature é, portanto, defensiva de receita além de melhoria de integração.
- **A plataforma não tem nenhum mecanismo de notificação externa.** O fluxo pedido é exclusivamente outbound — "Só saindo da gente pra eles. Eles querem receber, não mandar" ([09:02] Marcos).

Em resumo:

| | Hoje (polling) | Com a feature (webhooks) |
|---|---|---|
| Como o cliente descobre a mudança | Consulta repetida ao `GET /orders` ([09:00] Marcos) | Notificação enviada pela plataforma a cada mudança de status ([09:00, 09:02] Marcos) |
| Latência percebida | Depende da frequência de polling do cliente | < 10 segundos ([09:02] Marcos) |
| Custo para o cliente | Integração "lenta e cara" ([09:00] Marcos) | Recebe apenas os status que assinou ([09:33] Marcos) |
| Risco comercial | Ameaça de churn da Atlas até o fim do trimestre ([09:00] Marcos) | Compromisso de entrega para fim de novembro ([09:45] Marcos) |

## 3. Público-alvo e cenários de uso

**Público-alvo:** clientes B2B integradores da plataforma — na primeira fase, os três solicitantes: **Atlas Comercial, MaxDistribuição e Nova Cargo** ([09:00] Marcos). São times técnicos dos clientes que integram os sistemas deles à nossa API; a documentação de integração será publicada no portal de desenvolvedor ([09:26] e [09:40] Marcos). Internamente, operadores com role ADMIN também são usuários (reprocessamento de falhas — [09:36] Sofia/Larissa).

Cenários de uso:

1. **Sistema do cliente reage à expedição do pedido.** O cliente assina apenas os status que lhe interessam — exemplo dado na reunião: "só quero saber quando vira SHIPPED e DELIVERED" ([09:33] Marcos) — e o sistema dele (ex.: ERP/logística, cenário ilustrativo derivado desse exemplo) dispara seus processos internos ao receber a notificação, em vez de descobrir a mudança por polling.
2. **Operador ADMIN reprocessa entregas mortas.** Um evento que esgotou as tentativas de entrega vai para a fila morta (DLQ); um operador com role ADMIN aciona o replay manual, com registro de quem executou para auditoria ([09:18] Diego; [09:35–09:36] Diego/Sofia/Larissa).
3. **Cliente rotaciona a secret por política de segurança.** O cliente solicita nova secret pela API; a antiga permanece válida por 24 horas em paralelo "pra ele ter tempo de migrar os sistemas dele" ([09:21] Sofia) — motivado por incidente real de secret vazada em log de cliente ([09:22] Diego).
4. **Cliente consulta o histórico de entregas para diagnóstico.** "O cliente precisa conseguir ver o histórico de entregas. Tipo 'esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta'" ([09:34] Marcos) — dando ao time técnico do cliente visibilidade para depurar a integração do lado dele.

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta | Âncora |
|---|---|---|---|
| Notificar mudanças de status em "tempo real" | Latência entre mudança de status e recebimento pelo cliente (p95) | **< 10 segundos** | [09:02] Marcos; [09:09–09:10] Diego/Marcos |
| Entregar mesmo com indisponibilidade temporária do cliente | Eventos entregues automaticamente dentro da janela de retry | Entrega automática para indisponibilidades de até **~15h** (5 tentativas, backoff 1m/5m/30m/2h/12h) | [09:17] Diego/Larissa |
| Reter os clientes solicitantes | Adoção pelos 3 clientes (Atlas, MaxDistribuição, Nova Cargo) | 3 de 3 integrados **antes do fim do trimestre** | [09:00] Marcos |
| Reduzir o custo de integração dos clientes | Volume de polling no `GET /orders` pelos clientes integrados | Redução observável após adoção (meta qualitativa, derivada do problema relatado) | derivado de [09:00] Marcos |

## 5. Escopo

### Incluso

- Webhooks **outbound** de mudança de status de pedido ([09:00, 09:02] Marcos).
- CRUD de configuração de webhook por customer: criação, edição, remoção e listagem ([09:31] Marcos; [09:33] Bruno).
- Filtro por status: cada endpoint escolhe quais status quer ouvir ([09:33] Marcos/Bruno).
- Assinatura HMAC-SHA256 com secret única por endpoint e headers de identificação ([09:20–09:22] Sofia; [09:44–09:45] Diego/Sofia).
- Retry com backoff exponencial, DLQ e replay manual por ADMIN ([09:15–09:18] Diego/Larissa; [09:36] Sofia).
- Histórico de entregas consultável pelo cliente (~100 últimas) ([09:34] Marcos).
- Rotação de secret com grace period de 24h ([09:21] Sofia).

### Fora de escopo

| Item | Decisão registrada | Âncora |
|---|---|---|
| Notificação por e-mail quando o webhook do cliente falha | Marcos pediu; Larissa: "Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" | [09:37] Marcos/Larissa; [09:38] Marcos ("anotado como futuro") |
| Dashboard/painel visual para o cliente | Larissa: "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" | [09:39] Marcos; [09:40] Larissa |
| Webhooks inbound (cliente enviando para a plataforma) | "Só saindo da gente pra eles" | [09:02] Marcos; [09:03] Sofia |
| Arquivamento das linhas entregues da outbox | "Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature" | [09:08] Diego |
| Múltiplos workers em paralelo | "Problema do futuro, não agora" — single-worker nesta fase | [09:13] Diego |
| Rate limiting de envio para o cliente | Em aberto: "a gente observa e implementa se virar problema" | [09:38–09:39] Diego/Larissa |
| Endurecimento de roles do CRUD de configuração | "Por enquanto sim [qualquer role autenticada]. Mais pra frente a gente pode endurecer" | [09:36–09:37] Marcos/Sofia |

## 6. Requisitos funcionais

- **RF-01 — Notificação de mudança de status via webhook.** O sistema deve notificar o endpoint do cliente, via chamada HTTP outbound, a cada mudança de status dos pedidos dele, atendendo à expectativa de "tempo real" (< 10s).
  Âncora: [09:00, 09:02] Marcos.

- **RF-02 — Cadastro de webhook com secret gerada pela plataforma.** O cliente deve poder cadastrar um webhook (endpoint POST) informando a URL e a lista de status desejados; a secret é **gerada pela plataforma e devolvida na criação** — o cliente não escolhe a própria secret.
  Âncora: [09:31] Marcos.

- **RF-03 — Edição de webhook.** O cliente deve poder editar a configuração de um webhook existente (PATCH).
  Âncora: [09:33] Bruno.

- **RF-04 — Remoção de webhook.** O cliente deve poder remover um webhook (DELETE).
  Âncora: [09:33] Bruno.

- **RF-05 — Listagem de webhooks por customer.** O cliente deve poder listar os webhooks de um customer (GET). O `customer_id` é informado no body ou path — não deriva do JWT, que é do usuário operador.
  Âncora: [09:33] Bruno; [09:32] Marcos/Larissa.

- **RF-06 — Filtro por status assinado.** Cada webhook deve permitir escolher quais status ouvir (ex.: "só quero saber quando vira SHIPPED e DELIVERED"); mudanças de status não assinadas por nenhum webhook do customer não geram evento.
  Âncora: [09:33–09:34] Marcos/Bruno.

- **RF-07 — Histórico de entregas.** O cliente deve poder consultar o histórico das últimas ~100 entregas do seu webhook, incluindo sucesso/falha, payload, response e tempo de resposta.
  Âncora: [09:34] Marcos.

- **RF-08 — Replay de DLQ por ADMIN.** Um operador com role **ADMIN** deve poder reprocessar (replay) manualmente eventos da fila morta, recolocando-os para entrega; a ação deve registrar quem a executou, para auditoria. Demais operações do CRUD de configuração aceitam qualquer role autenticada nesta fase.
  Âncora: [09:18] Diego; [09:35–09:36] Diego/Sofia/Larissa; [09:36–09:37] Marcos/Sofia.

- **RF-09 — Rotação de secret com grace period de 24h.** O cliente deve poder solicitar nova secret pela API; a secret antiga permanece válida por 24 horas em paralelo, para dar tempo de migração dos sistemas do cliente, e depois expira.
  Âncora: [09:21–09:22] Sofia.

- **RF-10 — Assinatura HMAC e headers de identificação.** Cada requisição de webhook deve ser assinada com HMAC-SHA256 sobre o corpo e conter os headers `X-Signature`, `X-Event-Id`, `X-Timestamp` e `X-Webhook-Id`, permitindo ao cliente validar origem/integridade, deduplicar e identificar o cadastro de origem.
  Âncora: [09:20, 09:22] Sofia; [09:44–09:45] Diego/Sofia.

- **RF-11 — Publicação transacional do evento.** O registro do evento de notificação deve ocorrer na **mesma transação** da mudança de status: não pode existir status alterado sem evento registrado, nem evento registrado sem status alterado.
  Âncora: [09:06] Diego; [09:40–09:41] Bruno/Diego.

## 7. Requisitos não funcionais

- **RNF-01 — Latência.** Notificação recebida pelo cliente em menos de **10 segundos** após a mudança de status — definição de "tempo real" dada pelos próprios clientes.
  Âncora: [09:02] Marcos.

- **RNF-02 — Semântica de entrega at-least-once.** O cliente pode receber o mesmo evento mais de uma vez (retry, replay) e deduplica pelo `X-Event-Id`; o comportamento será documentado com destaque no portal de desenvolvedor.
  Âncora: [09:24–09:26] Diego/Larissa/Marcos.

- **RNF-03 — Atomicidade.** Falha ao registrar o evento causa rollback da mudança de status (e rollback da mudança descarta o evento).
  Âncora: [09:06] Diego; [09:40] Bruno.

- **RNF-04 — HTTPS obrigatório.** URLs de webhook com `http` são recusadas com erro de validação no cadastro.
  Âncora: [09:23] Sofia.

- **RNF-05 — Limite de payload de 64KB.** Eventos acima de 64KB não são enviados — geram erro, não truncamento.
  Âncora: [09:23–09:24] Sofia/Diego/Larissa.

- **RNF-06 — Timeout de 10 segundos.** Chamadas ao endpoint do cliente têm timeout de 10s; estouro conta como falha e agenda retry.
  Âncora: [09:42] Diego.

- **RNF-07 — Disponibilidade do envio independente da API.** O envio roda em processo separado (worker); reinício da API não interrompe a entrega de notificações.
  Âncora: [09:11] Diego.

- **RNF-08 — Auditoria do replay.** Todo replay de DLQ registra em log quem executou a ação.
  Âncora: [09:36] Sofia.

## 8. Decisões e trade-offs principais

Resumo executivo — o racional completo e as alternativas descartadas estão em cada ADR.

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Padrão Outbox no MySQL existente (sem broker/infra nova) | Simplicidade operacional em troca de acoplamento ao banco atual | [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md) |
| Worker em processo separado com polling de 2s | Latência mínima de ~2s em troca de não improvisar notificação reativa no MySQL | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| Retry com backoff exponencial (5 tentativas, ~15h) + DLQ em tabela separada | Após a janela, entrega vira intervenção manual (replay) | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256 com secret única por endpoint e rotação com grace de 24h | Gestão de secret por endpoint em troca de conter o raio de dano de vazamentos | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| Entrega at-least-once com dedup por `X-Event-Id` | Responsabilidade de deduplicação passa ao cliente (padrão de mercado) | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso dos padrões existentes do projeto (módulos, erros, logger, validação) | Nenhuma tecnologia nova em troca de consistência e velocidade de entrega | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Payload como snapshot renderizado na inserção | Evento reflete o estado do pedido no momento da mudança, mesmo que o pedido mude depois | [ADR-007](adrs/ADR-007-payload-snapshot-na-insercao.md) |

## 9. Dependências

- **MySQL e Prisma já existentes** — a solução não exige infraestrutura nova; a outbox vive no banco atual ([09:07] Diego/Larissa). O worker usa o mesmo banco e a mesma stack ([09:11] Bruno/Diego).
- **Módulo de pedidos** — a publicação do evento entra na transação do método `changeStatus` do service de orders (`src/modules/orders/order.service.ts`), o ponto mais sensível da integração ([09:40] Bruno).
- **Disponibilidade dos endpoints dos clientes** — a entrega em <10s e a entrega automática dependem de o endpoint HTTPS do cliente estar no ar; indisponibilidades além da janela de retry exigem replay manual ([09:16–09:18] Diego).
- **Revisão de segurança da Sofia antes do deploy** — mínimo de **2 dias úteis** reservados, com foco em HMAC e geração de secret; é gate para subir ([09:46] Sofia; reforçado em [09:49] Sofia).

## 10. Riscos e mitigação

Probabilidade e impacto são estimativas editoriais deste PRD; os fatos-base estão ancorados na reunião.

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Cliente indisponível por mais de ~15h perde a entrega automática do evento ([09:16–09:17] Diego) | Baixa | Médio | DLQ persistida + replay manual por ADMIN recoloca o evento para entrega ([09:18] Diego; [09:35–09:36]); histórico de entregas dá visibilidade ao cliente ([09:34] Marcos) |
| Atraso frente ao compromisso comercial de fim de novembro — estimativa de 3 sprints já inclui a revisão de segurança ([09:45] Marcos; [09:46] Larissa/Sofia) | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limiting explicitamente fora — [09:37–09:40]); revisão da Sofia agendada dentro dos 3 sprints, não ao final como extra ([09:47] Larissa; [09:49] Sofia) |
| Crescimento da tabela de outbox sem arquivamento nesta fase ([09:08] Diego) | Média | Baixo | Índices por status e `created_at` mantêm a leitura eficiente ([09:08] Diego); arquivamento das linhas entregues planejado como evolução (~30 dias) fora deste escopo |
| Vazamento de secret de webhook — já houve caso de cliente vazando secret em log ([09:22] Diego) | Baixa | Alto | Secret única por endpoint limita o raio de dano ("senão se vaza uma, vaza tudo" — [09:21] Sofia) + rotação via API com grace de 24h ([09:21] Sofia) |
| Cliente com alto volume de pedidos recebe rajadas de chamadas (ex.: 50 mudanças de status em 1 minuto = 50 POSTs) ([09:38] Diego) | Média | Baixo | Ponto em aberto assumido conscientemente: "observar e implementar se virar problema" ([09:39] Diego/Larissa); monitorar volume por cliente após o GA |

## 11. Critérios de aceitação

- [ ] Cliente cadastra um webhook (recebendo a secret na criação — [09:31]) e passa a receber a notificação de mudança de status em **menos de 10 segundos** ([09:02]).
- [ ] Cliente recebe **apenas** os status que assinou no cadastro do webhook ([09:33–09:34]).
- [ ] Toda notificação chega assinada (HMAC-SHA256) com os headers `X-Signature`, `X-Event-Id`, `X-Timestamp` e `X-Webhook-Id`, permitindo validação de origem e deduplicação ([09:20–09:26, 09:44–09:45]).
- [ ] Com o endpoint do cliente fora do ar, o sistema retenta automaticamente (5 tentativas, backoff 1m/5m/30m/2h/12h); esgotadas as tentativas, o evento vai para a DLQ e um ADMIN consegue reprocessá-lo com sucesso, com registro de auditoria ([09:15–09:18, 09:36]).
- [ ] Após rotação de secret, entregas assinadas com a secret antiga continuam válidas para o cliente durante a janela de 24h; depois disso, apenas a nova vale ([09:21–09:22]).
- [ ] Cliente consulta o histórico das últimas ~100 entregas com sucesso/falha, payload, response e tempo de resposta ([09:34]).
- [ ] Cadastro com URL `http` (não-HTTPS) é rejeitado com erro de validação ([09:23]).
- [ ] Nenhuma notificação por e-mail nem painel visual nesta fase — apenas endpoints de API ([09:37–09:40]).

## 12. Estratégia de testes e validação

- **Testes de integração ponta a ponta no padrão existente** (decisão da reunião: "Integração no order.service e testes ponta a ponta" compõem a estimativa — [09:46] Larissa). Seguir o padrão já estabelecido no projeto com Vitest + Supertest, tendo `tests/orders.test.ts` como referência (prática derivada do padrão do repositório, não discutida nominalmente na reunião). Cobertura mínima de produto: fluxo cadastro → mudança de status → entrega; filtro de status; retry → DLQ → replay; rotação de secret; validações de URL e payload.
- **Validação com os clientes-piloto antes do GA** (prática derivada: os três clientes solicitantes são os primeiros usuários — [09:00] Marcos): exercitar a integração contra endpoint de staging/teste de Atlas, MaxDistribuição e Nova Cargo, apoiada na documentação do portal de desenvolvedor que o Marcos publicará ([09:26, 09:40] Marcos), antes de confirmar o prazo cumprido ([09:47] Marcos).
- **Revisão de segurança como gate de deploy** (decisão da reunião): mínimo de 2 dias úteis da Sofia revisando o código de segurança — em especial HMAC e geração de secret — **antes de subir** ([09:46] Sofia; [09:49] Sofia: "só não esqueçam de me agendar pra revisão de segurança antes de subir").

---

Critérios de aceite técnicos, contratos de API e detalhes de implementação: ver [FDD](FDD.md). Racional arquitetural: [RFC](RFC.md) e [ADRs](adrs/).
