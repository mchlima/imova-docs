# ADR 0004 — CRM com múltiplos boards (pipelines) por tenant

- **Status:** aceito (2026-07-08)
- **Relação:** estende o [ADR 0002](0002-crm-portavel-multi-tenant.md) (funil como dado, multi-tenant).
- **Decisão em uma frase:** um tenant passa a ter **N boards (pipelines)**, cada um com seus próprios estágios; a oportunidade **pertence a um board** e é **repassada** de um para outro — modelando "a pré-venda qualifica e joga no board da corretora".

---

## 1. Contexto

Havia **1 board** por tenant (o funil `default`). O Meu Revelar precisa de dois funis
distintos: **Qualificação** (onde todo lead do site cai e é qualificado — key `default`)
e **Corretagem** (funil ativo de vendas — key `corretora`). O fluxo real: a pré-venda
recebe o lead, qualifica e **repassa** para o pipeline da corretagem.

> Nomenclatura na UI: os boards são chamados de **"pipelines"**. Os identificadores
> internos (keys `default`/`corretora`) permanecem estáveis.

## 2. Decisão

- A `Opportunity` ganha **`pipelineId`** — vive em exatamente um board por vez.
- `Pipeline` ganha **`ownerUserId`** (dono do board — *ownership leve*) e **`order`**.
- **Repasse** = `POST /opportunities/:id/move-pipeline`: seta o board de destino e o
  1º estágio dele (no topo da coluna), opcionalmente (re)atribui responsáveis.
- Estágios continuam **dado por board** (`Stage.pipelineId`); keys **únicas entre boards**
  (ex.: `PerdidoVendas`) para o lookup global de cor/label por key não colidir.
- **Novas oportunidades** (site/simulador, ingestão, criação manual) caem no **pipeline padrão**
  (o de menor `order` = Qualificação). `stageKey`/ordenação são **escopados por board**.

## 3. Ownership leve (não bloqueia)

Cada board tem um dono opcional. A UI **abre no board do usuário logado** (se ele for dono
de algum) e, caso contrário, no primeiro board — mas **ninguém fica bloqueado** de navegar
nos outros boards. É organização, não permissão rígida. O dono é configurável em
`PATCH /pipelines/:id` (seletor ao lado das abas).

## 4. Navegação e escopo no admin

- **Menu:** `CRM › Pipelines › {Qualificação, Corretagem}` — os pipelines são sub-itens
  dinâmicos (de `GET /pipelines`); cada um linka `/admin/pipelines?board=<key>`.
- **Página `/admin/pipelines`** (antiga "Oportunidades"): abas de pipeline; lista, kanban e
  filtro de status escopados ao pipeline ativo. Botão **"Configurar pipeline"** →
  `/admin/configuracoes?tab=funnel&board=<id>` (dono + etapas).
- **CRM (funil) e Configurações (estágios/dono):** seletor de pipeline (padrão = Qualificação).
- **Repasse:** botão "Enviar para \<pipeline\>" no detalhe da oportunidade.

## 5. Migração

Aditiva/reversível: `pipelineId` nullable + backfill (todas as oportunidades → Qualificação),
`prisma db push`, e `seed-boards.ts` (idempotente) cria a Corretagem e renomeia o `default`
para "Qualificação". Nenhum status de oportunidade existente foi alterado.

## 6. Consequências

- O conceito de "board" abre caminho para outros funis por tenant (ex.: pós-venda, locação).
- Keys de estágio precisam ser únicas entre boards do mesmo tenant (convenção, não constraint).
- Edição de estágios da Corretora e atribuição de donos ficam disponíveis no admin.
