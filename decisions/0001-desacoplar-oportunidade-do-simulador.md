# ADR 0001 — Campos dinâmicos da Oportunidade (desacoplar do simulador)

- **Status:** aceito (2026-06-30)
- **Contexto do produto:** Imova — CRM do corretor. A principal porta de entrada de leads hoje é o simulador público (`POST /opportunities`).
- **Decisão em uma frase:** a estrutura dos dados de enriquecimento da oportunidade deixa de ser **código** e passa a ser **dado editável** (campos personalizados / _custom properties_ agrupadas em seções), no padrão de CRMs como HubSpot/Pipedrive.
- **⚠️ Atualização (ver [ADR 0002](0002-crm-portavel-multi-tenant.md)):** decidimos que o CRM será um **serviço standalone multi-tenant**. Por isso, todos os modelos deste ADR (`FieldSection`, `FieldDefinition`, `Opportunity`) ganham `tenantId` e suas chaves únicas passam a ser **compostas por tenant** (ex.: `@@unique([tenantId, key])` em vez de `key @unique`). O restante do desenho (JSONB, promoção, tipos, seções) permanece válido.

---

## 1. Problema

A entidade `Opportunity` guarda, na mesma linha, campos de três naturezas, e todos são **colunas de primeira classe** espelhadas campo a campo no front. Um único campo atravessa 6 camadas (`simulador → CreateOpportunityDto → coluna Prisma → mapOpportunity → interface → template`), então qualquer mudança é migration + DTO + mapper + tipos + UI.

Além do acoplamento, há **mistura de naturezas**:

- **Intrínseco da oportunidade** (o motor do funil depende): `status`, `temperature`.
- **Enriquecimento/qualificação** (dado que enriquece, mas que o Michel quer remodelar com autonomia): "já tem imóvel em vista?", "aprovação de crédito", "prazo de compra", e **todo** o quadro "Preferências do imóvel".
- **Captura do simulador** (imutável, um instante T): `income`, `downPayment`, `propertyValue`, `fgts`, `amortization`, `buyerType`, `uf`, `city`.

Hoje os campos de qualificação/preferência estão **hardcoded** no drawer e no schema. Mudar a estrutura deles exige código/deploy. Isso é o oposto do que se quer.

---

## 2. Requisitos (o que a solução tem que garantir)

1. **Modelável como dado:** criar/editar/reordenar campos e seções sem código nem deploy.
2. **Separação de naturezas:** `status`/`temperature` são da oportunidade; qualificação e preferências são campos dinâmicos.
3. **CRM não depende dos campos dinâmicos por padrão** — só quando o Michel decidir usar um deles como base (filtro, agregação, coluna, gráfico).
4. **Consultável e filtrável no banco (não em memória)** — filtro/agregação rodam em SQL.
5. **Proveniência preservada** — o que o lead enviou no simulador não se perde quando o corretor edita.

---

## 3. Arquitetura em camadas

1. **Campos intrínsecos** — em coluna, o motor do funil depende deles: `status`, `temperature`, `boardOrder`, `lossReason`, `interestUf`/`interestCity` (roteamento RMSP + filtro), `propertyValue` (exibição/ordenação), `contactId`, datas.
2. **Definições de campo** (`FieldDefinition`) — dado: `key`, `label`, `type`, `options`, `section`, `order`, `indexed`, `archived`.
3. **Seções** (`FieldSection`) — os "quadros" do drawer, também dado: `key`, `label`, `order`.
4. **Valores** — em `Opportunity.fields` (JSONB), `{ chave: valor }`.
5. **Captura do simulador** — `Opportunity.capture` (JSONB), snapshot imutável (a fonte).

O drawer **renderiza seções → campos** a partir das definições, com um renderizador genérico por tipo. Adicionar/remover/reordenar campo = editar dado.

---

## 4. Modelo de dados

```prisma
model Opportunity {
  id            String   @id @default(uuid())
  // ── intrínsecos (motor do funil) ──
  status        String   @default("Lead")
  temperature   String   @default("Sem classificação")
  boardOrder    Float    @default(0)
  lossReason    String   @default("")
  interestUf    String   @default("")   // onde quer comprar — roteamento RMSP + filtro
  interestCity  String   @default("")
  propertyValue Int      @default(0)     // denormalizado da captura (exibição/ordenação)
  // ── captura imutável do simulador (a fonte) ──
  capture       Json     @default("{}")  // { income, downPayment, fgts, amortization, buyerType, ... }
  // ── valores dos campos dinâmicos ──
  fields        Json     @default("{}")  // { creditApproval: 'Sim', propertyType: 'Casa', ... }
  contactId     String
  contact       Contact  @relation(fields: [contactId], references: [id])
  activities    OpportunityActivity[]
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  @@index([contactId])
  @@map("opportunities")
}

model FieldSection {
  id        String            @id @default(uuid())
  key       String            @unique           // 'qualificacao', 'preferencias'
  label     String                              // 'Qualificação'
  order     Int               @default(0)
  fields    FieldDefinition[]
  @@map("field_sections")
}

model FieldDefinition {
  id        String       @id @default(uuid())
  key       String       @unique               // chave no JSONB: 'creditApproval'
  label     String                             // 'Aprovação de crédito'
  type      String                             // text|textarea|number|money|select|multiselect|boolean|date
  options   Json         @default("[]")        // para select/multiselect
  sectionId String
  section   FieldSection @relation(fields: [sectionId], references: [id])
  order     Int          @default(0)           // ordem dentro da seção
  indexed   Boolean      @default(false)       // promoção (ver §6)
  archived  Boolean      @default(false)       // soft delete — não apaga valores históricos
  createdAt DateTime     @default(now())
  updatedAt DateTime     @updatedAt
  @@map("field_definitions")
}
```

### Catálogo de tipos (v1)

`text` · `textarea` · `number` · `money` · `select` · `multiselect` · `boolean` · `date`.
(Extensível depois: `phone`, `url`, `relation`, etc.)

---

## 5. Decisão de armazenamento: JSONB (não EAV, não colunas)

**Escolhido: valores em JSONB no `Opportunity`.**

Ponto-chave que fecha o requisito #4: **`jsonb` no Postgres é consultado no banco, não em memória.**

```sql
SELECT * FROM opportunities WHERE fields->>'creditApproval' = 'Sim';
SELECT fields->>'creditApproval', count(*) FROM opportunities GROUP BY 1;
```

Ou seja, filtro/agregação em SQL desde o dia 1, sem nenhuma promoção.

Por que **não EAV** (tabela de valores): também filtra no banco, mas filtrar por N campos = N self-joins, valor como texto (tipagem ruim) e **agregação sofrida** — anti-padrão para relatório. JSONB é direto.

Por que **não coluna por campo**: é justamente o acoplamento que estamos removendo.

---

## 6. "Promoção" = performance, não capacidade

Como filtrar/agregar já funciona em SQL por padrão, promover um campo **não liga a consulta — só acelera**:

- **Não promovido (`indexed = false`, padrão):** valor no JSONB, filtrável por seq scan. No volume atual (dezenas/centenas), é instantâneo.
- **Promovido (`indexed = true`):** cria índice na chave —
  `CREATE INDEX ON opportunities ((fields->>'creditApproval'));` (ou GIN para busca ampla).
  Para relatório pesado, pode evoluir para uma _generated column_ tipada e indexada, filtrável/ordenável como coluna normal.

Isso materializa o requisito #3: o CRM só "se baseia" no que o Michel promover; o que muda ao promover é velocidade, não a possibilidade.

---

## 7. Simulador: snapshot separado

A captura do simulador continua um **snapshot imutável** em `Opportunity.capture` (JSONB). O bloco "Dados do simulador" do drawer lê o snapshot. Os valores podem **alimentar** campos dinâmicos (ex.: copiar para `fields` na criação, se algum campo espelhar o simulador), mas o registro original fica preservado à parte — proveniência garantida (requisito #5).

O simulador pode evoluir seus campos sem migration: novos dados entram no `capture` JSONB.

---

## 8. Renderização e API

- **Front:** `GET /field-definitions` (com seções) → o drawer monta as seções e, por tipo, um `FieldRenderer` (`<select>`, `<input money>`, `<checkbox>`, etc.). Valores salvos com `PATCH /opportunities/:id` gravando em `fields`.
- **Validação:** um validador por tipo coage/valida o valor antes de gravar no JSONB (ex.: `money` → inteiro; `select` → dentro de `options`).
- **Backend:** `Opportunity` deixa de ter DTO campo-a-campo para qualificação/preferência; passa a aceitar um `fields` parcial validado contra as definições.

---

## 9. Plano faseado

### Fase 1 — Engine + renderização (definições via seed)

Sem tela de gestão ainda; as definições entram por seed/migração.

1. **Aditivo:** criar `FieldSection`, `FieldDefinition`, `Opportunity.fields`, `Opportunity.capture`, `interestUf`/`interestCity`. `prisma db push`.
2. **Seed:** criar as seções ("Qualificação", "Preferências do imóvel") e as definições espelhando os campos atuais (aprovação de crédito, imóvel em vista, prazo, tipo, finalidade, quartos, vagas, bairros, necessidades) com tipos apropriados.
3. **Backfill:** para cada oportunidade, montar `capture` a partir das colunas de simulador; montar `fields` a partir das colunas de qualificação/preferência; copiar `uf`/`city` → `interestUf`/`interestCity`.
4. **Cortar o código:** `create()` grava `capture` e `fields`; drawer renderiza seções dinâmicas + bloco intrínseco (status/temperature) + bloco simulador (do `capture`). Remove os quadros hardcoded.
5. **Roteamento:** `isInServiceArea` passa a ler `interestCity`/`interestUf`.
6. **Destrutivo:** remover as colunas antigas migradas (`income`, `downPayment`, `fgts`, `amortization`, `buyerType`, `uf`, `city`, `hasProperty`, `hasCredit`, `deadline*`, `propertyType`, `purpose`, `bedrooms`, `parkingSpots`, `neighborhoods`, `needs`). `prisma db push --accept-data-loss` (dados já em `capture`/`fields`).

### Fase 2 — Tela de gestão

`/admin` para criar/editar/reordenar campos e seções, escolher tipo/opções e **promover** (que cria o índice). Autonomia total sem código.

---

## 10. Riscos e atenções

- **Roteamento RMSP** hoje usa `city`/`uf` → migrar para `interestCity`/`interestUf` no mesmo passo.
- **`propertyValue` denormalizado:** fonte de verdade fica no `capture`; manter a coluna em sincronia na criação (e se um dia for editável, definir quem manda).
- **Promoção via DDL a partir do app:** criar índice em runtime exige cuidado/permissão; na v1 a promoção pode ser um passo manual/administrativo antes de virar botão na tela.
- **Arquivar campo ≠ apagar valor:** `archived` só esconde da UI; os valores continuam no JSONB (histórico preservado).
- **Validação de tipos** dos campos dinâmicos precisa existir no backend (não confiar só no front).
- **Gráficos do CRM** (status/temperatura) não mudam — são intrínsecos. Gráficos sobre campos dinâmicos só depois de promovidos.

---

## 11. Alternativas consideradas (resumo)

- **EAV (tabela de valores):** consultável, mas complexo e ruim para agregação. Rejeitado.
- **JSON puro sem promoção/índice:** simples, mas sem caminho para performance de relatório. Coberto pelo híbrido JSONB.
- **Só camada de adaptação (adapter no código, schema igual):** alivia acoplamento de código, mas não dá modelagem dinâmica nem resolve mistura de naturezas. Insuficiente para o requisito #1.
- **Manter como está:** rejeitado (todos os problemas da §1 permanecem).
