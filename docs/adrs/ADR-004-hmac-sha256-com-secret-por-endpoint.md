# ADR-004: HMAC-SHA256 com secret por endpoint

| Campo | Valor |
|---|---|
| Status | Aceito |
| Data | Quinta-feira, 09:00 (conforme cabeçalho da transcrição; data calendário não registrada) |
| Decisores | Sofia (Segurança), Diego (Eng. Sênior Plataforma), Larissa (Tech Lead), Marcos (PM) |

## Contexto

O Sistema de Webhooks de Notificação de Pedidos envia eventos com dados de pedidos para endpoints HTTP fora da nossa infraestrutura. O cliente receptor precisa de duas garantias: autenticidade (a requisição veio realmente da plataforma) e integridade (o payload não foi adulterado no caminho). Como colocou Sofia ([09:19]): "a gente tá expondo eventos com dados de pedidos pra um endpoint fora da nossa infra. O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio."

Há ainda o risco operacional de vazamento de secret pelo lado do cliente — já aconteceu: "A gente já teve cliente que vazou secret em log de aplicação dele uma vez" ([09:22] Diego). O desenho precisa limitar o raio de explosão de um vazamento e permitir recuperação sem downtime de integração.

## Decisão

Formalizada por Sofia em [09:22]: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h."

Em detalhe:

1. **Assinatura HMAC-SHA256 sobre o corpo do request**, enviada no header `X-Signature`. "A gente assina o payload com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo X-Signature" ([09:20] Sofia). O algoritmo é SHA-256 porque "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).
2. **Secret única por endpoint de webhook**, não global da plataforma ([09:21] Sofia). A secret é gerada pela plataforma e devolvida ao cliente no momento da criação do webhook: "secret é gerada pela gente e devolvida na criação" ([09:31] Marcos).
3. **Rotação de secret com grace period de 24h**: o cliente pede uma nova secret pela API; "Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele. Depois disso, a antiga morre" ([09:21] Sofia).
4. **TLS obrigatório**: a URL do webhook tem que ser `https`; cadastro com `http` é recusado com erro de validação no schema Zod ([09:23] Sofia).
5. **Header `X-Timestamp`** com o timestamp do envio, "pra cliente conseguir detectar replay attack se quiser" ([09:44] Diego). Acompanha os demais headers do envio: `X-Event-Id`, `X-Signature` e `X-Webhook-Id` ([09:44] Diego/Sofia).

## Alternativas Consideradas

### Secret global da plataforma

Uma única secret compartilhada por todos os clientes/endpoints. Descartada por Sofia em [09:21]: "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo." O raio de explosão de um vazamento seria toda a base de clientes, e a rotação exigiria coordenar todos os clientes simultaneamente.

### Rotação sem grace period (troca imediata)

Trocar a secret e invalidar a antiga na hora quebraria a validação de assinatura do cliente até que ele atualizasse os sistemas dele. O grace period de 24h com as duas secrets válidas em paralelo foi o mecanismo escolhido justamente "pra ele ter tempo de migrar os sistemas dele" ([09:21] Sofia).

## Consequências

### Positivas

- Cliente valida autenticidade e integridade de cada entrega com algoritmo padrão de mercado e biblioteca disponível em qualquer stack ([09:20] Sofia).
- Vazamento de uma secret compromete apenas um endpoint, não a plataforma inteira ([09:21] Sofia).
- Recuperação de vazamento sem downtime de integração, via rotação com grace period ([09:21] Sofia) — cenário real já vivido ([09:22] Diego).
- `X-Timestamp` habilita defesa contra replay attack do lado do cliente ([09:44] Diego).
- Rejeição de URLs `http` no cadastro elimina tráfego de payload assinado em canal não criptografado ([09:23] Sofia).

### Negativas

- Gestão de secrets por endpoint: geração, armazenamento e ciclo de vida de uma secret para cada webhook cadastrado, em vez de uma configuração única.
- Durante a rotação existe uma janela de 24h em que duas secrets são válidas simultaneamente para o mesmo endpoint — a verificação/assinatura precisa lidar com esse estado duplo, e a expiração da secret antiga precisa ser controlada.
- A responsabilidade de verificar a assinatura (e o `X-Timestamp`) é do cliente; a plataforma não consegue impor isso.

## Referências

- Transcrição: [09:19] Sofia — "o cliente tem que conseguir validar que a requisição veio realmente da gente"
- Transcrição: [09:20] Sofia — "HMAC-SHA256 é o padrão de mercado"
- Transcrição: [09:21] Sofia — "Senão se vaza uma, vaza tudo"
- Transcrição: [09:22] Sofia — "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h"
- Transcrição: [09:23] Sofia — "URL do webhook tem que ser https [...] é só uma validação no schema Zod"
- Transcrição: [09:31] Marcos — "secret é gerada pela gente e devolvida na criação"
- Transcrição: [09:44] Diego — "X-Timestamp com o timestamp do envio (pra cliente conseguir detectar replay attack se quiser)"
- Relacionado: [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) — headers de entrega (`X-Event-Id`)
- Relacionado: [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) — validação Zod segue o padrão do projeto
