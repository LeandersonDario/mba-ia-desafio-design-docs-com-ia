# Da Reunião ao Documento: Design Docs Gerados por IA — Processo de Produção

> Entrega do desafio do MBA: transformar a transcrição de uma reunião técnica (`TRANSCRICAO.md`) e o código de um OMS existente em um pacote completo de design docs, usando IA como ferramenta principal de produção. O enunciado original está no [repositório base do desafio](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js/TypeScript/Express/Prisma/MySQL) decidiu, em uma reunião de ~55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança, construir um **Sistema de Webhooks de Notificação de Pedidos** — mas nada foi registrado além da transcrição da call. Minha tarefa foi produzir, a partir dessa transcrição e do código existente, a documentação completa da feature: PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade, em nível acionável o suficiente para o time de engenharia começar a implementar.

A restrição central do desafio é a **rastreabilidade**: nenhum requisito, decisão ou restrição pode ser inventado — tudo precisa ter origem identificável na transcrição (timestamp + falante) ou no código-fonte (caminho de arquivo). Igualmente importante é o que *não* entra: a reunião descartou e adiou várias ideias (e-mail de falha, dashboard, exactly-once, Redis…), e elas precisavam aparecer como fora de escopo ou alternativas descartadas, nunca como requisito. O papel exigido foi o de maestro: dirigir a IA com prompts específicos, revisar criticamente cada saída e iterar até a consistência.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code (web)** com o modelo **Claude Fable 5** | Ferramenta única de produção: exploração do código, análise da transcrição, geração e revisão de todos os documentos, versionamento (commits/push) |
| **Subagentes do Claude Code** (tipos `Explore` e `general-purpose`) | Trabalho paralelo com contexto isolado: 2 agentes de exploração (transcrição + código), 2 agentes de ADRs em paralelo, agentes dedicados para FDD, PRD e Tracker — cada um com prompt dirigido e escopo de arquivos disjunto |
| **Plugin `superpowers`** (skills `writing-plans`, `executing-plans`, `dispatching-parallel-agents`) | Metodologia: plano escrito e aprovado antes da execução, execução por tarefas com checkpoints, e padrão de despacho de agentes paralelos (um agente por domínio independente, prompts autocontidos, saída especificada) |
| **Modo Plan do Claude Code** | Fase de planejamento somente-leitura: exploração e desenho do plano antes de qualquer escrita |

## Workflow adotado

O trabalho seguiu um plano escrito e aprovado antes de qualquer documento ser gerado:

1. **Exploração paralela (2 subagentes simultâneos)** — um agente leu a `TRANSCRICAO.md` inteira e produziu um relatório estruturado (decisões com timestamps, alternativas descartadas, itens adiados, questões em aberto, requisitos, métricas); outro mapeou o código (módulos, `changeStatus`, máquina de estados, classes de erro, middlewares, schema Prisma, testes) e confirmou a ausência de qualquer mecanismo de eventos/webhooks. Esses dois relatórios viraram o **"context pack"**: o bloco de fatos com âncoras que alimentou todos os prompts seguintes.
2. **ADRs primeiro (2 subagentes em paralelo)** — as 7 decisões viram o esqueleto de tudo. Agente A: ADR-001 (outbox), ADR-002 (worker/polling), ADR-003 (retry/DLQ), ADR-007 (snapshot). Agente B: ADR-004 (HMAC), ADR-005 (at-least-once), ADR-006 (reuso de padrões, com referências a arquivos reais). Cada prompt trazia as âncoras esperadas e a instrução de **verificar cada timestamp contra o arquivo e corrigir o prompt se divergisse** — e os agentes de fato reportaram divergências (ver Iterações).
3. **RFC** — escrito na sessão principal, consolidando a proposta em nível de arquitetura, com alternativas descartadas, questões em aberto e links para os 7 ADRs. Conciso de propósito: o detalhe ficou para o FDD.
4. **FDD e PRD (2 subagentes em paralelo)** — ambos dependem só de ADRs+RFC, então rodaram simultaneamente: FDD com fluxos, modelos Prisma propostos, 7 contratos de endpoint, matriz `WEBHOOK_*` e a seção obrigatória de integração com 11 caminhos reais; PRD como consolidação de produto (RF-01..RF-11, RNFs, riscos com probabilidade × impacto × mitigação).
5. **Tracker (subagente)** — varredura dos documentos prontos, gerando a tabela de rastreabilidade com validação de cada âncora.
6. **Validação mecânica (script)** — um script shell extrai todos os `[hh:mm]` citados nos docs e confere que existem na transcrição; extrai todos os caminhos `src/`, `prisma/`, `tests/` citados e confere que existem no repositório (ignorando os propostos, como `src/worker.ts` e `src/modules/webhooks/*`); confere nomenclatura/contagem dos ADRs e as metas percentuais do Tracker.
7. **README (este arquivo) e revisão final** — checklist de critérios de aceite item por item antes do push final.

Commits e push foram feitos por documento, na branch de trabalho, ao fim de cada etapa.

## Prompts customizados

Prompt do agente de análise da transcrição (fase 1 — exploração dirigida, não "resuma a reunião"):

```text
Leia o arquivo TRANSCRICAO.md por completo (transcrição de reunião técnica de ~55 min
no formato [hh:mm] Nome: fala, sobre um Sistema de Webhooks de Notificação de Pedidos).

Produza um relatório estruturado com:
1. Participantes: nome e papel de cada um.
2. Decisões fechadas: cada decisão arquitetural tomada, com o(s) timestamp(s) e
   falante(s) onde ela aparece. Procure especialmente por: padrão Outbox no MySQL,
   política de retry com backoff e DLQ, autenticação HMAC-SHA256 com secret por
   endpoint, garantia at-least-once com X-Event-Id, worker em processo separado com
   polling, reuso dos padrões existentes do projeto — e quaisquer outras decisões.
3. Alternativas discutidas e descartadas: cada alternativa rejeitada, com timestamp,
   falante e o trade-off que motivou o descarte.
4. Itens explicitamente fora de escopo ou adiados para fases futuras.
5. Questões deixadas em aberto (levantadas mas não decididas).
6. Requisitos funcionais explícitos (mínimo esperado: 8), cada um com timestamp.
7. Requisitos não funcionais / restrições (segurança, performance, confiabilidade).
8. Ganchos com o código existente mencionados na reunião (arquivos, métodos como
   changeStatus, classes, padrões citados).
9. Métricas/metas quantitativas mencionadas (SLAs, percentuais, tempos).

Seja exaustivo nos timestamps — eles serão usados num tracker de rastreabilidade.
Cite sempre no formato [hh:mm] Nome.
```

Trecho do prompt dos agentes de ADR (fase 2 — âncoras pré-mapeadas + ordem de verificação contra a fonte):

```text
Antes de escrever, leia TRANSCRICAO.md para citar timestamps e falas EXATAS.
Não invente nada que não esteja na transcrição.

Arquivo: docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md
Decisão: 5 tentativas com backoff 1m/5m/30m/2h/12h (~15h de janela) e depois DLQ em
tabela separada webhook_dead_letter ([09:15–09:18] Diego/Larissa); reprocessamento
manual via POST /admin/webhooks/dead-letter/:id/replay com role ADMIN e log de
auditoria ([09:35–09:36] Sofia/Larissa).
Alternativas descartadas: 3 tentativas ([09:16] Bruno propôs; Diego: "3 é pouco",
houve cliente com manutenção de 2 horas); retry indefinido ([09:15] Diego — evento
pendurado pra sempre); DLQ como flag "failed" na própria outbox ([09:17–09:18]).

Formato MADR: Status, Contexto, Decisão, Alternativas Consideradas, Consequências
(positivas E negativas, trade-off explícito), Referências.
TODA afirmação factual ancorada em timestamp da transcrição — verifique o timestamp
real no arquivo antes de citar; se a âncora que passei divergir, corrija pela
transcrição e reporte a divergência.
```

Trecho do prompt do agente do Tracker (fase 5 — metas quantitativas verificáveis no próprio prompt):

```text
Para TRANSCRICAO, Localização = timestamp + falante no formato [09:17] Diego —
VALIDE cada um com grep na TRANSCRICAO.md: o timestamp E o falante têm que bater
com uma fala real que suporte o item. Para CODIGO, caminho real do arquivo.

Metas obrigatórias (verifique antes de terminar):
- Cobertura ≥80% dos itens identificáveis (todos os RFs, RNFs, fora de escopo,
  objetivos, riscos, alternativas, questões em aberto, ADRs, contratos, códigos
  WEBHOOK_*, fluxos, modelos e pontos de integração).
- ≥70% das linhas com Fonte = TRANSCRICAO.
- ≥5 linhas com Fonte = CODIGO.
Ao final, retorne qualquer item que você NÃO conseguiu ancorar — para eu corrigir
o documento de origem, não o tracker.
```

## Iterações e ajustes

Os principais momentos em que a saída precisou de correção (registrados durante a execução):

1. **Âncoras de timestamp imprecisas no context pack.** O relatório de exploração atribuía algumas decisões a intervalos ou falantes ligeiramente errados, e os agentes de ADR — instruídos a verificar antes de citar — corrigiram e reportaram: a formalização do outbox é de `[09:08] Larissa` (não do intervalo todo [09:06–09:08]); o "Decidido: HMAC-SHA256…" é da Sofia em `[09:22]` (não [09:21]); a formalização do at-least-once é da Larissa em `[09:26]` (Diego apenas propõe em [09:24–09:25]); as três falas do "snapshot. Decidido." estão todas em `[09:52]` (não [09:51–09:52]); a exigência de ADMIN + auditoria no replay é de `[09:36] Sofia` (não [09:35]).
2. **Fato comercial reescrito por cima da fonte.** No rascunho do RFC eu havia escrito que a Atlas "condicionou a renovação de contrato" à feature; relendo `[09:00] Marcos`, a fala real é que a Atlas "chegou a sugerir que… eles podem migrar pro nosso concorrente" até o fim do trimestre. O RFC e o PRD foram ajustados para a formulação suportada pela transcrição.
3. **Erro de classificação na matriz de erros do FDD.** O prompt pedia `WEBHOOK_PAYLOAD_TOO_LARGE` como HTTP 400; o agente detectou que o limite de 64KB se aplica ao payload **gerado pela plataforma** no envio pelo worker (não a um request do cliente da API), reclassificou como erro interno do worker (falha → retry/DLQ) e documentou a nuance. Também detectou que `WEBHOOK_NOT_FOUND` não pode ser subclasse do `NotFoundError` existente (que fixa o código `NOT_FOUND` sem parâmetro) e propôs subclasse direta de `AppError`.
4. **Cenário ilustrativo marcado como derivação.** O prompt do PRD sugeria um cenário "ERP do cliente atualiza logística ao receber SHIPPED"; a transcrição só suporta o exemplo de filtro "só quero saber quando vira SHIPPED e DELIVERED" (`[09:33] Marcos`). O agente manteve o cenário, mas marcado explicitamente como ilustrativo derivado — mesmo tratamento dado a tudo que é prática derivada e não decisão da reunião (ex.: suítes de teste propostas, percentil da métrica de latência).
5. **Data da reunião não inventada.** O cabeçalho da transcrição registra apenas "quinta-feira, 09:00", sem data de calendário. A primeira tentação (minha e dos agentes) era datar os documentos; a versão final declara explicitamente "data de calendário não registrada" em todos os metadados.
6. **Validação mecânica como rede de segurança.** Após cada lote, um script shell conferiu todos os timestamps e caminhos citados. Foi ele que separou os 4 caminhos de teste *propostos* (`tests/webhook*.test.ts`) dos reais — confirmando que o FDD os apresentava corretamente como "suíte nova" e não como arquivo existente.

No total, foram 4 ciclos principais de geração → revisão crítica → correção (exploração/context pack → ADRs → RFC/FDD/PRD → Tracker/validação), além dos ajustes pontuais acima.

## Como navegar a entrega

| Ordem sugerida | Arquivo | O que responde |
|---|---|---|
| 1 | [`docs/PRD.md`](docs/PRD.md) | Por que e o quê — problema, público, escopo, requisitos, riscos |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Como pretendemos resolver — proposta, alternativas, questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/) | Por que decidimos exatamente assim — ADR-001 a ADR-007 |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir em detalhe — fluxos, contratos, erros, integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada coisa — rastreabilidade item a item |

A fonte de tudo: [`TRANSCRICAO.md`](TRANSCRICAO.md) (não alterada) e o código em `src/`, `prisma/` e `tests/` (não alterado).
