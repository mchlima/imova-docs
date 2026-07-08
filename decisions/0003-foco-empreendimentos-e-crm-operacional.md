# ADR 0003 — Foco em empreendimentos da construtora e CRM operacional

- **Status:** proposto (2026-07-01)
- **Relação:** aplica o [ADR 0001](0001-desacoplar-oportunidade-do-simulador.md) (campos dinâmicos) e o [ADR 0002](0002-crm-portavel-multi-tenant.md) (CRM portável). Não altera o núcleo do CRM; adiciona operação manual e uma camada de aquisição por empreendimento no site.
- **Decisão em uma frase:** o negócio deixa de ser "captação ampla via simulador" e passa a **atacar os empreendimentos de uma construtora específica** — com **LPs por empreendimento** capturando lead (simulador + formulário), uma **home neutra de vitrine**, e um **CRM operável à mão** (Jéssica trabalhando a base da Vanessa).

---

## 1. Contexto e mudança de modelo

Mudança de modelo de negócios:

- **Operação humana e imediata.** A **Jéssica** vai trabalhar leads que **já existem** — a base da **Vanessa** (corretora amiga). O CRM, até aqui, só sabia *receber* do simulador. Precisa passar a permitir **criar e editar contatos e oportunidades à mão**.
- **Foco estreito.** A Vanessa atende uma **construtora específica**. O alvo deixa de ser "todo mundo que quer financiar" e passa a ser **os empreendimentos dessa construtora**.
- **Aquisição por empreendimento.** Cada empreendimento ganha uma **LP dedicada**, que capta lead com **simulador + formulário de contato**.
- **Home neutra.** A home vira uma **vitrine** com **cards para as LPs dos empreendimentos**, **sem remover guias nem simulador**.
- **Risco declarado:** **canibalização de SEO** entre LPs, guias, simulador e home.

O que **não** muda: o núcleo do CRM continua genérico e domain-free (ADR 0002). A construtora/Vanessa é *fonte* de leads e de empreendimentos, **não** um novo tenant — quem opera o CRM é a Jéssica, dentro do tenant atual. Multi-tenant real fica para quando o CRM for revendido.

---

## 2. Decisões

### 2.1 CRM operacional (criar/editar à mão) — fase rápida, desbloqueia já

- **Contatos:** CRUD completo. Uma página de **Contatos** no admin (lista + busca) e criação/edição por drawer (reaproveitando o `ContactDrawer`).
- **Oportunidades:** criação manual escolhendo um **contato existente** ou **criando um novo na hora**; define estágio, temperatura, e preenche as seções de campos personalizados (reaproveitando o `OpportunityDrawer` em modo "novo").
- **Origem:** oportunidade criada à mão nasce com `source = 'manual'` (rótulo "Manual", já existe em `SOURCE_LABELS`). Lead importado da base da Vanessa pode usar `source = 'import'`.
- **Backend:** endpoints `POST /contacts`, `PATCH /contacts/:id`, `GET /contacts` (lista/busca) e `POST /opportunities` (criação manual: contactId existente **ou** payload de contato novo + stageKey/fields). O `PATCH /opportunities/:id` e a engine de campos já existem.
- **Escopo de tenant:** tudo escopado pelo `TenantService` atual, como o resto.

> Racional: a Jéssica precisa trabalhar leads **hoje**. Isso não depende de nada da camada pública/SEO e entrega valor imediato.

### 2.2 Empreendimentos como registro tipado no código (por ora)

Começamos com **LPs no código** (decisão do time), mas **estruturadas como dado** para a migração futura a uma entidade gerenciável ser mecânica.

- Um **registro tipado** em `app/data/empreendimentos.ts`: um array de objetos com shape estável.
- Uma **rota dinâmica** `app/pages/empreendimentos/[slug].vue` que renderiza a LP a partir do registro.
- Um **hub** `app/pages/empreendimentos/index.vue` (listagem/vitrine), reusável nos cards da home.

Shape proposto (esboço — congela o contrato para virar tabela depois):

```ts
export interface Empreendimento {
  slug: string            // 'reserva-parque-vila-mariana'
  nome: string
  construtora: string
  status: 'lançamento' | 'em obras' | 'pronto para morar'
  cidade: string
  bairro: string
  uf: string
  precoAPartirDe: number   // em reais, para "a partir de R$ ..."
  tipologias: string[]     // ['2 dorms', '3 dorms']
  areaMin: number          // m²
  areaMax: number
  diferenciais: string[]   // lazer, localização, etc.
  descricao: string        // texto único (anti-conteúdo-fino)
  imagens: string[]
  coords?: { lat: number; lng: number }
  seo: { title: string; description: string }
  publicado: boolean
}
```

**Captação na LP:** cada LP integra o **simulador** (pré-preenchendo `propertyValue`/cidade a partir do empreendimento) e um **formulário de contato**. Ambos postam para o `POST /capture` genérico (ADR 0001/0002) com:
- `source = 'lp:<slug>'` (rastreia qual empreendimento gerou o lead);
- `fields.simulador.*` quando vier do simulador;
- `stageKey` decidido pelo site (RMSP é regra do site — ADR 0002): dentro da área → `Lead`, fora → `Nutrição`.

**Evolução (fora deste escopo, mas garantida pelo shape):** quando houver volume, `Empreendimento` vira tabela + CRUD no admin, sem tocar nas LPs (que passam a ler do banco em vez do array). O contrato de dados é o mesmo.

### 2.3 Home neutra (vitrine) mantendo guias e simulador

- A home deixa de ser um funil único de simulação e passa a apresentar **cards dos empreendimentos** (a partir do hub) + acessos a **simulador** e **guias**.
- **Nada é removido:** simulador e guias continuam com URLs e navegação próprias.
- A home mantém o `RealEstateAgent` JSON-LD (marca Meu Revelar) e ganha ligação clara para o hub de empreendimentos.

---

## 3. Estratégia anti-canibalização de SEO

O risco é real porque teremos vários tipos de página potencialmente disputando as mesmas queries. O princípio é **uma intenção de busca por tipo de página**, sem sobreposição:

| Tipo de página | Intenção | Queries-alvo | Exemplo |
|---|---|---|---|
| **LP de empreendimento** | Transacional / comercial | **nome do empreendimento**, "apartamento [bairro] [construtora]", "lançamento [região]" | "Reserva Parque Vila Mariana" |
| **Hub de empreendimentos** | Navegacional / categoria | "empreendimentos [construtora]", "lançamentos [cidade]" | "lançamentos zona sul SP" |
| **Simulador** | Ferramenta | "simulador financiamento", "calcular parcela" | canônico da calculadora |
| **Guias** | Informacional | "como funciona FGTS", "o que é ITBI" | conteúdo educativo |
| **Home** | Marca + topo de categoria | "Meu Revelar", termos genéricos de marca | — |

Regras de disciplina:

1. **Nomes específicos, não genéricos.** Cada LP mira o **nome do empreendimento** (query única por definição) — canibalização entre LPs é quase nula porque cada uma tem um alvo próprio. **Evitar** criar várias LPs disputando "apartamento no bairro X".
2. **Sem conteúdo fino/duplicado.** Dois empreendimentos no mesmo bairro se diferenciam por **produto e nome**, com `descricao` única. Nada de LPs quase-idênticas geradas por template vazio.
3. **Um dono por intenção.** A intenção "calcular financiamento" pertence ao **simulador**; as LPs **usam** o simulador embutido mas **não** competem por essa query (a LP não deve ser otimizada para "simulador de financiamento").
4. **Canonical + hub-and-spoke.** Cada página é self-canonical. Linkagem interna clara: home → hub → LPs; LPs → guias relevantes (financiamento, FGTS, ITBI); guias → simulador. Sem N páginas apontando para a mesma keyword.
5. **Schema por tipo.** LP de empreendimento com schema de imóvel/oferta (`Residence`/`Offer` ou `RealEstateListing`); guias com `Article`; home/negócio com `RealEstateAgent` (já existe); simulador como ferramenta.
6. **Sitemap segmentado + prioridades.** Agrupar LPs, guias e ferramentas; refletir a hierarquia.
7. **Medição.** Acompanhar no Search Console o sinal clássico de canibalização (duas URLs alternando na mesma query). Se aparecer, consolidar/diferenciar.

---

## 4. Sequência de execução

- **Fase A — CRM operacional (urgente).** Contatos CRUD + criação manual de oportunidade (§2.1). Entrega a Jéssica trabalhando a base da Vanessa. Independe da camada pública.
- **Fase B — Empreendimentos + LPs.** Registro tipado + rota dinâmica + hub (§2.2), com captação via `/capture`.
- **Fase C — Home vitrine.** Reorganizar a home com cards do hub, preservando guias/simulador (§2.3).
- **Fase D — SEO.** Aplicar §3 (schema por tipo, canonical, linkagem interna, sitemap, Search Console).

Fases B–D são incrementais e podem começar com **1–2 empreendimentos reais** antes de escalar.

---

## 5. Consequências

**A favor**
- Foco comercial claro (empreendimentos de uma construtora) em vez de captação difusa.
- CRM deixa de depender do simulador como única entrada — operação humana real.
- Empreendimentos como dado tipado → migração futura a CMS/DB sem reescrever LPs.
- Núcleo do CRM permanece genérico (ADR 0002 intacto).

**Custos / riscos**
- Manter LPs no código exige deploy para cada novo empreendimento (aceito enquanto o volume é baixo; mitigado pela evolução prevista em §2.2).
- SEO exige disciplina contínua (§3), não é "configurar e esquecer".
- A home neutra pode, no curto prazo, diluir sinais que hoje se concentram no funil do simulador — monitorar.

**Questões em aberto**
- Fonte dos dados dos empreendimentos (material da construtora: plantas, preços, imagens) — depende da Vanessa.
- Consentimento/LGPD para trabalhar a base importada da Vanessa (leads que não vieram do nosso formulário): validar base legal antes de operar.
- Política de `stageKey`/roteamento para leads importados vs. leads de LP.
