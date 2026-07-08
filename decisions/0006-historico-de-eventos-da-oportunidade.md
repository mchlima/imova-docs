# ADR 0006 — Histórico de alterações/movimentações da oportunidade

- **Status:** aceito (2026-07-08)
- **Relação:** complementa o CRM (ADR 0002/0004) e o estágio por id (ADR 0005).
- **Decisão em uma frase:** cada oportunidade ganha um **log imutável de eventos do sistema** (`OpportunityEvent`), separado das atividades manuais (`OpportunityActivity`), exibido numa aba **"Histórico"** no drawer.

---

## 1. Contexto

As `OpportunityActivity` são registros **manuais** (ligação, nota, follow-up), editáveis e deletáveis. Faltava rastrear **o que o sistema mudou** — movimentações de estágio (inclusive por drag no kanban), repasses entre pipelines, ganho/perdido, responsáveis, temperatura, criação e edição de campos — de forma automática e auditável.

## 2. Decisão

- Novo model **`OpportunityEvent`** (tabela `opportunity_events`): `type`, `data` (JSON, payload por tipo), `author` (nome do usuário; `"Sistema"` para ingestão/captura), `createdAt`. Imutável (sem update/delete pela aplicação); cai em cascata ao excluir a oportunidade.
- **Rótulos "congelados":** o `data` guarda os rótulos no momento do evento (`fromStageLabel`, `toPipelineLabel`, etc.), então o histórico permanece legível mesmo após renomear/excluir estágios ou pipelines.
- **Tipos:** `created`, `stage_changed`, `won`, `lost`, `pipeline_changed`, `assignees_changed`, `temperature_changed`, `fields_updated`. Ganho/perdido são detectados a partir das flags `isWon`/`isLost` do estágio de destino (o `lost` guarda o `lossReason`).
- **Onde é gerado (backend, `OpportunitiesService`):**
  - `createManual`/`ingest` → `created`.
  - `update` → diffa estado anterior × novo: estágio (→ `stage_changed`/`won`/`lost`), temperatura, responsáveis (added/removed por nome), campos (rótulos alterados).
  - `moveToPipeline` → `pipeline_changed`.
  - `reorder` (drag no kanban) → `stage_changed`/`won`/`lost` por card que trocou de coluna.
  - O autor vem de `@CurrentUser()` no controller.
- **Entrega:** os eventos vêm embutidos na oportunidade (`withRelations.events`, mais recente primeiro) — sem endpoint novo.
- **Front:** aba **"Histórico"** (Atividades / Histórico / Dados) no `OpportunityDrawer`, timeline read-only (ícone + texto + autor · data).

## 3. Consequências

- Auditoria e rastreabilidade das movimentações sem poluir as atividades manuais.
- Migração é **aditiva** (só cria a tabela) — sem backfill; oportunidades antigas simplesmente não têm eventos anteriores à feature.
- Edição de campos é registrada como lista de rótulos alterados (sem valores antigos/novos, para não vazar dados sensíveis nem inflar o log). Se um dia for preciso o valor, o `data` comporta a extensão.
