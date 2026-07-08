# ADR 0008 — Armazenamento de documentos do lead (R2 privado)

- **Status:** aceito (2026-07-08)
- **Relação:** complementa o CRM (ADR 0002/0004) e o split Contato↔Oportunidade.
- **Decisão em uma frase:** documentos do lead (RG, comprovantes, etc.) ficam num **bucket R2 privado** (`imova-leads`), acessível **só via URL pré-assinada** gerada pelo backend; o **dono é o Contato** (reutilizável entre oportunidades), com **vínculo opcional à Oportunidade** que originou o envio.

---

## 1. Contexto

A corretora precisa solicitar/armazenar a documentação do lead (simulação de financiamento, análise de crédito). Requisitos: **privado e seguro**, acesso **só pelo CRM**. Dilema de modelagem: guardar na oportunidade dificulta reaproveitar em oportunidades futuras do mesmo contato; guardar no contato perde a relação com a oportunidade.

## 2. Decisão

**Modelo — `Document` pertence ao Contato, com vínculo opcional à Oportunidade:**
- `contactId` (obrigatório) = dono ⇒ reutilizável entre todas as oportunidades da pessoa.
- `opportunityId` (nullable) = oportunidade que originou o envio ⇒ rastreabilidade.
- `category` (curada: RG/CPF, comprovante de renda/residência, extrato, IR, certidão, contrato, **Outro** com `categoryLabel` livre), `fileName`, `storageKey`, `mimeType`, `size`, `uploadedBy`.
- `onDelete`: excluir a **oportunidade** faz `SetNull` (o documento continua no contato); excluir o **contato** faz `Cascade`.

**Armazenamento/segurança (só pelo CRM):**
- Bucket **privado** `imova-leads` no Cloudflare R2 (mesma conta do `imova-public`; `LeadsStorageService` separado do `R2Service`). Sem URL pública.
- **Upload** proxied pelo backend (`FileInterceptor`, multipart) com validação de tipo (PDF/imagens/Office) e tamanho (**25 MB**); key `tenant/{tenantId}/contacts/{contactId}/{uuid}-{arquivo}`.
- **Ver/baixar** via **URL pré-assinada** (GET, TTL 5 min) gerada sob demanda; `?download=1` força download, senão abre inline. A `storageKey` nunca é exposta ao front.
- Endpoints sob `JwtAuthGuard`: `POST /documents`, `GET /documents/contact/:contactId`, `GET /documents/:id/url`, `DELETE /documents/:id`.
- Config: `R2_LEADS_BUCKET` (credenciais reutilizam o token da conta; suporta `R2_LEADS_ACCESS_KEY_ID/SECRET` dedicados se quiser escopar).

**Front:**
- Componente reutilizável `DocumentsPanel` (+ `DocumentRow`). Aba **"Documentos"** no drawer da oportunidade (mostra "desta oportunidade" × "outros do contato", reutilizáveis) e seção **Documentos** no drawer do contato (lista única).

## 3. Consequências

- Reaproveitamento (docs na pessoa) + rastreabilidade (origem na oportunidade).
- Migração **aditiva** (cria a tabela `documents`).
- Objetos do R2 não caem por cascata do banco ao excluir contato — a limpeza do R2 é feita no delete explícito de documento; limpeza em massa (ao excluir contato) fica como melhoria futura.
