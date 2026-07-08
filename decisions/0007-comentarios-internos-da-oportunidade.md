# ADR 0007 — Comentários internos da oportunidade

- **Status:** aceito (2026-07-08)
- **Relação:** complementa atividades (ADR 0002) e histórico (ADR 0006).
- **Decisão em uma frase:** cada oportunidade ganha uma **thread de comentários internos** da equipe (`OpportunityComment`), numa aba própria no drawer, separada das atividades (ações de CRM) e do histórico (eventos do sistema).

---

## 1. Contexto

As `OpportunityActivity` (nota, ligação, follow-up) são ações de CRM com tipo/agenda; o `OpportunityEvent` é log automático. Faltava um espaço de **conversa livre da equipe** sobre a oportunidade — comentários internos, ao estilo de um chat/discussão.

## 2. Decisão

- Novo model **`OpportunityComment`** (tabela `opportunity_comments`): `body`, `authorId` (dono), `author` (nome congelado p/ exibição), `createdAt`, `updatedAt`.
- **Autoria:** apenas o autor (`authorId`) pode **editar/excluir** o próprio comentário — enforcement no backend (`ForbiddenException`); o front esconde as ações de terceiros. Autor vem de `@CurrentUser()`.
- **Endpoints:** `POST/PATCH/DELETE /opportunities/:id/comments[/:commentId]` — todos retornam a oportunidade com `withRelations` (comentários em ordem cronológica asc, como chat).
- **Front:** aba **"Comentários"** (Atividades / Comentários / Histórico / Dados) no `OpportunityDrawer`: lista rola acima, composer fixo embaixo (Ctrl/⌘+Enter envia), avatar por autor, edição inline e marca "editado".

## 3. Consequências

- Colaboração da equipe registrada por oportunidade, sem misturar com ações de CRM nem com o log automático.
- Migração **aditiva** (só cria a tabela) — sem backfill.
- Comentários não geram eventos no histórico (são um canal à parte); se um dia for desejável, o gancho é simples.
