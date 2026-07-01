# ADR 0002 — CRM como serviço portável multi-tenant

- **Status:** aceito (2026-06-30)
- **Relação:** amplia e reenquadra o [ADR 0001](0001-desacoplar-oportunidade-do-simulador.md) (campos dinâmicos).
- **Decisão em uma frase:** o CRM deixa de ser uma feature da Imova e passa a ser um **produto próprio** — um **serviço standalone com API**, **multi-tenant**, que qualquer projeto integra por HTTP; extraível a qualquer momento sem reescrita.

---

## 1. Objetivo

Poder, a qualquer momento, **desacoplar o CRM deste projeto e usá-lo com outros**. A Imova passa a ser apenas **um cliente** do CRM (um tenant), assim como projetos futuros.

Duas decisões-alvo tomadas:

- **Distribuição:** serviço standalone + API (backend, admin e banco próprios). Cada projeto integra por HTTP/webhook, independente de stack.
- **Tenancy:** uma instância serve **vários projetos** (multi-tenant), com escopo de tenant em toda entidade, funil e campos.

---

## 2. Princípio-mestre: dependência de mão única

> O núcleo do CRM **nunca** importa nada do domínio "imobiliária". A Imova depende do CRM; o CRM não conhece a Imova.

| Fica no **núcleo do CRM** (genérico, portável) | Sai para a **Imova** ou vira **config por tenant** |
|---|---|
| Contato, Oportunidade (negócio num funil), Atividades/histórico | Simulador (fonte de captura) |
| Campos dinâmicos (ADR 0001) | Regras RMSP / ITBI / roteamento |
| Funil/estágios, temperatura, motivos de perda (como **dado**) | Guias/CMS, site, conteúdo |
| Follow-up, dashboards genéricos | Marca "Imova" |
| Ingestão genérica, automações por tenant | Mapeamento simulador→campos |
| Auth/identidade própria | — |

Qualquer coisa específica de imobiliária que hoje esteja no core é dívida a pagar.

---

## 3. Multi-tenancy

### 3.1 Modelo

```prisma
model Tenant {
  id        String   @id @default(uuid())
  slug      String   @unique          // 'imova'
  name      String
  apiKeys   ApiKey[]                   // para ingestão/integração
  createdAt DateTime @default(now())
  @@map("tenants")
}

model ApiKey {
  id        String   @id @default(uuid())
  tenantId  String
  tenant    Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  hash      String                     // guardar só o hash da chave
  label     String   @default("")
  revokedAt DateTime?
  createdAt DateTime @default(now())
  @@index([tenantId])
  @@map("api_keys")
}
```

### 3.2 Regras

- **`tenantId` em toda entidade** do core: `Contact`, `Opportunity`, `OpportunityActivity`, `FieldSection`, `FieldDefinition`, `Pipeline`, `Stage`, listas de config.
- **Uniques compostos por tenant:** `@@unique([tenantId, key])`, `@@unique([tenantId, slug])`, dedup de e-mail **por tenant** (dois projetos podem ter o mesmo contato sem colidir).
- **Escopo obrigatório em toda query** — nenhuma leitura/escrita sem `tenantId`. Ideal: forçar via camada (middleware Prisma / repositório que injeta o tenant do contexto de auth), pra não depender de disciplina manual.
- **Auth por tenant:** usuários do CRM pertencem a um tenant; sessão carrega o `tenantId`; ingestão externa autentica por **API key** do tenant.

---

## 4. Funil como dado (pipeline configurável)

Hoje os status (`Lead`, `Contatar`, `Qualificar`, `Repassado`, `Perdido`, `Nutrição`) estão hardcoded em front e back. Para ser portável, viram **dado por tenant**:

```prisma
model Pipeline {
  id        String  @id @default(uuid())
  tenantId  String
  key       String                    // 'default'
  label     String
  stages    Stage[]
  @@unique([tenantId, key])
  @@map("pipelines")
}

model Stage {
  id          String  @id @default(uuid())
  tenantId    String
  pipelineId  String
  key         String                  // 'lead', 'contatar'...
  label       String
  order       Int     @default(0)
  inKanban    Boolean @default(true)  // aparece no kanban? (final/nutrição podem ficar fora)
  isWon       Boolean @default(false)
  isLost      Boolean @default(false)
  @@unique([tenantId, pipelineId, key])
  @@map("stages")
}
```

- **Temperatura** e **motivos de perda** também viram listas de config por tenant (ou campos dinâmicos do sistema).
- O kanban, os badges e os gráficos passam a ler os estágios do tenant, não constantes.
- Seed inicial: os estágios atuais da Imova (incluindo `Nutrição` fora do kanban).

---

## 5. Ingestão genérica

O simulador deixa de ter endpoint dedicado com campos de imobiliária. Entra um **endpoint genérico**, autenticado por API key do tenant:

```
POST /ingest         (Authorization: Bearer <api-key do tenant>)
{
  "source": "simulador",
  "contact": { "name": "...", "channels": [{ "type": "email", "value": "..." }] },
  "capture": { "income": 5000, "propertyValue": 300000, ... },   // snapshot imutável
  "fields":  { "propertyType": "Casa" },                          // valores de campos dinâmicos (opcional)
  "stageKey": "lead"                                              // opcional; senão, estágio default
}
```

- O CRM resolve o contato (**dedup por tenant**), cria a oportunidade, guarda `capture` (snapshot) e `fields`.
- **Regras de negócio (RMSP)** ficam **fora do core**: ou a Imova calcula o `stageKey` antes de postar, ou vira uma **automação por tenant** (config declarativa) — nunca código no núcleo.
- Qualquer projeto/stack integra postando nesse formato.

---

## 6. Auth e admin próprios

- O CRM tem **auth próprio** (usuários, sessões, escopo de tenant) — hoje o admin usa o JWT da Imova; na extração, o CRM passa a dono da identidade.
- O **admin do CRM** (telas de oportunidades, contatos, follow-up, dashboard, gestão de campos/funil) é do produto CRM. Hoje vivem no `imova-site`; devem ser mantidas **atrás de uma fronteira de API limpa** para migrarem para um admin próprio quando extrair.

---

## 7. Estratégia da costura — agora vs. depois

Não precisamos extrair já. Precisamos **manter a costura limpa** para extrair barato depois.

**Agora (dentro do repo atual):**
- CRM como **módulo isolado** no `imova-backend`, com **zero imports** do domínio Imova.
- `tenantId` em todas as entidades desde o primeiro dia (Imova = tenant `imova`).
- Fronteira em **formato de API** (a Imova fala com o CRM como se fosse remoto, mesmo in-process).
- Funil/estágios e campos como dado (ADR 0001 + §4).
- Ingestão genérica (§5) — o simulador já posta no formato final.

**Depois (quando quiser):**
- Promover o módulo a **serviço/repo próprio** + banco próprio + admin próprio. Como a costura já é API + multi-tenant, é *lift-and-shift*.

---

## 8. Impacto no ADR 0001

- `FieldSection`, `FieldDefinition`, `Opportunity` ganham `tenantId`.
- `key @unique` → `@@unique([tenantId, key])`; dedup de e-mail passa a ser por tenant.
- O resto do 0001 (JSONB, promoção = índice, catálogo de tipos, seções dinâmicas, snapshot do simulador) segue igual.

---

## 9. Plano faseado (revisado)

- **Fase 0 — Fundação portável:** `Tenant`/`ApiKey`; `tenantId` + escopo forçado em tudo; **funil como dado** (Pipeline/Stage) com seed dos estágios da Imova; **ingestão genérica** `/ingest`; front (kanban/badges/gráficos) lendo estágios do tenant.
- **Fase 1 — Campos dinâmicos (ADR 0001):** engine + renderização a partir de definições (seed), com `tenantId`.
- **Fase 2 — Telas de gestão:** campos, seções, funil e tenants via UI; promoção de campo (índice).
- **Fase 3 — Extração (quando quiser):** serviço/repo/admin próprios + auth do CRM.

---

## 10. Riscos e atenções

- **Isolamento multi-tenant é crítico:** um vazamento de escopo mistura dados de projetos. Forçar o `tenantId` na camada (não confiar em disciplina) e ter testes de isolamento.
- **Custo transversal:** todo query/serviço passa a carregar tenant — vale investir na infraestrutura de escopo cedo (Fase 0), antes de haver muito código.
- **Auth em transição:** enquanto o admin usar o JWT da Imova, mapear esse usuário para o tenant `imova`; desacoplar de fato só na Fase 3.
- **Migração dos dados atuais:** tudo que existe hoje entra como tenant `imova` (backfill de `tenantId`).
- **RMSP não pode vazar pro core:** manter como cálculo da Imova (pré-`/ingest`) ou automação declarativa por tenant.
- **Regressão de UI:** kanban/gráficos deixam de usar constantes; garantir paridade visual ao ler estágios do tenant.
