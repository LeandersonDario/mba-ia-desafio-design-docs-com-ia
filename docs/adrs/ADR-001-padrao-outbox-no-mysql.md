# ADR-001: Padrão Outbox no MySQL existente

| Campo | Valor |
|---|---|
| Status | Aceito |
| Data | Quinta-feira (conforme cabeçalho da transcrição; dia de calendário não registrado) |
| Decisores | Larissa (Tech Lead), Diego (Eng. Sênior Plataforma), Bruno (Eng. Pleno Pedidos) |

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram notificação quando o status de seus pedidos muda, com tolerância de latência "abaixo de 10 segundos" ([09:00] e [09:02] Marcos). A questão central levantada foi: "a gente dispara isso sincronamente no service de orders quando o status muda, ou faz algum tipo de fila/outbox?" ([09:03] Larissa).

As forças em jogo:

- A transação de mudança de status já é pesada: atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity` ([09:04] Bruno).
- Não pode existir caso de status mudar sem o evento sair, nem de evento sair sem o status ter mudado ([09:06] Diego; reforçado em [09:40] Bruno e [09:41] Diego: "Se ficar fora da transação, perde a garantia toda").
- O time é pequeno e não quer operar infraestrutura adicional ([09:07] Diego).

## Decisão

Adotar o **padrão Outbox no MySQL já existente**: quando o status do pedido muda, **na mesma transação SQL** que atualiza `orders` e `order_status_history`, é inserida uma linha na tabela `webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP ([09:06] Diego: "dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também insere uma linha numa tabela tipo webhook_outbox"). Se a transação principal comitou, o evento foi registrado; se deu rollback, o evento some junto — "Não tem inconsistência possível" ([09:06] Diego). Decisão formalizada em [09:08] Larissa: "Tá decidido então: outbox em MySQL".

Detalhes de modelagem e leitura:

- Índices no campo de status (**pendente, processando, falhou, entregue**) e em `created_at` ([09:08] Diego).
- O worker lê apenas os pendentes em **batch pequeno**, processa e marca como entregue ([09:08] Diego).
- Se a inserção na outbox falhar, a transação inteira sofre rollback ([09:40] Bruno: "Se a outbox falhar de inserir, rollback").

## Alternativas Consideradas

### Disparo síncrono no service de orders

Fazer a chamada HTTP ao webhook dentro do próprio fluxo de mudança de status. Descartada: "Síncrono não rola. A transação de mudança de status hoje já é pesada [...] qualquer cliente lento vai travar mudança de status pra outros pedidos" ([09:04] Bruno). Além disso, se o cliente estiver fora do ar, não há o que fazer — "dá rollback na mudança de status? Não dá" ([09:04] Bruno). Diego encerrou o assunto ao entrar na call: "Síncrono está fora de questão" ([09:06]).

### Redis Streams / message broker dedicado

Usar uma fila externa (Redis Streams "ou alguma coisa parecida") para os eventos. Descartada porque exigiria subir e operar mais infraestrutura ([09:07] Larissa: "a gente acabaria precisando subir mais infra"), desproporcional para o tamanho do time: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" ([09:07] Diego).

## Consequências

### Positivas

- Atomicidade garantida entre a mudança de status e o registro do evento — sem estados inconsistentes ([09:06] Diego).
- Nenhuma infraestrutura nova para provisionar ou operar; usa o MySQL que já existe ([09:07] Diego).
- A mudança de status nunca fica bloqueada por cliente lento ou fora do ar ([09:04] Bruno).
- Leitura eficiente dos pendentes via índices em status e `created_at` ([09:08] Diego).

### Negativas

- Latência mínima inerente ao modelo de polling do worker (ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)) — aceitável dentro do requisito de "abaixo de 10 segundos" ([09:02] Marcos, [09:09] Diego).
- A tabela `webhook_outbox` cresce continuamente. O arquivamento de linhas entregues ("depois de 30 dias ou assim") ficou explicitamente **fora do escopo** desta feature ([09:08] Diego) e precisará ser tratado depois.

## Referências

- Transcrição: [09:04] Bruno — "Síncrono não rola."
- Transcrição: [09:06] Diego — "Síncrono está fora de questão."
- Transcrição: [09:06] Diego — "dentro da mesma transação SQL [...] a gente também insere uma linha numa tabela tipo webhook_outbox"
- Transcrição: [09:07] Diego — "Subir Redis Cluster pra isso é overengineering."
- Transcrição: [09:08] Larissa — "Tá decidido então: outbox em MySQL."
- Relacionado: [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) — como a outbox é consumida.
- Relacionado: [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) — o que acontece quando a entrega falha.
- Relacionado: [ADR-007](ADR-007-payload-snapshot-na-insercao.md) — conteúdo gravado na linha da outbox.
