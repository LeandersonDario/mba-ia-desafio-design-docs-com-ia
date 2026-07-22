# Architectural Decision Records

Este diretório armazena os ADRs (Architecture Decision Records) do Sistema de Webhooks de Notificação de Pedidos, no formato exigido pelo desafio: `ADR-NNN-titulo-em-kebab-case.md`.

Cada ADR segue o formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências) e ancora suas afirmações em falas da reunião (`TRANSCRICAO.md`, formato `[hh:mm] Nome`) ou em arquivos reais do código.

## Índice

| ADR | Decisão |
|---|---|
| [ADR-001](ADR-001-padrao-outbox-no-mysql.md) | Padrão Outbox no MySQL existente |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado com polling de 2s |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial e DLQ em tabela separada |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace de 24h |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação via X-Event-Id |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](ADR-007-payload-snapshot-na-insercao.md) | Payload como snapshot renderizado na inserção |
