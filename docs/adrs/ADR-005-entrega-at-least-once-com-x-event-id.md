# ADR-005: Entrega at-least-once com X-Event-Id

| Campo | Valor |
|---|---|
| Status | Aceito |
| Data | Quinta-feira, 09:00 (conforme cabeçalho da transcrição; data calendário não registrada) |
| Decisores | Diego (Eng. Sênior Plataforma), Larissa (Tech Lead), Sofia (Segurança), Marcos (PM) |

## Contexto

Com o padrão outbox ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)) e o worker de envio em processo separado ([ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)), era preciso definir a semântica de entrega dos webhooks: o que a plataforma garante ao cliente sobre cada evento — que ele chega pelo menos uma vez, exatamente uma vez, ou no máximo uma vez?

Diego colocou a questão diretamente ([09:24]): "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado." Bruno perguntou como o cliente diferencia entregas duplicadas ([09:25]), e Sofia apontou o trade-off: "Isso joga responsabilidade pro cliente" ([09:25]).

## Decisão

Registrada por Larissa em [09:26]: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."

Em detalhe:

1. **Garantia de entrega at-least-once**: todo evento que entra na outbox será entregue pelo menos uma vez; entregas duplicadas são possíveis e admitidas por design ([09:24] Diego).
2. **Deduplicação pelo cliente via header `X-Event-Id`**: "A gente manda um event_id no header, X-Event-Id, com um UUID gerado quando o evento entra na outbox. É único por evento. Se o cliente recebeu duas vezes, ele dedupica pelo event_id do lado dele" ([09:25] Diego).
3. **Referência de mercado**: "é o padrão de mercado. Stripe faz assim, GitHub faz assim" ([09:25] Diego).
4. A responsabilidade do cliente será documentada de forma destacada no portal de desenvolvedor ([09:26] Marcos: "Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema").

## Alternativas Consideradas

### Exactly-once

Garantir que cada evento chegue exatamente uma vez. Descartada por Diego ([09:25]): "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos." Exactly-once exige protocolo transacional entre emissor e receptor (algo que a plataforma não controla, já que o receptor é o sistema do cliente), com complexidade desproporcional ao benefício.

### At-most-once (sem retry para evitar duplicatas)

Não discutida como opção viável na reunião: seria incompatível com a política de retry com backoff e DLQ já decidida ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) e com o requisito de negócio de que os clientes B2B sejam confiavelmente notificados das mudanças de status ([09:00] Marcos). Perder eventos silenciosamente é pior do que entregá-los duas vezes com identificador de dedup.

## Consequências

### Positivas

- Semântica simples de implementar e operar: o worker pode reenviar com segurança em qualquer cenário de falha, sem coordenação com o receptor ([09:25] Diego).
- `X-Event-Id` como UUID gerado na inserção na outbox dá ao cliente uma chave de idempotência estável por evento, válida através de todos os retries ([09:25] Diego).
- Alinhamento com o padrão de mercado (Stripe, GitHub), reduzindo atrito de integração para clientes que já consomem webhooks de outros provedores ([09:25] Diego).
- Combina naturalmente com o retry com backoff e a DLQ ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)): reenviar nunca corrompe o estado do cliente que deduplica corretamente.

### Negativas

- O cliente **precisa** implementar deduplicação por `X-Event-Id`; a garantia joga responsabilidade para o lado dele ([09:25] Sofia). Cliente que ignorar o header processará eventos duplicados.
- Duplicatas são possíveis por construção — por exemplo, se o worker morre entre o envio HTTP bem-sucedido e a marcação do evento como entregue na outbox, o evento será reenviado no próximo ciclo.
- Exige documentação destacada no portal de desenvolvedor para que a responsabilidade do cliente fique explícita ([09:26] Marcos).

## Referências

- Transcrição: [09:24] Diego — "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes."
- Transcrição: [09:25] Diego — "X-Event-Id, com um UUID gerado quando o evento entra na outbox"
- Transcrição: [09:25] Diego — "Stripe faz assim, GitHub faz assim"
- Transcrição: [09:25] Sofia — "Isso joga responsabilidade pro cliente"
- Transcrição: [09:26] Larissa — "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."
- Relacionado: [ADR-001](ADR-001-padrao-outbox-no-mysql.md) — o evento (e seu UUID) nasce na inserção na outbox
- Relacionado: [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) — retries são a principal fonte de duplicatas
- Relacionado: [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) — demais headers da entrega (`X-Signature`, `X-Timestamp`)
