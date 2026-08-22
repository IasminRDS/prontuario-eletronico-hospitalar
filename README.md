# Prontuário Eletrônico Hospitalar (PE-E2 / SNPE)

[![backend-ci](https://github.com/IasminRDS/prontuario-eletronico-hospitalar/actions/workflows/ci.yml/badge.svg)](https://github.com/IasminRDS/prontuario-eletronico-hospitalar/actions/workflows/ci.yml)
![NestJS](https://img.shields.io/badge/NestJS-10-E0234E?logo=nestjs&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js-16-000000?logo=nextdotjs&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)
![FHIR R4](https://img.shields.io/badge/FHIR-R4-E36002)
![License](https://img.shields.io/badge/licen%C3%A7a-MIT-blue)

Plataforma de **prontuário eletrônico hospitalar** estilo SUS. Monorepo com **API NestJS** e **frontend Next.js**, cobrindo o fluxo clínico real (chegada → triagem/fila → atendimento → exames → prescrição → internação → evolução → alta), com **RBAC granular por permissão**, **multi-tenancy** (multi-hospital), **auditoria LGPD transacional**, Outbox/eventos e interoperabilidade **FHIR R4**.

> Integração **100% real** entre frontend e backend (`/api/v1`) — sem mocks. O fluxo clínico completo já foi validado ponta a ponta (login → alta) via HTTP e via navegador.

---

## Sumário

- [Stack](#stack)
- [Arquitetura](#arquitetura)
- [Estrutura de pastas](#estrutura-de-pastas)
- [Como rodar localmente](#como-rodar-localmente)
- [Autenticação e RBAC](#autenticação-e-rbac)
- [Fluxos clínicos](#fluxos-clínicos)
- [Camada clínica do frontend (React Query)](#camada-clínica-do-frontend-react-query)
- [Documentação](#documentação)
- [Deploy](#deploy)

---

## Stack

**Backend** — NestJS 10 · Prisma · PostgreSQL 16 · JWT (access + refresh) · Argon2 · class-validator · RBAC por permissão · Outbox + eventos idempotentes · FHIR R4 · Jest/Supertest · Swagger.

**Frontend** — Next.js 16 (App Router) · TypeScript · **TanStack Query (React Query) 5** · Zustand (auth) · Axios (interceptor Bearer + refresh automático) · React Hook Form + Zod · Tailwind CSS v4 · design system próprio.

---

## Arquitetura

- **Multi-tenancy real:** isolamento por `hospitalId` **automático** via middleware Prisma + `AsyncLocalStorage`. Nenhum serviço filtra manualmente; queries clínicas sem tenant são bloqueadas. A identidade do cidadão permanece **nacional** (MPI/Cidadão).
- **FSM clínica no domínio puro:** transições de estado do paciente validadas no backend, independentes de framework.
- **Auditoria imutável (LGPD):** cada mutação clínica grava um evento de auditoria na **mesma transação** (atomicidade) — quem fez, o quê, quando, em qual tenant.
- **RBAC granular por permissão** (`recurso:ação`): o backend é a autoridade; o frontend espelha as permissões para condicionar botões/rotas via `<Can>`.
- **Interoperabilidade FHIR R4:** `Patient`, `Encounter`, `MedicationRequest`, `Observation`.

---

## Estrutura de pastas

```
.
├── backend/                      → API NestJS + Prisma + PostgreSQL
│   ├── prisma/
│   │   ├── schema.prisma         → modelo físico (multi-tenant, clínico)
│   │   ├── migrations/           → migrations versionadas
│   │   └── seed.ts               → seed idempotente (estrutura + catálogos + usuários + pacientes)
│   └── src/
│       ├── modules/              → um módulo por domínio (controller/service/dto)
│       │   ├── pronto-socorro/   ├── internacao/       ├── exames/
│       │   ├── prescricao-hospitalar/  ├── cirurgia/   ├── pacientes/  …
│       ├── shared/               → RBAC, guards, interceptors, tenant, auditoria
│       └── infra/                → prisma, auth, fhir, config
│
├── frontend/                     → App web Next.js (App Router)
│   └── src/
│       ├── app/
│       │   ├── providers.tsx     → QueryClientProvider (React Query)
│       │   ├── login/            → autenticação
│       │   └── (app)/            → shell protegido (sidebar + topbar)
│       │       ├── pronto-socorro/   → fila + chegada + chamar + painel de atendimento
│       │       └── internacao/       → mapa de leitos + internar + evolução + alta
│       ├── modules/
│       │   ├── clinical/         → CAMADA CLÍNICA (React Query)
│       │   │   ├── types.ts      → contratos alinhados aos DTOs do backend
│       │   │   ├── keys.ts       → query keys centralizadas
│       │   │   ├── emergencia.ts → service + hooks (fila/chegada/chamar/finalizar)
│       │   │   ├── exames.ts     → service + hooks (tipos/solicitar/resultado)
│       │   │   ├── prescricao.ts → service + hooks (criar/administrar/suspender)
│       │   │   └── internacao.ts → service + hooks (leitos/internar/evoluir/alta)
│       │   └── shared/rbac/      → permissions, <Can>, nav
│       ├── services/             → axios client (Bearer + refresh) — reutilizado
│       └── store/                → auth store (Zustand)
│
└── docs/                         → documentação de arquitetura
```

---

## Como rodar localmente

**Requisitos:** Node.js 20+, PostgreSQL 16 (ou Docker), npm.

### 1. Backend (porta 3000)

```bash
cd backend
cp .env.example .env                 # ajuste segredos e DATABASE_URL

# Suba um PostgreSQL (Docker):
docker run -d --name pe-pg \
  -e POSTGRES_USER=prontuario -e POSTGRES_PASSWORD=prontuario \
  -e POSTGRES_DB=prontuario_e2 -p 5432:5432 postgres:16-alpine

npm install
npm run prisma:generate
npm run prisma:deploy                # aplica as migrations
npm run prisma:seed                  # estrutura + catálogos + usuários + pacientes
npm run start:dev                    # http://localhost:3000/api/v1  (Swagger em /api/docs)
```

**Usuários semeados** (idempotente):

| Login       | Senha           | Perfil        | Acesso |
|-------------|-----------------|---------------|--------|
| `admin`     | `Admin@123`     | SuperAdmin    | cross-tenant |
| `dr.souza`  | `Medico@123`    | Médico        | fluxo clínico completo |
| `enf.lima`  | `Enfermeiro@123`| Enfermeiro    | triagem, PS, exames, administração |
| `gestor`    | `Gestor@123`    | Administrador | gestão do hospital |

O seed cria ainda: 4 setores (Emergência, UTI, Clínica Médica, Centro Cirúrgico), 12 leitos livres, catálogo de exames (Hemograma, Glicemia, Raio-X, Tomografia, Urina I), medicamentos (Dipirona, Paracetamol, Ceftriaxona, Soro Fisiológico) e 15 pacientes fictícios (CPF com dígito verificador válido).

### 2. Frontend (porta 3001)

```bash
cd frontend
cp .env.local.example .env.local     # NEXT_PUBLIC_API_URL=http://localhost:3000/api/v1
npm install
npm run dev                          # http://localhost:3001
```

Acesse `http://localhost:3001/login` e entre com `dr.souza / Medico@123`.

### Monorepo (na raiz)

```bash
npm install          # instala backend + frontend (workspaces)
npm run dev          # API (3000) + web (3001) juntos
npm run build
```

---

## Autenticação e RBAC

- **Login** → o backend emite `accessToken` + `refreshToken` (JWT). O `hospitalId` (tenant) e o `perfil` vão no token.
- **Axios interceptor** injeta `Authorization: Bearer <token>` e faz **refresh automático e transparente** em `401` (single-flight); se irrecuperável, limpa a sessão e volta ao login.
- **Proteção de rota** client-side re-hidrata a sessão do token; o backend continua sendo a autoridade real.
- **RBAC granular** (`recurso:ação`): o componente `<Can perm="exam:write">…</Can>` (ou `any={[…]}`) condiciona botões/seções. Perfis → permissões espelham o backend.

---

## Fluxos clínicos

Fluxo principal (validado ponta a ponta, sem Postman):

```
login → chegada (PS) → fila → chamar paciente → atendimento
      → solicitar exame → registrar resultado → prescrição
      → internação (leito) → evolução (SOAP) → alta (libera leito)
```

Regras de negócio aplicadas no backend e refletidas na UI:
- Internar exige leito `livre` do mesmo tenant → `409` se ocupado; a alta libera o leito para higienização.
- Fila do PS só permite "Chamar" quem está `aguardando`; transições inválidas retornam `409`.
- Erros da API (`400`, `403`, `409`) são tratados e exibidos na interface.

---

## Camada clínica do frontend (React Query)

Todos os **dados novos** usam React Query — `useQuery` para leitura e `useMutation` com **`invalidateQueries` automático** após cada mutação (a fila, os leitos e as internações se atualizam sozinhos). Exemplo real:

```ts
// modules/clinical/emergencia.ts
export function useRegistrarChegada() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (input: CreateChegadaInput) => emergenciaService.registrarChegada(input),
    onSuccess: () => qc.invalidateQueries({ queryKey: clinicalKeys.ps.fila }),
  });
}
```

O `QueryClient` (em `app/providers.tsx`) **não re-tenta em erros 4xx** (respostas de negócio legítimas) e faz `refetch` ao focar a janela — adequado a um sistema clínico.

---

## Documentação

A pasta [`docs/`](docs/) traz o detalhamento que não cabe aqui:

| Documento | Conteúdo |
|---|---|
| [VISAO-GERAL.md](docs/VISAO-GERAL.md) | O problema, o recorte do sistema e as decisões de produto |
| [ARQUITETURA-SNPE.md](docs/ARQUITETURA-SNPE.md) | Multi-tenancy, camadas, Outbox e modelo de eventos |
| [API.md](docs/API.md) | Contrato dos endpoints do `/api/v1` |
| [BASE-LEGAL-LGPD.md](docs/BASE-LEGAL-LGPD.md) | Base legal do tratamento e desenho da auditoria |
| [SECURITY.md](docs/SECURITY.md) | Modelo de ameaças, autenticação e RBAC |
| [DEPLOY.md](docs/DEPLOY.md) · [OPERACOES.md](docs/OPERACOES.md) | Publicação, migrations e rotina de operação |
| [HOMOLOGACAO.md](docs/HOMOLOGACAO.md) · [FASE5-VALIDACAO-E2E.md](docs/FASE5-VALIDACAO-E2E.md) | Roteiro de validação ponta a ponta |

---

## Deploy

- **Backend:** imagem Docker (`Dockerfile` na raiz do monorepo — build a partir da raiz usa o `package-lock.json` da raiz). Em produção, referencie a imagem pelo `:${SHA}` (imutável) — o CI (`.github/workflows/ci.yml`) compila, testa (unit + e2e com Postgres real) e builda no mesmo commit. Configure `DATABASE_URL`, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `CORS_ORIGINS`. Aplique migrations com `npm run prisma:deploy` antes de subir.
- **Frontend:** `npm run build && npm run start` (Node) ou deploy na Vercel. Defina `NEXT_PUBLIC_API_URL` apontando para a API pública (`https://…/api/v1`).
- **Banco:** PostgreSQL 16 gerenciado. Rode o seed apenas em ambientes de demonstração.

---

## Segredos

Nenhum segredo é versionado. Cada app tem seu `.env` (ignorado); os templates são `backend/.env.example` e `frontend/.env.local.example`.
