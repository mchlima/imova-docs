# Runbook de Deploy — Meu Revelar

> Estado atual (preparação, **nada em produção**): frontend e backend versionados e
> enviados ao GitHub; build de produção do frontend validado localmente. Falta escolher
> hospedagem e provisionar a infraestrutura. **Regra do projeto: não fazer deploy sem
> autorização explícita.**

## 1. Arquitetura a hospedar

| Componente | Stack | Repositório | Como sobe |
|---|---|---|---|
| Frontend | Nuxt 4 (SSR/Nitro) | `mchlima/imova-site` | `npm run build` → `node .output/server/index.mjs` |
| Backend | NestJS 11 + Prisma 6 | `mchlima/imova-backend` | `npm run build` → `npm run start:prod` (`node dist/main`) |
| Banco | PostgreSQL | — | serviço gerenciado |
| Uploads | Cloudflare R2 (bucket `imova-public`) | — | já é serviço externo |

Domínio registrado: **meurevelar.com.br**. Sugestão de topologia:
`meurevelar.com.br` (frontend) + `api.meurevelar.com.br` (backend) — manter no mesmo
domínio-raiz facilita o cookie de sessão do admin.

## 2. Decisões pendentes (definir antes de publicar)

1. **Hospedagem do frontend** — Nuxt SSR precisa de runtime Node (não é estático puro por
   causa do `/capture` e do SSR). Opções: Vercel, Netlify, ou Node em VPS/container.
2. **Hospedagem do backend** — Railway, Render, Fly.io ou VPS (precisa de processo Node
   persistente + porta).
3. **PostgreSQL gerenciado** — Neon, Supabase, Railway, RDS, etc.
4. **Estratégia de migração do banco** — ⚠️ hoje o schema é aplicado via `prisma db push`
   (não há pasta `prisma/migrations`). Para produção, escolher:
   - gerar migration inicial (`prisma migrate dev --name init`) e usar `prisma migrate deploy` no CI/deploy; **ou**
   - assumir `prisma db push` no deploy (mais simples, menos rastreável).

## 3. Variáveis de ambiente

### Backend (`imova-backend/.env`)
| Var | Produção |
|---|---|
| `DATABASE_URL` | string do Postgres gerenciado (com `sslmode=require` se aplicável) |
| `JWT_SECRET` | segredo forte novo (não reusar o de dev) |
| `JWT_EXPIRES_IN` | `7d` |
| `PORT` | porta exigida pelo host |
| `CORS_ORIGIN` | `https://meurevelar.com.br` |
| `COOKIE_SECURE` | `true` (HTTPS) |
| `R2_ACCOUNT_ID` / `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` | credenciais do R2 |
| `R2_BUCKET` | `imova-public` |
| `R2_PUBLIC_BASE` | URL pública do bucket (sem barra no fim) |

### Frontend (`imova-site/.env`)
| Var | Produção |
|---|---|
| `NUXT_PUBLIC_API_BASE` | `https://api.meurevelar.com.br` |
| `NUXT_PUBLIC_SITE_URL` | `https://meurevelar.com.br` |

## 4. Passo a passo

1. **Provisionar Postgres** e obter `DATABASE_URL`.
2. **Deploy do backend**:
   - setar env vars (seção 3);
   - `npm ci && npm run build`;
   - aplicar schema (`prisma migrate deploy` **ou** `prisma db push` — ver decisão 2.4);
   - seed: `npm run db:seed` (tenant + campos + admin) e `npm run db:seed:locations` (IBGE);
   - `npm run start:prod`.
3. **Deploy do frontend**:
   - setar `NUXT_PUBLIC_API_BASE` e `NUXT_PUBLIC_SITE_URL`;
   - `npm ci && npm run build`;
   - servir `node .output/server/index.mjs`.
4. **DNS**: `meurevelar.com.br` → frontend; `api.meurevelar.com.br` → backend. HTTPS em ambos.
5. **Ajustar CORS/cookie** no backend: `CORS_ORIGIN=https://meurevelar.com.br`,
   `COOKIE_SECURE=true`. Conferir `SameSite` do cookie `imova_token` para o login do admin
   funcionar entre os subdomínios.

## 5. Verificação pós-deploy

- [ ] Home (`/`) e LP (`/ares-do-horto`) abrem com imagens e SSR ok.
- [ ] `sitemap.xml` e `robots.txt` usam `https://meurevelar.com.br`.
- [ ] `POST /capture` cria oportunidade (testar o formulário de contato e o simulador).
- [ ] Login do admin (`/admin/login`) autentica e o cookie persiste (HTTPS + SameSite ok).
- [ ] Upload de capa no CMS grava no R2 e a imagem pública carrega.
- [ ] Guias do CMS aparecem na home e em `/guias`.

## 6. Riscos / atenção

- **Cookie de auth cross-site**: o admin usa cookie httpOnly. Front e back em subdomínios do
  mesmo raiz simplifica; domínios distintos exigem `SameSite=None; Secure`.
- **Migração de schema**: sem `prisma/migrations`, `migrate deploy` não roda — decidir a
  estratégia (seção 2.4) antes do primeiro deploy.
- **Segredos**: gerar `JWT_SECRET` novo para produção; nunca commitar `.env` (já ignorado).
- **Identificadores internos** permanecem como `imova` (slug do tenant, bucket `imova-public`,
  cookie `imova_token`) — ver ADR/rebrand; não confundir com a marca pública "Meu Revelar".
