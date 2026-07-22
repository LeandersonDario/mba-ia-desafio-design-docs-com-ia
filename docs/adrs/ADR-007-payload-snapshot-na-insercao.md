# ADR-007: Payload como snapshot renderizado na inserção da outbox

| Campo | Valor |
|---|---|
| Status | Aceito |
| Data | Quinta-feira (conforme cabeçalho da transcrição; dia de calendário não registrado) |
| Decisores | Larissa (Tech Lead), Diego (Eng. Sênior Plataforma), Bruno (Eng. Pleno Pedidos) |

## Contexto

No fim da reunião, Bruno levantou uma dúvida de modelagem da outbox ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)): "o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?" ([09:51] Bruno).

Como há um intervalo entre a inserção do evento e o envio pelo worker — polling, retries que podem se estender por ~15 horas ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) — o pedido pode mudar de status novamente antes de o evento ser entregue. O formato do payload já havia sido definido antes ([09:43] Diego).

## Decisão

O payload do evento é **renderizado como snapshot completo no momento da inserção na outbox**, refletindo o estado do pedido no instante em que o status mudou: "Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito" ([09:52] Larissa). Diego concordou — "snapshot na inserção" ([09:52]) — e Bruno fechou: "Beleza, snapshot. Decidido." ([09:52]).

Decisões relacionadas ao conteúdo da linha da outbox:

- **Payload JSON enxuto** com `event_id`, `event_type` (ex.: `"order.status_changed"`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`. **Sem items**: "Não manda items pra não inflar. Se o cliente quiser detalhes, ele bate no GET /orders/:id depois" ([09:43] Diego).
- O **id da linha da outbox é UUID**, seguindo o padrão do projeto: "UUID, segue o padrão do resto do projeto. Tudo é uuid" ([09:51] Larissa).

## Alternativas Consideradas

### Guardar só referência (order_id) e renderizar o payload na hora do envio

A outbox guardaria apenas os ids e o worker montaria o payload consultando o banco no momento do envio ([09:51] Bruno). Descartada porque o pedido pode ter mudado novamente entre a transição de status e o envio efetivo — o evento enviado não refletiria o estado da transição que o originou, produzindo o "caso esquisito" descrito por Larissa ([09:52]): um evento `from_status`/`to_status` acompanhado de dados do pedido que já não correspondem àquela transição. Com retries de até ~15 horas ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), essa janela de divergência é real.

## Consequências

### Positivas

- Cada evento é imutável e historicamente fiel: representa exatamente o estado do pedido quando o status mudou ([09:52] Larissa).
- O envio (e reenvio via replay da DLQ) não depende de consulta adicional ao estado atual do pedido — o worker só lê a linha da outbox e envia.
- Payload enxuto e estável mantém o evento pequeno, bem abaixo do limite de 64KB definido em [09:24] ([09:43] Diego, [09:44] Bruno: "mantém payload enxuto").
- O snapshot preservado na `webhook_dead_letter` serve de evidência de debug fiel ao momento da falha ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)).

### Negativas

- O payload armazenado ocupa mais espaço na outbox do que uma simples referência, somando-se ao crescimento da tabela já registrado como consequência em [ADR-001](ADR-001-padrao-outbox-no-mysql.md) (arquivamento fora de escopo, [09:08] Diego).
- Se o cliente precisar do estado atual (e não do estado da transição), precisa buscar via `GET /orders/:id` ([09:43] Diego) — responsabilidade dele.
- Mudanças futuras no formato do payload não se aplicam retroativamente a eventos já inseridos/reprocessados: o snapshot é gravado no formato vigente na inserção.

## Referências

- Transcrição: [09:51] Bruno — "guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?"
- Transcrição: [09:52] Larissa — "Eu prefiro renderizado já, na hora da inserção."
- Transcrição: [09:52] Diego — "Concordo, snapshot na inserção."
- Transcrição: [09:52] Bruno — "Beleza, snapshot. Decidido."
- Transcrição: [09:43] Diego — "Não manda items pra não inflar."
- Transcrição: [09:51] Larissa — "UUID, segue o padrão do resto do projeto."
- Relacionado: [ADR-001](ADR-001-padrao-outbox-no-mysql.md) — a tabela onde o snapshot é gravado.
- Relacionado: [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) — janela de retries que motiva o snapshot.
- Relacionado: [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) — o `event_id` do payload é o mesmo UUID do header `X-Event-Id`.
