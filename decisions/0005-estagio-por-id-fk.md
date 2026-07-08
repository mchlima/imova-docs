# ADR 0005 — Estágio da oportunidade por id (FK), com id externo para integrações

- **Status:** aceito (2026-07-08)
- **Relação:** substitui o mecanismo de identidade de estágio do [ADR 0002](0002-crm-portavel-multi-tenant.md) e revisa o item de keys do [ADR 0004](0004-crm-multiplos-boards.md).
- **Decisão em uma frase:** a oportunidade passa a referenciar o estágio por **`Opportunity.stageId` (FK → `Stage.id`)** em vez da string `Opportunity.status` (que casava com `Stage.key`); a `key` derivada do nome é **removida** e cada estágio mantém **dois ids**: `id` (interno, alvo da FK) e **`externalId`** (estável, para integrações referenciarem).

---

## 1. Contexto

Até aqui `Opportunity.status` era uma **string solta** igual à `Stage.key` (`@@unique([tenantId, pipelineId, key])`) — não havia integridade referencial. Consequências:

- Renomear/editar um estágio (ou sua key) **órfãva** oportunidades: o `status` ficava sem coluna correspondente. Mitigamos com migração manual no delete/edit, mas a fragilidade era estrutural.
- A key era **derivada do nome** — pouco prática e sujeita a colisão; obrigava keys "únicas entre boards" como convenção (não constraint).

O usuário concluiu que **id é melhor que key, tanto interna quanto externamente**.

## 2. Decisão

- **`Opportunity.stageId`** (`String?`) com relação `stage Stage? @relation(fields:[stageId], references:[id])` + `@@index([stageId])`. A FK garante integridade: não é possível apontar para um estágio inexistente, nem apagar um estágio que ainda tem oportunidades.
- **`Stage.key` é removida.** A identidade do estágio é o **`id`** (uuid interno).
- **`Stage.externalId`** (`String @unique @default(uuid())`): id **estável** para integrações externas referenciarem o estágio — não muda ao renomear o estágio e é independente do id interno.
- **Contratos de API:**
  - Admin (interno): `stageId` em criar/editar/reordenar/mover (`POST/PATCH /opportunities`, reorder do kanban).
  - Integrações (externo): `stageExternalId` no `/ingest` e `/capture` (o service resolve `externalId → id` e, com isso, o pipeline do estágio). Ausente = primeiro estágio do board padrão.
- **Config de estágios:** sem campo de key; renomear (label) e recolorir não afetam as oportunidades (FK por id). O `externalId` é exibido **somente-leitura** ("ID externo") para copiar em integrações.
- **Excluir estágio:** com oportunidades, exige `moveTo` (estágio destino do mesmo pipeline) → `updateMany stageId` → delete. Sem isso a FK bloquearia o delete.

## 3. Migração (dados de produção)

Requer **backup**. Não-destrutiva até o passo final:

1. `prisma/migrate-stageid.ts` (SQL cru, idempotente): adiciona `opportunities.stageId` e `stages.externalId`; gera `externalId` (`gen_random_uuid()`) para estágios sem ele; faz o **backfill** de `stageId` casando `(tenantId, pipelineId, stages.key == opportunities.status)`; reporta órfãs (`stageId` nulo).
2. Inspecionar o relatório de órfãs.
3. `prisma db push --accept-data-loss`: cria a FK/`@@index`/`@unique` e **derruba** as colunas antigas `stages.key` e `opportunities.status`.

## 4. Consequências

- Integridade referencial de estágio (fim das oportunidades órfãs); renomear estágio é trivial e seguro.
- Duas identidades desacopladas: o id interno pode existir sem vazar para fora; integrações usam um id externo estável.
- `Pipeline.key` **permanece** por ora (uso futuro; ver ADR 0004) — este ADR trata apenas de `Stage`. Aplicar o mesmo padrão (id + externalId) ao `Pipeline` fica como evolução futura.
- `seed-boards.ts` (migração já executada do ADR 0004) referencia `Stage.key` e não roda mais no schema novo — mantido apenas como registro histórico.
