# ADR 0009 — Checklist de tarefas da oportunidade

- **Status:** aceito (2026-07-09)
- **Relação:** complementa o CRM (ADR 0002/0004) e o histórico de eventos (ADR 0006).
- **Decisão em uma frase:** cada oportunidade tem uma **checklist de tarefas** simples (título, concluída ou não, data de conclusão); marcar/desmarcar gera **evento no histórico**.

---

## 1. Contexto

A corretora precisa de um checklist leve de pendências por oportunidade (ex.: "pedir comprovante de renda", "agendar visita"), separado das **Atividades** (timeline de follow-up com data/tipo/notas) e dos **Comentários** (discussão interna).

## 2. Decisão

**Modelo — `OpportunityTask` (1 oportunidade → N tarefas):**
- Campos: `title`, `done` (bool), `completedAt` (nullable), `createdAt`. Nada de responsável/prazo por ora (deliberadamente simples).
- `onDelete: Cascade` da oportunidade.
- Ordenação: `createdAt` asc (mais antiga no topo).

**API (sob `JwtAuthGuard`, aninhada na oportunidade):**
- `POST /opportunities/:id/tasks` `{ title }`
- `PATCH /opportunities/:id/tasks/:taskId` `{ title?, done? }` — ao alternar `done`, seta/limpa `completedAt`.
- `DELETE /opportunities/:id/tasks/:taskId`
- Toda mutação retorna a oportunidade com `withRelations` (o front remapeia e emite `updated`).

**Histórico:** marcar/desmarcar gera `OpportunityEvent` (ADR 0006) `task_completed` / `task_reopened`, com o `title` no `data`. Criar/renomear/excluir tarefa **não** gera evento (só o toggle, conforme pedido).

**Front:**
- `OpportunityTask` no `opportunityModel`; `Opportunity.tasks` vem no payload.
- Seção **Tarefas** na aba **"Oportunidade"**, entre **Descrição** e **Documentos**: checkbox p/ concluir, contador `concluídas/total`, "Concluída em {data}", adicionar por input e excluir no hover.

## 3. Consequências

- Checklist objetivo por oportunidade, sem confundir com Atividades (follow-up) nem Comentários.
- Migração **aditiva** (cria a tabela `opportunity_tasks`).
- Extensível no futuro (responsável, prazo, ordem manual) sem quebrar o shape atual.
