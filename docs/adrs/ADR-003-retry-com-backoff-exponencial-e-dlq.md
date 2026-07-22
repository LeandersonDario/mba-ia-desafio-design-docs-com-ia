# ADR-003: Retry com backoff exponencial e DLQ em tabela separada

| Campo | Valor |
|---|---|
| Status | Aceito |
| Data | Quinta-feira, 09:00 (conforme cabeçalho da transcrição; data calendário não registrada) |
| Decisores | Larissa (Tech Lead), Diego (Eng. Sênior Plataforma), Bruno (Eng. Pleno Pedidos), Sofia (Segurança — controle de acesso do replay), Marcos (PM — validou a janela) |

## Contexto

Com a entrega feita por worker assíncrono ([ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)), é preciso definir o que acontece quando o endpoint do cliente está indisponível: "Se o cliente tá offline, o que a gente faz?" ([09:14] Larissa). Forças em jogo:

- Já houve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego).
- Um evento não pode ficar "pendurado pra sempre se o cliente sumiu" ([09:15] Diego).
- Falhas permanentes precisam ficar visíveis para debug e reprocessamento ([09:18] Diego).

## Decisão

**Retry com backoff exponencial, teto de 5 tentativas, e Dead Letter Queue (DLQ) persistida em tabela separada.**

- **5 tentativas**, com progressão de backoff **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas** — "Total de quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego). Formalizado em [09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h." Marcos validou a janela: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável" ([09:17]).
- Esgotadas as tentativas, o evento é considerado falha permanente e movido para a tabela **`webhook_dead_letter`**, separada da outbox, contendo "a payload, motivo da falha e timestamp" ([09:18] Diego).
- **Reprocessamento manual** via endpoint admin **`POST /admin/webhooks/dead-letter/:id/replay`**, que "recoloca na outbox como pendente" ([09:18] Diego). O endpoint exige **role ADMIN** e **log de auditoria** de quem executou o replay: "Mexer em fila de entrega de notificação não é coisa de operador. E o endpoint de admin tem que logar quem fez o replay, pra auditoria" ([09:36] Sofia). Confirmado em [09:36] Larissa: "role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe."
- Timeout de 10 segundos na chamada HTTP conta como falha e entra no ciclo de retry ([09:42] Diego).

## Alternativas Consideradas

### 3 tentativas (mais agressivo)

Proposta por Bruno: "3 não é melhor? Mais agressivo" ([09:16]). Rejeitada: "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada" ([09:16] Diego). Três tentativas não cobrem janelas reais de indisponibilidade já observadas.

### Retry indefinido com backoff

Citada por Diego como posição defendida por "algumas pessoas", mas descartada por ele mesmo: "isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu. Cinco já dá pra cobrir uma janela de até 12 ou 24 horas" ([09:15] Diego).

### DLQ como flag "failed" na própria outbox

Levantada por Larissa: "Faz numa tabela separada ou marca como 'failed' na própria outbox?" ([09:17]). Descartada em favor da tabela separada: "Eu fazia uma tabela webhook_dead_letter separada [...] Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento" ([09:18] Diego). Manter falhas permanentes na outbox sujaria a leitura dos pendentes pelo worker.

## Consequências

### Positivas

- Janela de ~15 horas de retentativas cobre indisponibilidades reais de clientes, inclusive manutenções planejadas de horas ([09:16]–[09:17] Diego).
- Nenhum evento fica em retry eterno; falha permanente tem destino definido e auditável ([09:15], [09:18] Diego).
- A `webhook_dead_letter` serve como evidência para debug e ponto de partida do reprocessamento ([09:18] Diego).
- Replay restrito a ADMIN, com trilha de auditoria ([09:36] Sofia/Larissa).

### Negativas

- Eventos que caem na DLQ dependem de intervenção manual — não há reprocessamento automático nem notificação ao cliente por email, que ficou fora de escopo desta fase ([09:37] Larissa: "Email tá fora de escopo dessa fase").
- Cliente indisponível por mais de ~15 horas perde a entrega automática do evento (aceito em [09:17] Marcos).
- Uma tabela adicional (`webhook_dead_letter`) para modelar e manter.

## Referências

- Transcrição: [09:15] Diego — "Backoff exponencial [...] depois de um teto de tentativas considera falha permanente e move pra DLQ."
- Transcrição: [09:16] Diego — "3 é pouco."
- Transcrição: [09:17] Larissa — "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h."
- Transcrição: [09:18] Diego — "uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp."
- Transcrição: [09:36] Sofia — "o endpoint de admin tem que logar quem fez o replay, pra auditoria."
- Relacionado: [ADR-001](ADR-001-padrao-outbox-no-mysql.md) — a outbox de onde os eventos falham e para onde o replay os devolve.
- Relacionado: [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) — o worker que executa as tentativas.
- Relacionado: [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) — replay implica possibilidade de entrega duplicada.
