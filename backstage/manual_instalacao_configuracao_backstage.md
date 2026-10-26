# Manual do Backstage na org `acme` — instalação, configuração, templates e troubleshooting

**Contexto de uso:** portal interno (IDP) da organização GitHub `acme`, para provisionamento de infraestrutura na **AWS** (e futuramente **Azure**) via **Terraform**, rodando em **Docker** em máquina local (sem Kubernetes por enquanto), com integração ao **GitHub via Organization**.

Este manual documenta o setup **real** já implementado nos repositórios da org — não um exemplo genérico. Os nomes de repositório usados aqui (`infra-portal`, `infra-cloud`, `terraform-modules`) são os nomes reais em `github.com/acme/`.

**Histórico deste documento:** em 2026-09-14 o manual de instalação foi consolidado com toda a documentação de conceitos, decisões de configuração e troubleshooting que vivia espalhada em `docs/` (`conceitos/`, `configurações-extras/`, `troubleshooting/`) e `infra-cloud/docs/` no workspace `backstage-org`. O conteúdo foi conferido contra o estado atual dos repositórios — o que já foi corrigido aparece como histórico, não como pendência. O pipeline GitHub Actions embutido nos skeletons tem documento próprio: [`pipeline_terraform_scaffolder.md`](pipeline_terraform_scaffolder.md). Os ADRs dos módulos Terraform (`ADR-001` S3, `ADR-002` VPC) continuam em `docs/adrs/` — são design dos módulos, não do Backstage, e só são referenciados onde um skeleton hardcoda o baseline deles.

---

## Sumário

**Parte I — Fundamentos e instalação**

1. [Organização dos repositórios e como se conectam](#1-organização-dos-repositórios-e-como-se-conectam)
   - 1.1 Estrutura interna do `infra-portal` · 1.2 Regra de ouro para segredos · 1.3 Onde criar os módulos Terraform
2. [Conceitos: Catálogo vs. Templates](#2-conceitos-catálogo-vs-templates)
3. [Instalação passo a passo (do zero até rodando em Docker)](#3-instalação-passo-a-passo-do-zero-até-rodando-em-docker)
   - 3.4 Ajustes de código que o `create-app` não faz sozinho · 3.7 Build e subida via Docker Compose (bundle pré-compilado) · 3.9 Comandos do dia a dia
4. [Integração com GitHub Organization](#4-integração-com-github-organization)
   - 4.3 Criação do GitHub App · 4.4 Permissões concedidas e por quê · 4.5 Procedimento ao adicionar permissão · 4.6 Diagnóstico via API · 4.7 Configuração no `app-config.yaml` · 4.8 Merge de config · 4.9 `catalog.rules` · 4.10 Providers · 4.11 Troubleshooting de login e sincronização

**Parte II — Portal de infraestrutura com Terraform**

5. [Separação entre "gerar código" e "aplicar infra"](#5-separação-entre-gerar-código-e-aplicar-infra)
   - 5.2 Criar (Scaffolder) vs. descobrir (Catalog Discovery) · 5.3 Backend de estado · 5.4 Organização por provider
6. [Modelo de repositórios: um repositório de infra por time](#6-modelo-de-repositórios-um-repositório-de-infra-por-time)
7. [Templates do `infra-cloud`](#7-templates-do-infra-cloud)
   - 7.1 Anatomia comum · 7.2 `infra-repo-bootstrap` · 7.3 `aws-s3-bucket` · 7.4 `aws-vpc` · 7.5 Organização do skeleton · 7.6 O workflow gerado
8. [Passo a passo: expondo um módulo Terraform já existente como template](#8-passo-a-passo-expondo-um-módulo-terraform-já-existente-como-template)
9. [Pipeline Terraform nos repositórios gerados](#9-pipeline-terraform-nos-repositórios-gerados)
   - 9.1 OIDC com a AWS · 9.2 `terraform init` com módulo privado · 9.3 Variáveis por repositório

**Parte III — Operação**

10. [Plugins e integrações](#10-plugins-e-integrações)
11. [Documentação de recursos (TechDocs)](#11-documentação-de-recursos-techdocs)
12. [Usuários e Grupos: donos e consumidores de recursos](#12-usuários-e-grupos-donos-e-consumidores-de-recursos)
13. [O que reiniciar a cada alteração / hot-reload](#13-o-que-reiniciar-a-cada-alteração--hot-reload)
14. [Troubleshooting: catálogo de erros](#14-troubleshooting-catálogo-de-erros)
15. [Checklists](#15-checklists)
16. [Pendências e próximos passos](#16-pendências-e-próximos-passos)
17. [Referências](#17-referências)

---

# Parte I — Fundamentos e instalação

## 1. Organização dos repositórios e como se conectam

Backstage tende a virar bagunça rápido se tudo (app, catálogo, templates, docs) morar no mesmo lugar sem critério. A organização usada na org `acme` usa **repositórios separados**, cada um com um ciclo de vida e um público diferente:

```
github.com/acme/
├── infra-portal/                # código do app Backstage (este é o "software")
├── infra-cloud/                 # templates do Scaffolder (skeletons Terraform)
├── terraform-modules/           # módulos Terraform reutilizáveis (S3, VPC, ...)
├── docs/                        # ADRs dos módulos Terraform (o resto foi consolidado neste manual)
├── catalog-org-data/            # reservado para entidades do catálogo — ainda vazio, sem conteúdo
└── developer-portal/            # nome antigo/alternativo cogitado para o app — ainda vazio, não usar
```

Localmente esses repositórios ficam agrupados num workspace `backstage-org/`, que **não é um repositório git** — cada subdiretório tem seu próprio `.git`. Ao commitar/pushar, sempre rodar `git` de dentro do subdiretório do repo correto.

**Por que separar assim:**

- **`infra-portal`** muda pouco e exige rebuild de imagem Docker (código TypeScript). Só o time de plataforma mexe aqui.
- **`infra-cloud`** muda com frequência moderada (novo template = novo tipo de recurso) e **não precisa de rebuild** — é lido em runtime pelo Scaffolder, via `catalog.locations` do tipo `url` (seção 4.7).
- **`terraform-modules`** é consumido pelos templates via `source = "git::https://github.com/acme/terraform-modules.git//<módulo>?ref=<tag>"`, versionado com tags Git (ex. `aws-s3-bucket/v0.1.0`, `aws-vpc/v0.1.0`), testado independente do Backstage.
- **`catalog-org-data`** e **`developer-portal`** existem como pastas reservadas na org, mas **estão vazios hoje**. Os dados de `User`/`Group` vêm de duas fontes que já funcionam sem esse repositório (seções 4.7 e 12.1): o arquivo local `examples/org.yaml` (dados de exemplo) e o provider `githubOrg`, que sincroniza Teams/Members da organização diretamente.

Essa separação evita que uma alteração de baixo risco (adicionar um usuário no catálogo) precise passar pelo mesmo pipeline de build/deploy do app em si.

| Repositório | Contém | Quem edita | Frequência de mudança |
|---|---|---|---|
| `infra-portal` | Código do app Backstage | Time de plataforma | Baixa |
| `infra-cloud` | `template.yaml` do Scaffolder + `skeleton/` (estrutura que vira o conteúdo do repositório do recurso, incluindo `.github/workflows/`) | Time de plataforma | Moderada |
| `terraform-modules` | Módulos HCL reutilizáveis (`aws/s3-bucket`, `aws/vpc`, futuramente `azure/`), versionados por tag | Time de plataforma/infra | Moderada, sempre versionada |
| `<repo-do-time>` (recebe o PR do Scaffolder) | `.tf` já preenchido (chamando o módulo via `ref` fixo) + `.github/workflows/terraform-ci-cd.yml` + `catalog-info.yaml` | Time dono do recurso | Uma vez por recurso criado, depois só em mudanças pontuais |

**Como o `infra-portal` descobre entidades e templates** — dois mecanismos distintos, não confundir:

1. **`catalog.locations` (estático, em `app-config.yaml` e `app-config.production.yaml`)** — aponta via URL do GitHub direto pros `template.yaml` do `infra-cloud`. É assim que os templates aparecem em "Create". Mudar o path de um template no `infra-cloud` exige atualizar essas URLs **nos dois** arquivos (seção 4.8).
2. **`catalog.providers.github.acmeProvider` (discovery dinâmico)** — varre toda a org `acme` a cada 1 min procurando `**/catalog-info.yaml` na branch `main` e registra `Component`/`System`/`Resource` sozinho, sem step `catalog:register` nos templates (seções 4.10 e 5.2).

Fluxo de ponta a ponta:

```mermaid
flowchart LR
    subgraph portal["infra-portal (Backstage)"]
        CR[Create: lista de Templates]
        CAT[Catalog: System / Resource]
    end
    subgraph cloud["infra-cloud"]
        T1[infra-repo-bootstrap]
        T2[aws-s3-bucket / aws-vpc]
    end
    subgraph gh["GitHub org acme"]
        REPO["repo de infra do time<br/>infra/&lt;recurso&gt;/ + workflow"]
        TFM[(terraform-modules<br/>privado)]
    end
    CR -->|locations type: url| T1
    CR -->|locations type: url| T2
    T1 -->|publish:github<br/>commit direto na main| REPO
    T2 -->|publish:github:pull-request| REPO
    REPO -->|acmeProvider acha<br/>catalog-info.yaml na main| CAT
    REPO -->|GitHub Actions: plan em PR,<br/>apply via workflow_dispatch| AWS[(AWS via OIDC)]
    REPO -.->|terraform init clona| TFM
```

### 1.1 Estrutura interna do repositório do app (`infra-portal`)

```
infra-portal/
├── app-config.yaml                 # config base, versionada, SEM segredos
├── app-config.production.yaml      # overrides de produção, versionada, referencia ${ENV_VARS}
├── catalog-info.yaml               # entidade do próprio Backstage no catálogo (dogfooding)
├── docker-compose.yml
├── .env                            # segredos locais, NUNCA commitado
├── packages/
│   ├── app/
│   │   └── src/
│   │       └── modules/
│   │           ├── nav/            # sidebar customizada
│   │           ├── home/           # widgets da home page
│   │           └── signIn/         # tela de login customizada — obrigatória, ver seção 3.4
│   └── backend/
│       ├── src/index.ts            # lista de plugins/módulos registrados no backend
│       └── Dockerfile              # gerado pelo create-app, referenciado no docker-compose.yml
├── plugins/                        # plugins customizados internos — vazio hoje, só o README padrão
└── examples/
    ├── entities.yaml                # System/Component/API de exemplo
    └── org.yaml                     # User/Group `guest` de exemplo, ver seção 12.4
```

**Atenção a arquivos soltos que não deveriam existir:** o `packages/backend/Dockerfile` copia config para dentro da imagem com `COPY ... app-config*.yaml ./` — esse `*` é um glob, não um `--config` explícito. Qualquer arquivo que bata no padrão `app-config*.yaml` (ex. uma cópia esquecida tipo `app-config copy.yaml`) entra na imagem sem necessidade. Não quebra nada, mas é lixo na imagem — vale conferir `ls app-config*.yaml` na raiz do `infra-portal` de vez em quando.

Não existe hoje um `app-config.local.yaml` no repositório (dev usa só `app-config.yaml`, com SQLite em memória, seção 3.3).

### 1.2 Regra de ouro para segredos

- Nunca commitar tokens/segredos em `app-config*.yaml`. Sempre referenciar via `${VAR}`, como já é feito em `infra-portal/app-config.yaml`:
  ```yaml
  integrations:
    github:
      - host: github.com
        apps:
          - appId: ${GITHUB_APP_APP_ID}
            clientId: ${GITHUB_APP_CLIENT_ID}
            clientSecret: ${GITHUB_APP_CLIENT_SECRET}
            privateKey: ${GITHUB_APP_PRIVATE_KEY}
  ```
  `webhookSecret` é opcional no schema do Backstage e **não está configurado hoje** (o webhook do GitHub App está desativado, seção 4.3, item 4). Se for ativá-lo no futuro, adicione `webhookSecret: ${GITHUB_APP_WEBHOOK_SECRET}` só depois de a variável existir no `.env` — referenciar uma variável ausente derruba o backend no boot com erro de config (seção 3.7.3).
- Em máquina local, essas variáveis vêm do `.env` (carregado pelo `docker-compose.yml` via `env_file`), nunca commitado (está no `.gitignore`).
- Quando isso for para um servidor compartilhado, troque o `.env` por um secret manager (AWS Secrets Manager / SOPS + git-crypt), mas a estrutura do `app-config.yaml` não muda — só a origem da variável.
- A chave privada `.pem` do GitHub App não deve ficar em repositório de documentação. Se estiver guardada junto deste manual, é só por conveniência local e precisa estar no `.gitignore` do repositório.

### 1.3 Onde criar os módulos Terraform (`terraform-modules`)

Pergunta comum: os módulos Terraform reutilizáveis (o HCL de verdade que provisiona S3, VPC etc.) devem morar dentro do repositório de configuração do Backstage, dentro do repositório de templates, ou em um repositório totalmente separado?

**Resposta: repositório separado (`terraform-modules`), nunca dentro de `infra-portal`.**

**Por que não dentro de `infra-portal`:** é código do portal (TypeScript), buildado em imagem Docker. Misturar `.tf` ali não tem efeito nenhum em runtime — o backend do Backstage não interpreta Terraform, só dispara ações de `git`/CI.

**Por que não dentro de `infra-cloud` (junto com os skeletons):** cada `skeleton/` é **copiado** inteiro para dentro do repositório de destino quando um recurso é criado — se o módulo estivesse dentro do skeleton, cada bucket S3 criado ganharia sua **própria cópia congelada** do módulo, sem forma de saber depois quais recursos usam qual versão, e sem forma de aplicar um fix (ex.: de segurança) em todos de uma vez.

**Por que um repositório `terraform-modules` separado resolve isso:**

- **Fonte única de verdade** — o módulo `aws/s3-bucket` existe em um lugar só; todo skeleton que precisa dele referencia essa fonte via Git.
- **Versionado com tags Git**, independente do ciclo do Backstage e dos templates. Convenção `<módulo>/vX.Y.Z` (ex. `aws-s3-bucket/v0.1.0`, `aws-vpc/v0.1.0`) — e não `v0.1.0` plana, porque `terraform-modules` é monorepo e uma tag Git é global ao repositório: uma tag plana impediria versionar os módulos de forma independente. A skill `git-terraform` do `terraform-modules` automatiza add → commit → push → tag; o bump semver é sempre decisão de quem chama.
  ```hcl
  module "bucket" {
    source = "git::https://github.com/acme/terraform-modules.git//aws/s3-bucket?ref=aws-s3-bucket/v0.1.0"
    # ...
  }
  ```
  O `main.tf` gerado pelo skeleton referencia uma **tag fixa**, nunca `main` — isso garante que um recurso já criado não muda de comportamento sozinho quando o módulo evolui. Atualizar a versão usada por um recurso é uma mudança explícita (PR trocando o `ref`).
- **Testável isoladamente** — dá pra rodar `terraform validate`/`plan` no próprio `terraform-modules` (skill `terraform-deploy` e subagentes `aws-architect`/`devops-engineer` desse repositório), sem precisar do Backstage rodando.
- **Reutilizável entre múltiplos templates.**

**Convenções do `terraform-modules` que afetam os skeletons:** nenhuma `variable` tem `default` (valores ficam fora do módulo), variáveis do mesmo domínio ficam agrupadas em `object(...)`, e várias são `nullable = false`. Consequência: **toda** variável obrigatória do módulo precisa ser preenchida no `main.tf` do skeleton, ou o `plan` falha (seções 7.4 e 8, Passo 2).

**Atenção:** `acme/terraform-modules` é **privado** — isso tem impacto direto no `terraform init` do pipeline dos repos gerados (seção 9.2).

---

## 2. Conceitos: Catálogo vs. Templates

### O problema de vocabulário

O fluxo que motivou boa parte desta documentação foi descrito assim:

> Cliente precisa criar um bucket S3, então ele acessa o Backstage, clica no catálogo, escolhe o S3, preenche o formulário, clica em criar e o fluxo segue.

Essa frase mistura dois conceitos que no Backstage são **coisas diferentes**, mesmo aparecendo lado a lado na mesma barra de navegação:

| O que o usuário faz | Nome real no Backstage | O que é |
|---|---|---|
| "Clica no catálogo, escolhe o S3" | **Create** (Scaffolder) | Lista de *Templates* disponíveis — assistentes que geram algo novo |
| "Preenche o formulário, clica em criar" | Execução de um *Template* | Roda os `steps` do `template.yaml` |
| Resultado aparecer listado depois | **Catalog** (Software Catalog) | Inventário de entidades que existem (ou foram declaradas) na organização |

**O "S3" que o cliente escolhe é um Template. O que sobra registrado depois, pra outras pessoas acharem, é uma entidade no Catalog.**

### Software Catalog: o inventário

Registro central de **entidades**, cada uma um YAML (`catalog-info.yaml`):

```yaml
apiVersion: backstage.io/v1alpha1
kind: <tipo-da-entidade>
metadata:
  name: <identificador único>
spec:
  type: <subtipo>
  owner: <quem é responsável>
```

| `kind` | Representa | Exemplo no nosso caso |
|---|---|---|
| `Component` | Algo com código-fonte que roda (serviço, job, lib) | `component:default/infra-portal` (o próprio portal, descoberto pelo provider) |
| `Resource` | Infraestrutura de apoio, sem código próprio | o bucket S3 / a VPC (gerado por `aws-s3-bucket`/`aws-vpc`) |
| `System` | Agrupamento lógico de Components/Resources | o repositório de infra de um time (gerado por `infra-repo-bootstrap`) |
| `Group` / `User` | Times e pessoas | `group:default/plataforma` (sincronizado pelo provider `githubOrg`) |
| `Template` | **Um assistente de criação** — o próprio template também é uma entidade do catálogo | `template:default/aws-s3-bucket` |
| `Location` | Aponta pra onde um `catalog-info.yaml` mora | as entradas de `catalog.locations` |

O ponto que costuma confundir: **`Template` também é `kind` de entidade do catálogo.** A tela "Create" nada mais é do que o Catalog filtrado por `kind: Template`. É a mesma base de dados, vista por uma lente diferente.

### Scaffolder: a fábrica que preenche o Catalog

Um Template é uma **receita**: formulário (`parameters`) + sequência de ações (`steps`) que, ao rodar, normalmente **produzem** outras entidades. Para o `aws-s3-bucket`:

```mermaid
flowchart TB
    subgraph catalog["Software Catalog"]
        T["kind: Template<br/>template:default/aws-s3-bucket<br/>(o que aparece em 'Create')"]
        R["kind: Resource<br/>resource:default/&lt;resourceName&gt;<br/>(o bucket, registrado após o merge)"]
    end

    U([Cliente]) -->|1. abre Create, escolhe o card| T
    T -->|2. preenche parameters| F[Formulário]
    F -->|3. clica em Create| S1["fetch:template<br/>renderiza skeleton/ (main.tf, backend.tf, catalog-info.yaml)"]
    S1 --> S2["publish:github:pull-request<br/>abre PR no repo do time"]
    S2 -.->|revisão humana + CI terraform plan| PR[Pull Request]
    PR -.->|merge na main| D["acmeProvider (discovery)<br/>encontra catalog-info.yaml"]
    D -->|cria| R
```

### Por que isso importa pro objetivo "alguém poder criar a partir dali"

Se o objetivo é que, depois de o bucket existir, **outro time consiga referenciá-lo** (um `Component` com `dependsOn: [resource:default/<bucket>]`, ou um template futuro com `ui:field: EntityPicker` "escolha um bucket existente"), é a entidade **`Resource`** no Catalog que viabiliza isso — não o Template. Como o registro só acontece depois do merge (via discovery), a `Resource` aparece no catálogo **depois** de o PR ter sido aprovado — o comportamento desejado. Ver seção 12.2 para o modelo "dono vs. consumidor".

> **Create** = escolher e preencher um `Template` (a receita).
> **Catalog** = onde aparece o `Resource`/`System` que a receita produziu, para o resto da organização descobrir e reutilizar.

### Histórico das pendências levantadas por este conceito (2026-08-24 → 2026-09-02)

- **`register` rodar só depois do PR revisado/mergeado — resolvido (2026-09-02).** Não é possível resolver reordenando `steps`: o Scaffolder executa `fetch → publish → register` numa única chamada síncrona no clique de "Create". A solução foi **remover o step `register` e confiar no discovery** (seção 5.2). A alternativa "registro via CI" (`POST /api/catalog/locations` no fim do job `apply`) não foi necessária — fica registrada só caso a org precise registrar entidades fora do alcance do provider (ex.: repos fora da org `acme`).
- **Template do S3 não aparecia em "Create" — resolvido.** Duas causas: `catalog.rules` global bloqueando `Template` (seção 4.9) e `catalog.locations` não replicado em produção (seção 4.8). Detalhes na seção 14.
- **`githubOrg` ≠ discovery de `catalog-info.yaml` — superado.** Em 2026-08-24 o único provider dinâmico era o `githubOrg` (só `Group`/`User`). O `acmeProvider` foi adicionado depois (seção 4.10).

---

## 3. Instalação passo a passo (do zero até rodando em Docker)

**Ponto que gera confusão:** o `docker-compose.yml` sozinho **não instala nada** — ele só sobe containers a partir de uma imagem. Antes de existir uma imagem para subir, é preciso **criar o código-fonte do app Backstage** (feito uma única vez, fora do Docker, com Node/Yarn), **fazer os ajustes de código que o `create-app` não faz sozinho** (seção 3.4 — o passo mais frequentemente esquecido) e **configurar as integrações** (GitHub App). O Compose é só a última etapa. A ordem real é:

```
1. Criar o código-fonte do app (npx @backstage/create-app)     → fora do Docker
2. Testar localmente sem Docker (yarn start)                    → fora do Docker, opcional mas recomendado
3. Ajustes de código que o create-app não inclui por padrão      → fora do Docker, OBRIGATÓRIO (seção 3.4)
4. Criar o GitHub App na organização                             → manual, no GitHub (seção 4)
5. Preencher .env e app-config.yaml com as credenciais            → fora do Docker
6. yarn build:backend                                             → gera os artefatos que o Dockerfile espera
7. Build da imagem (docker compose build) e subida (up -d)        → aqui entra o Docker
8. Validar (logs + login em localhost:7007)
```

Os passos 1–3 só existem **uma vez**, quando o portal é criado. No dia a dia, alterações de código passam a ser: editar → `yarn build:backend` → `docker compose build` → `docker compose up -d` (seção 13 detalha o que exige isso e o que não exige).

Sem os três ajustes de código do passo 3, o app **nunca** loga com GitHub — não são uma correção de bug, são parte do setup obrigatório, e já estão feitos no `infra-portal` de hoje.

### 3.1 Pré-requisitos na máquina local

- **Node.js** (a versão usada em `infra-portal/package.json` é `"22 || 24"`, mesma major do `FROM node:24-trixie-slim` do Dockerfile) e **Yarn 4** (`packageManager: yarn@4.13.0`) — necessários só para os passos 1–3 e 6, não para rodar o container depois.
- **Docker** e **Docker Compose** instalados.

### 3.2 Passo 1 — Criar o código-fonte do app

```bash
npx @backstage/create-app@latest
```

O CLI pergunta o nome do app (`infra-portal`) e cria a pasta com `packages/app`, `packages/backend`, `app-config.yaml`, `package.json`, etc. **O `Dockerfile` já vem pronto** dentro de `packages/backend/Dockerfile`.

### 3.3 Passo 2 — Testar localmente sem Docker (recomendado)

```bash
cd infra-portal
yarn install
yarn start
```

Abre em `http://localhost:3000` (frontend) + `http://localhost:7007` (backend), com um banco SQLite em memória por padrão (`backend.database.client: better-sqlite3`). Serve para confirmar que o app está íntegro antes de complicar com Docker/Postgres/GitHub App — é o jeito mais rápido de isolar um problema (é do app ou é do Docker?).

**Como validar que o backend subiu de verdade** (e avisos que são normais):

- No log, o que importa é `rootHttpRouter info Listening on :7007` e `Plugin initialization complete`. A porta 7007 **não serve nenhuma página** — só rotas `/api/*` — então abrir `http://localhost:7007` direto retorna `404` propositalmente. Quem se abre no navegador é sempre a `3000`.
- `kubernetes warn Failed to initialize kubernetes backend: valid kubernetes config is missing` é esperado (`@backstage/plugin-kubernetes-backend` está registrado mas não há cluster) e inofensivo.
- `curl http://localhost:7007/api/catalog/entities` sem nada retorna **401** (`AuthenticationError: Missing credentials`) — esperado: todo endpoint exige credenciais. Para validar via terminal:
  ```bash
  TOKEN=$(curl -s -X POST http://localhost:7007/api/auth/guest/refresh | jq -r '.backstageIdentity.token')
  curl -s http://localhost:7007/api/catalog/entities -H "Authorization: Bearer $TOKEN" | jq .
  ```

### 3.4 Passo 3 — Ajustes de código que o `create-app` não faz sozinho

Um app recém-gerado **não loga com GitHub de verdade**, mesmo com um GitHub App configurado corretamente. Faltam três ajustes, nenhum incluído por padrão. Os três já estão feitos no `infra-portal` — esta seção documenta o que existe e por quê, para o caso de recriar o app do zero.

**1. Sobrescrever a tela de login.** O `create-app` usa o "novo sistema de frontend" (`createApp`/`features` em `packages/app/src/App.tsx`). A página de sign-in padrão vem **hardcoded só com o provider `guest`** — não lê `auth.providers` do `app-config.yaml`. É preciso sobrescrevê-la com um módulo próprio, já presente em `packages/app/src/modules/signIn/`:

```tsx
// packages/app/src/modules/signIn/SignInPage.tsx
import { githubAuthApiRef } from '@backstage/core-plugin-api';
import { SignInPage } from '@backstage/core-components';
import { SignInPageBlueprint } from '@backstage/plugin-app-react';

export const customSignInPage = SignInPageBlueprint.make({
  params: {
    loader: async () => props => (
      <SignInPage
        {...props}
        providers={[
          { id: 'github-auth-provider', title: 'GitHub', message: 'Entrar com sua conta do GitHub', apiRef: githubAuthApiRef },
          'guest',
        ]}
      />
    ),
  },
});
```

```ts
// packages/app/src/modules/signIn/index.ts
import { createFrontendModule } from '@backstage/frontend-plugin-api';
import { customSignInPage } from './SignInPage';

export const signInModule = createFrontendModule({ pluginId: 'app', extensions: [customSignInPage] });
```

E registrado no array `features` de `packages/app/src/App.tsx`, junto com `authPlugin` (de `@backstage/plugin-auth`, que fornece a rota `/oauth2` usada pelo popup de login):

```tsx
// packages/app/src/App.tsx
import authPlugin from '@backstage/plugin-auth';
import { signInModule } from './modules/signIn';

export default createApp({
  features: [catalogPlugin, authPlugin, navModule, homeModule, signInModule],
});
```

O mecanismo de override é o mesmo já usado para customizar a sidebar (`packages/app/src/modules/nav/`).

**2. Registrar o provider de auth do GitHub no backend.** `@backstage/plugin-auth-backend-module-github-provider` pode estar no `package.json` **sem nunca ter sido importado** em `packages/backend/src/index.ts`. Estar no `package.json` não registra o módulo:

```ts
// packages/backend/src/index.ts
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
```

Sem essa linha, `/api/auth/github/start` retorna erro e o backend nunca loga `"Configuring auth provider: github"` na subida.

**3. Registrar o módulo que sincroniza Teams/Members da organização.** `@backstage/plugin-catalog-backend-module-github-org` **não vem instalado por padrão**. Sem ele, `catalog.providers.githubOrg` é lida e ignorada silenciosamente: nenhuma entidade `User` real é criada, só o `guest` de exemplo. Sintoma: o resolver `usernameMatchingUserEntityName` nunca encontra o `User` do seu username → "unable to resolve user identity".

```ts
backend.add(import('@backstage/plugin-catalog-backend-module-github-org'));
```

**4. (Não é sobre login, é sobre catálogo de infraestrutura.)** Há um quarto módulo registrado, `@backstage/plugin-catalog-backend-module-github` — o discovery automático de `catalog-info.yaml` na org inteira, central para a Parte II:

```ts
backend.add(import('@backstage/plugin-catalog-backend-module-github'));
```

Depois de qualquer um desses ajustes, **nada tem efeito em produção sem rebuild** — ver a pegadinha do bundle desatualizado na seção 3.7.2.

### 3.5 Passo 4 — Criar o GitHub App

Feito manualmente na organização, **antes** de preencher configs — passo a passo na seção 4.3. Ao final você tem: `App ID`, `Client ID`, `Client Secret` e o arquivo `.pem` (private key). O `Webhook Secret` só é necessário se ativar o webhook — hoje está desativado.

### 3.6 Passo 5 — Preencher `.env` e `app-config.yaml`

```bash
# .env
GITHUB_APP_APP_ID=123456
GITHUB_APP_CLIENT_ID=Iv1.xxxxxxxxxxxx
GITHUB_APP_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
# GITHUB_APP_WEBHOOK_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  # só preencha (e só então referencie no app-config.yaml) se o webhook estiver ativo
GITHUB_APP_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
POSTGRES_PASSWORD=escolha-uma-senha-forte
```

Os blocos `integrations.github`, `auth.providers.github` e `catalog.providers.*` do `app-config.yaml` são mostrados por completo na seção 4.7.

### 3.7 Passos 6 e 7 — Build e subida via Docker Compose

O `infra-portal` usa **Docker Compose com 3 serviços** (`backstage`, `postgres`, `adminer`):

```yaml
# infra-portal/docker-compose.yml
services:
  backstage:
    build:
      context: .
      dockerfile: packages/backend/Dockerfile
    container_name: backstage
    ports:
      - "7007:7007"
    env_file:
      - .env
    environment:
      NODE_ENV: production
      POSTGRES_HOST: postgres
      POSTGRES_PORT: 5432
      POSTGRES_USER: backstage
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - backstage
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    container_name: backstage-postgres
    environment:
      POSTGRES_USER: backstage
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - backstage_postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U backstage"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - backstage
    restart: unless-stopped

  adminer:
    image: adminer:latest
    container_name: backstage-adminer
    ports:
      - "8081:8080"
    networks:
      - backstage
    restart: unless-stopped

networks:
  backstage:

volumes:
  backstage_postgres_data:
```

**O `packages/backend/Dockerfile` não compila o TypeScript dentro do container.** Ele só empacota artefatos gerados antes, no host, via `yarn build:backend`. Isso está documentado no próprio topo do `Dockerfile`:

```dockerfile
# Before building this image, be sure to have run the following commands in the repo root:
#
# yarn install --immutable
# yarn tsc
# yarn build:backend
...
COPY --chown=node:node packages/backend/dist/bundle.tar.gz app-config*.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz
...
CMD ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
```

Por isso a ordem correta é sempre:

```bash
yarn tsc                   # type-check
yarn build:backend         # passo 6 — gera packages/backend/dist/{bundle,skeleton}.tar.gz no host
docker compose build       # passo 7a — copia esses .tar.gz (e app-config*.yaml) pra dentro da imagem
docker compose up -d       # passo 7b — sobe backstage + postgres + adminer
```

Consequências práticas desse `Dockerfile`:

- `app-config.yaml` e `app-config.production.yaml` são copiados para dentro da imagem no build — **não há bind mount** deles no `docker-compose.yml`. Um `docker compose restart` sozinho **não** aplica mudanças de config; é preciso rebuildar a imagem.
- O `CMD` com dois `--config` é a razão da regra de merge de config da seção 4.8.
- `docker compose restart backstage` serve, por outro lado, pra **descartar o installation token cacheado do GitHub App** (seção 4.5) — reinicia o processo, não reconstrói a imagem.

#### 3.7.1 Erro comum: `docker compose build` falha com `"/packages/backend/dist/skeleton.tar.gz": not found`

```
failed to solve: failed to compute cache key: failed to calculate checksum of ref ...:
"/packages/backend/dist/skeleton.tar.gz": not found
```

**Causa:** o `Dockerfile` faz `COPY` de `dist/skeleton.tar.gz` e `dist/bundle.tar.gz`, que só existem depois de `yarn build:backend` no host. **Correção:** rodar o build do backend antes; confirmar com `ls -la packages/backend/dist/*.tar.gz`.

#### 3.7.2 Erro comum: módulo novo no backend não aparece rodando (bundle desatualizado)

Mais traiçoeiro que o anterior: o `dist/bundle.tar.gz` **existe**, o build passa, o container sobe sem erro — mas um módulo recém-adicionado em `packages/backend/src/index.ts` simplesmente não aparece nos logs, sem nenhuma mensagem.

**Causa:** o `Dockerfile` copia um `dist/bundle.tar.gz` **pré-existente**, gerado da última vez que `yarn build:backend` rodou. Se você editar `src/index.ts` e for direto para `docker compose build`/`up`, a imagem sobe com o bundle **antigo**. `docker compose restart` reinicia o *processo*, não reconstrói a imagem — o bundle desatualizado continua sendo servido indefinidamente.

Caso real (2026-09-02), ao confirmar que o `acmeProvider` estava escaneando a org: só `GithubMultiOrgEntityProvider:prd:refresh` nos logs, nenhuma menção ao `acmeProvider`. Config correta dentro do container (`/app/app-config.production.yaml`), módulo no `package.json` e no `index.ts`:

| Arquivo | Última modificação |
|---|---|
| `packages/backend/src/index.ts` (adicionou o `backend.add(...github)`) | 2026-08-27 |
| `packages/backend/dist/bundle.tar.gz` (usado pelo `docker build`) | 2026-08-21 |

Bundle 6 dias mais antigo que o código.

**Correção**, sempre que mexer em `packages/backend/src/index.ts`:

```bash
yarn tsc && yarn build:backend
docker compose build backstage
docker compose up -d backstage
```

**Como confirmar que o módulo está no bundle**, antes mesmo de subir:

```bash
tar xzf packages/backend/dist/bundle.tar.gz -O packages/backend/dist/index.cjs.js \
  | grep -o "plugin-catalog-backend-module-github[a-zA-Z-]*" | sort -u
# esperado:
# plugin-catalog-backend-module-github
# plugin-catalog-backend-module-github-org
```

E, depois de subir:

```bash
docker logs backstage | grep "Registered scheduled task"
# esperado (entre outras):
# Registered scheduled task: github-provider:acmeProvider:refresh, cadence PT1M
```

Silêncio total (sem task nova, sem erro) é a assinatura deste problema — não confundir com "provider desabilitado por config".

#### 3.7.3 Dois erros de config que travam o boot (antes de qualquer log de plugin)

- **`POSTGRES_USER` ausente** — `app-config.production.yaml` referencia `${POSTGRES_USER}`, mas esse valor só existe hardcoded no serviço `postgres` do Compose. Serviços do Compose **não herdam** variáveis uns dos outros — o serviço `backstage` precisa repetir `POSTGRES_USER: backstage` no próprio `environment:` (como no exemplo acima).
- **`webhookSecret` referenciado sem a variável existir** — se `app-config.yaml` tiver `webhookSecret: ${GITHUB_APP_WEBHOOK_SECRET}` mas a chave estiver ausente no `.env`, o backend derruba com "couldn't find env var". Como o campo é opcional, a correção é não referenciá-lo enquanto o webhook estiver desativado. É o estado atual do `infra-portal`.

### 3.8 Passo 8 — Validar

```bash
docker compose logs -f backstage
```

Espere `Backend successfully started` (o backend roda as migrations do Postgres automaticamente na primeira subida). Acesse `http://localhost:7007` e teste o login via GitHub — se o `signIn.resolver` da seção 4.7 estiver certo e seu usuário já tiver sido sincronizado como `User`, funciona de primeira. Se falhar, ver seção 4.11.

**Pontos importantes sobre o Compose:**

- **`adminer`** é opcional — útil para debug do catálogo (tabelas `entities`, `refresh_state`, `final_entities`, `backstage_backend_tasks__tasks`), em `http://localhost:8081`.
- **Volume nomeado** para o Postgres é obrigatório — sem ele, cada `docker compose down` apaga o catálogo inteiro.
- **Backup do Postgres**: agende um `cron` local rodando `docker exec backstage-postgres pg_dump -U backstage backstage > backup_$(date +%F).sql`.
- **TLS/proxy reverso**: se isso for exposto fora do localhost, coloque um Nginx/Traefik na frente — o Backstage não deve terminar TLS.
- **Migração futura para Kubernetes**: os serviços já estão isolados, a migração para Helm/K8s é direta (mesma imagem, mesmas envs).

### 3.9 Comandos do dia a dia (`infra-portal`)

Rodar sempre a partir de `infra-portal/`:

```sh
yarn install
yarn start              # dev, frontend (3000) + backend (7007)
yarn build:backend      # build de produção do backend (gera dist/*.tar.gz)
yarn build-image        # build:backend + build da imagem Docker
yarn tsc                # typecheck (incremental)
yarn lint                # lint só do que mudou desde origin/main
yarn lint:all
yarn test                # backstage-cli repo test (jest, workspace-aware)
yarn test:all            # com coverage
yarn test:e2e            # playwright (playwright.config.ts)
yarn fix                  # autofix de lint
```

Teste único: `yarn test packages/app/src/App.test.tsx`.

---

## 4. Integração com GitHub Organization

A org `acme` usa uma **Organization**, não conta pessoal — isso dá controle centralizado de quem tem acesso, e permite sincronizar Teams como Grupos no catálogo automaticamente (seção 12).

### 4.1 GitHub App em vez de OAuth App ou PAT

Use um **GitHub App** instalado na organização — é o que o `infra-portal` usa (`Portal-Infra-ACME`, owner `acme`, em `integrations.github[0].apps`). Motivos:

- Permissões granulares por recurso (conteúdo, PRs, workflows) em vez de escopos genéricos.
- Instalação restrita a repositórios específicos da org, se quiser (hoje: `repository_selection: all` — todos, presentes e futuros).
- Rate limit por instalação, maior e mais estável que token pessoal (um App sem instalação, porém, cai para o rate limit de IP anônimo — seção 4.11, item 3).
- Não expira nem fica atrelado à saída de um funcionário.
- Suporta tanto autenticação de usuários (login) quanto automação (criar PRs, ler catálogo) com a mesma identidade.

**Ponto central:** o Scaffolder e o discovery autenticam **como a App**, não como o usuário logado. Permissão individual do usuário no GitHub é irrelevante pra essas escritas (seção 6 discute o que isso implica).

### 4.2 Criando e organizando Teams e Members na Organization

A seção 12 explica como o Backstage **sincroniza** Teams/Members como `Group`/`User` — mas isso pressupõe que a estrutura já existe no GitHub. Crie-a **antes ou em paralelo** de criar o GitHub App.

**4.2.1 Convidando membros**

1. Na organização: **Settings → People → Invite member**.
2. Convide pelo username ou e-mail.
3. Defina o **role**: **Member** (suficiente para logar no Backstage e ser sincronizado como `User`) ou **Owner** (administrativo total — reserve para o time de plataforma).
4. A pessoa precisa **aceitar o convite** — convites pendentes não são lidos pelo provider `githubOrg`.

**4.2.2 Criando Teams (viram `Group` no catálogo)**

1. Na organização: **Teams → New team**.
2. **Team name** vira o *slug* (ex.: "Time Financeiro" → `time-financeiro`) — literalmente o `metadata.name` do `Group` (`group:default/time-financeiro`). Escolha já pensando no que vai em `spec.owner` nos `catalog-info.yaml`.
3. **Parent team** (opcional) — popula `spec.parent`/`spec.children` automaticamente (seção 12.3); não edite esses campos no catálogo, edite a hierarquia no GitHub.
4. **Visibility**: prefira `Visible`. Um time `Secret` ainda é lido pela API com `Organization → Members: Read`, mas fica escondido da navegação do GitHub para não-membros.
5. Adicione os **membros**, definindo **Maintainer** vs. **Member**. O Backstage não distingue esse papel hoje (todos entram em `spec.members`), mas facilita configurar o Permission framework (seção 10) depois.

**4.2.3 Checklist antes de confiar no sync**

- [ ] Todo mundo que vai logar já **aceitou** o convite.
- [ ] Todo Team que deve virar `Group` já existe, com o **slug** pretendido.
- [ ] Hierarquia de Teams já reflete a estrutura real, se for usar.
- [ ] O GitHub App já foi **instalado** na organização — sem isso, nada é lido (seção 4.11, item 3).

Depois, force um refresh manual (seção 4.11, item 3) e confirme em `http://localhost:7007/catalog?filters%5Bkind%5D=group`.

### 4.3 Passo a passo de criação do GitHub App

1. Na organização: **Settings → Developer settings → GitHub Apps → New GitHub App**.
2. **Homepage URL**: `http://localhost:7007` (ou o domínio que for usar).
3. **Callback URL**: `http://localhost:7007/api/auth/github/handler/frame`.
4. **Webhook**: deixe **desativado** enquanto rodar em máquina local sem exposição pública (estado atual) — confie no polling do catálogo (seção 13). Só ative apontando para `http://.../api/catalog/github` se expor a URL publicamente (túnel `ngrok`/`cloudflared`).
5. **Permissions** — ver a tabela completa da seção 4.4.
6. **Where can this GitHub App be installed?** → "Only on this account".
7. Gere e baixe a **private key** (`.pem`), guarde o **App ID** e **Client ID/Secret**.
8. **Instale o App na organização**, escolhendo "All repositories" ou uma lista — sem este passo, o provider falha por rate limit (seção 4.11, item 3), mesmo com credenciais corretas.

### 4.4 Permissões concedidas hoje e por quê

Todas foram descobertas empiricamente, uma por uma, cada falta gerando um `403`/"Resource not accessible by integration" (histórico na seção 14):

| Permissão | Nível | Exigida por |
|---|---|---|
| Repository → Metadata | Read | base (sempre) |
| Repository → Contents ("code") | Read and write | `GithubUrlReader` (baixar skeleton), Git Data API (`publish:github:pull-request` cria blobs/trees/commits), `publish:github` (commit na `main`), discovery |
| Repository → Commit statuses | Read-only | `GithubUrlReader.getRepoDetails` (`GET .../commits/main/status`) ao resolver a branch antes de baixar o skeleton |
| Repository → Pull requests | Read and write | `publish:github:pull-request` |
| Repository → Workflows | Read and write | Criar/alterar arquivos em `.github/workflows/` via PR — "Contents: write" **não basta** pra esse caminho |
| **Repository → Administration** | Read and write | `publish:github` (`POST /orgs/{org}/repos`, criar repositório) — só o `infra-repo-bootstrap` exercita isso. **Atenção:** existe também uma permissão de *organização* chamada "Administration" (`organization_administration`) — é diferente e **não** resolve esse caso |
| Repository → Checks | Read | — |
| Organization → Members | Read | provider `githubOrg` (Teams/Members) |

`organization_administration: write` também foi concedida numa tentativa que **não** resolveu a criação de repositório (seção 14, erro 14.11) — não precisa ser revertida, mas é uma decisão à parte mantê-la.

⚠️ `Administration` (em qualquer dos dois níveis) é ampla: além de criar repositório, permite apagar, transferir e alterar branch protection/webhooks em qualquer repo alcançado pela instalação (aqui, todos).

### 4.5 Procedimento obrigatório ao adicionar uma permissão (3 passos)

Pular qualquer um mantém o `403`/"Resource not accessible by integration":

1. **Salvar no App**: `github.com/organizations/acme/settings/apps/portal-infra-acme/permissions`. Confirmar em "You are about to change the permissions..." → **Accept new permissions**.
2. **Aceitar a atualização na instalação da org**: `github.com/organizations/acme/settings/installations` → App → *Configure* (aparece um banner de permissões pendentes). Editar o App **não propaga sozinho** para instalações existentes.
3. **Reiniciar o backend** (`docker compose restart backstage` em `infra-portal/`): o Backstage cacheia o installation access token (válido por até 1h); um token emitido antes da mudança não ganha o novo escopo retroativamente.

### 4.6 Diagnóstico: ver exatamente quais permissões a App tem

Em vez de adivinhar, consultar a API com um JWT assinado pela chave privada da App (RS256, `iss=appId`):

```bash
curl -s -H "Authorization: Bearer $JWT" \
     -H "Accept: application/vnd.github+json" \
     https://api.github.com/app
# → campo "permissions": { contents: write, statuses: read, ... }
```

Comparar com a [tabela oficial de permissões exigidas por endpoint](https://docs.github.com/en/rest/overview/permissions-required-for-github-apps). Lição principal: quando um nome de permissão aparece em mais de uma seção da tela (repositório vs. organização), checar as duas — a chave JSON (`administration` vs. `organization_administration`) desambigua. A API permite **ler** permissões, não alterar — isso só existe na UI e só um owner da org consegue.

### 4.7 Configuração no `app-config.yaml`

Conteúdo real, hoje, das duas configs do `infra-portal` (`app-config.yaml` para dev, `app-config.production.yaml` sobrepondo para produção — ver a regra de merge na seção 4.8):

```yaml
# infra-portal/app-config.yaml (dev)
integrations:
  github:
    - host: github.com
      apps:
        - appId: ${GITHUB_APP_APP_ID}
          clientId: ${GITHUB_APP_CLIENT_ID}
          clientSecret: ${GITHUB_APP_CLIENT_SECRET}
          privateKey: ${GITHUB_APP_PRIVATE_KEY}
          # webhookSecret: ${GITHUB_APP_WEBHOOK_SECRET}  # opcional — só inclua se o webhook estiver ativo

auth:
  environment: development
  providers:
    github:
      development:
        clientId: ${GITHUB_APP_CLIENT_ID}
        clientSecret: ${GITHUB_APP_CLIENT_SECRET}
        signIn:
          resolvers:
            - resolver: usernameMatchingUserEntityName

catalog:
  rules:
    - allow: [Component, System, API, Resource, Location]
  locations:
    - type: file
      target: ../../examples/entities.yaml
    - type: url
      target: https://github.com/acme/infra-cloud/blob/main/aws/s3-bucket/template.yaml
      rules:
        - allow: [Template]
    - type: url
      target: https://github.com/acme/infra-cloud/blob/main/aws/vpc/template.yaml
      rules:
        - allow: [Template]
    - type: url
      target: https://github.com/acme/infra-cloud/blob/main/infra-repo-bootstrap/template.yaml
      rules:
        - allow: [Template]
    - type: file
      target: ../../examples/org.yaml
      rules:
        - allow: [User, Group]
  providers:
    githubOrg:
      id: dev                      # prd em app-config.production.yaml
      githubUrl: https://github.com
      orgs: [acme]
      schedule:
        frequency: { seconds: 30 } # prod: 30 min / 3 min
        timeout: { seconds: 20 }
    github:
      acmeProvider:
        organization: 'acme'
        # catalog-info.yaml não fica na raiz dos repos de infra (ex.
        # infra/<resourceName>/catalog-info.yaml), então o catalogPath
        # precisa ser um glob recursivo, não o default "/catalog-info.yaml"
        catalogPath: '**/catalog-info.yaml'
        filters:
          branch: 'main'
        schedule:
          frequency: { minutes: 1 }
          timeout: { seconds: 30 }
```

Pontos que costumam confundir:

- **`signIn.resolvers: usernameMatchingUserEntityName`** faz o login funcionar casando o username do GitHub com a entidade `User` sincronizada — sem isso, o login falha mesmo com o OAuth funcionando.
- **Ter essas chaves no `app-config.yaml` não é suficiente.** Tanto o provider de auth quanto os dois de catálogo só funcionam se os módulos estiverem **instalados _e_ registrados** em `packages/backend/src/index.ts` (seção 3.4) — e se o bundle usado pelo container for **recente** (seção 3.7.2).
- As locations `type: file` de `examples/` são dados de exemplo do `create-app` — ver seção 12.4 para quando removê-las.

### 4.8 Merge de `app-config.yaml` × `app-config.production.yaml`

O `CMD` do Dockerfile sobe com `--config app-config.yaml --config app-config.production.yaml`. O Backstage faz merge recursivo de **objetos**, mas **substitui arrays inteiros** pelo último arquivo que os define — não há merge item a item. Como `app-config.production.yaml` define seu próprio `catalog.locations`, qualquer location adicionada só em `app-config.yaml` é **descartada silenciosamente** em produção (o template aparece em `yarn start`, nunca no Docker, sem erro nos logs).

**Regra:** toda vez que adicionar/remover uma location (principalmente `type: url` de templates), replicar nos dois arquivos, com o mesmo `rules: [allow: Template]`. `app-config.production.yaml` tem hoje um bloco `catalog.locations` idêntico em conteúdo ao de dev, exceto pelos paths `type: file` (`./examples/...` em vez de `../../examples/...`, porque o `WORKDIR` do container é `/app`) e pelo `id`/`schedule` dos providers.

Não é possível eliminar a duplicação removendo o override de produção, justamente por causa dos paths `type: file`. A alternativa seria um `app-config.templates.yaml` dedicado carregado via `--config` extra nos dois ambientes — não feito por ser mudança maior; se a duplicação virar dor recorrente, é a próxima opção.

`catalog.providers`, por ser um mapa de chaves nomeadas (`acmeProvider`, `dev`/`prd`), não sofre esse problema — mas ao adicionar um provider, replicar nos dois arquivos do mesmo jeito.

Esse bug já aconteceu **duas vezes** (2026-08-26 com `aws-s3-bucket`, 2026-09-09 com `aws-vpc`) — por isso o item entrou no checklist obrigatório de novos templates (seção 15).

### 4.9 `catalog.rules` e locations de `Template`

`catalog.rules` global (fora de qualquer location) restringe quais `kind` podem ser registrados via location "solta": por padrão só `Component`/`System`/`API`/`Resource`/`Location`. `Template`, `User` e `Group` ficam de fora por segurança — um Template pode disparar execução de ações arbitrárias. O botão "Register Existing Component" cria uma Location genérica, sujeita à regra global, e retorna:

```
NotAllowedError: Entity template:default/aws-s3-bucket at
url:https://github.com/acme/infra-cloud/blob/main/aws-s3-bucket/template.yaml
is not of an allowed kind for that location
```

Por isso cada location de template/org precisa do próprio `rules: [allow: ...]` explícito (seção 4.7). Locations estáticas são lidas na inicialização — editar o arquivo com o processo rodando não basta (e em Docker, exige rebuild, seção 3.7).

### 4.10 Providers do catálogo: `githubOrg` vs. `github`

Dois providers dinâmicos, **complementares** (não substitutos):

| Provider | Módulo | Entidades que produz | O que varre |
|---|---|---|---|
| `catalog.providers.githubOrg` (`id: dev`/`prd`) | `@backstage/plugin-catalog-backend-module-github-org` | `Group` / `User` | Teams e membros da org (seção 12.1) |
| `catalog.providers.github.acmeProvider` | `@backstage/plugin-catalog-backend-module-github` | Qualquer `kind` do `catalog-info.yaml` encontrado (`Component`, `Resource`, `System`…) | Repositórios da org, via `catalogPath` na branch `main` |

Sem o `githubOrg`, os `owner:` dos `catalog-info.yaml` descobertos apontariam para `Group`s inexistentes.

O `acmeProvider` é o que faz um recurso provisionado pelo Scaffolder aparecer no catálogo **sem** step `catalog:register` — assim que o `catalog-info.yaml` existe na `main` de algum repositório da org (depois do merge), o provider encontra e registra sozinho, no próximo ciclo (até 1 min). Cria quando aparece, atualiza quando muda, **remove quando o repo/arquivo some**.

Efeito confirmado (2026-09-02) nos logs e no Postgres:

```
Registered scheduled task: github-provider:acmeProvider:refresh, cadence PT1M
Read 6 GitHub repositories (6 matching the pattern)
```

```sh
docker exec backstage-postgres psql -U backstage -d backstage_plugin_catalog \
  -c "SELECT entity_ref FROM final_entities;"
# → component:default/infra-portal (descoberto do catalog-info.yaml do próprio portal)
```

**Efeito colateral inofensivo:** o glob também encontra os `catalog-info.yaml` **não renderizados** dentro dos skeletons no `infra-cloud` (com placeholders `${{ values.resourceName }}`). O catálogo rejeita na validação de política e gera só um `warn`:

```
Policy check failed for component:default/${{ values.name | dump }}; ...
```

Não é bug. Se o ruído incomodar, restringir o discovery por topic do GitHub ou `filters.repoSlug`, excluindo `infra-portal` e `infra-cloud`.

### 4.11 Troubleshooting de login e de sincronização

Depois que os ajustes de código (seção 3.4) e a config (seção 4.7) estão em vigor, os problemas que sobram são estes:

**1. Schema errado do `catalog.providers.githubOrg` derruba o backend no boot.** Se o schema for antigo (`orgUrl` aninhado sob uma chave arbitrária), o backend crasha com `Missing required config value at 'catalog.providers.githubOrg.id'`. Use o schema da seção 4.7 (`id` + `githubUrl` + `orgs: [...]`) — schema atual do módulo "multi-org".

**2. GitHub App configurado, mas nunca instalado na organização.** O provider falha com `API rate limit exceeded for <seu IP>` num request GraphQL — parece limite de taxa, mas o header `x-ratelimit-limit: 0` é a assinatura de uma chamada **sem autenticação nenhuma**. Sem instalação, o Backstage não consegue emitir um installation token e a chamada sai anônima. Correção: `https://github.com/organizations/<org>/settings/installations` → Install.

**3. Forçar sincronização imediata do `githubOrg`.** Depois de instalar o App, o catálogo não ressincroniza na hora: o agendador (`backend-tasks`) **persiste o próximo horário no Postgres** (`backstage_backend_tasks__tasks`), então reiniciar o container não força nova tentativa antes do `cadence`:

```bash
docker exec backstage-postgres psql -U backstage -d backstage_plugin_catalog -c \
  "update backstage_backend_tasks__tasks set next_run_start_at = now() where id = 'GithubMultiOrgEntityProvider:<id-do-provider>:refresh';"
```

(`<id-do-provider>` é `dev` em dev ou `prd` em produção.) O log deve mostrar `Reading GitHub users and teams for org: acme` seguido de `Committed N GitHub users`.

**Checklist rápido de diagnóstico:**

| Sintoma | Onde olhar | O que procurar |
|---|---|---|
| Só aparece "Guest" na tela de login | `packages/app/src/App.tsx` | `signInModule` está no `features`? (seção 3.4, item 1) |
| Botão do GitHub aparece mas dá erro antes de redirecionar | `docker logs backstage` | `"Configuring auth provider: github"` apareceu na subida? (seção 3.4, item 2) |
| Login autentica no GitHub mas retorna "unable to resolve user identity" | `docker logs backstage` | `GET /api/catalog/entities/by-name/User/default/<seu-usuário>` retornou `404` ou `200`? (seção 3.4, item 3) |
| Provider de catálogo não sincroniza nada | `docker logs backstage` | `refresh failed` com `rate limit` e `x-ratelimit-limit: 0`? → App não instalado na org |
| Backend crasha no boot mencionando `catalog.providers.githubOrg` | `docker logs backstage` | `Missing required config value at 'catalog.providers.githubOrg.id'` → schema errado |
| `OwnerPicker` só mostra "guest" | UI do template | Ver seção 12.4 |
| Módulo novo no backend não aparece nos logs, sem erro | `docker logs backstage` | Bundle desatualizado — seção 3.7.2 |
| Scaffolder falha com `403`/"Resource not accessible by integration" | Resultado do step | Permissão faltando na App — seções 4.4/4.5 e 14 |

---

# Parte II — Portal de infraestrutura com Terraform

## 5. Separação entre "gerar código" e "aplicar infra"

Esse é o objetivo final do portal: alguém pede um recurso (bucket S3, VPC) pelo Backstage, e sai um PR com o código Terraform pronto, revisado por pipeline.

### 5.1 O Backstage nunca roda `terraform apply`

Esta é a decisão de design mais importante do portal. O Scaffolder é responsável só por *gerar e propor* a mudança; quem *aplica* é sempre um pipeline de CI/CD rodando **fora** do Backstage, no repositório de destino, depois de revisão humana.

**Por quê:**

1. **Credenciais de apply nunca ficam no container do Backstage.** Isolando o `apply` no CI/CD, as credenciais de nuvem ficam só lá, via OIDC sem secret de longa duração (seção 9.1), e o Backstage só precisa de permissão para abrir PR/criar repositório.
2. **Revisão humana antes de qualquer mudança real.** O `terraform plan` publicado como comentário no PR dá a qualquer pessoa do time a chance de olhar antes de aprovar.
3. **Histórico de mudança fica no Git**, não em logs do Backstage.
4. **Cada repositório/time pode ter sua própria política de aprovação** (GitHub Environments), sem mexer no Backstage.

**Quem faz o quê:**

| Etapa | O que acontece | Onde roda | Responsável |
|---|---|---|---|
| 1 | Coletar parâmetros do usuário | Backstage — formulário do Scaffolder | Scaffolder |
| 2 | Gerar o código Terraform a partir do `skeleton/`, referenciando um módulo fixo de `terraform-modules` | Backstage — `fetch:template` | Scaffolder |
| 3 | Abrir um PR no repositório do time (`publish:github:pull-request`) — ou, para o primeiro recurso de um time novo, criar o repositório com commit direto na `main` (`publish:github`, `infra-repo-bootstrap`) | Backstage — `publish:github*` | Scaffolder |
| 4 | Aprovar (ou pedir mudanças) no PR — o `catalog-info.yaml` gerado ainda só existe na branch do PR | GitHub, review do PR | Pessoa do time |
| 5 | `terraform plan` no PR; após o merge, `apply` manual via `workflow_dispatch` | GitHub Actions | CI/CD |
| 6 | **Só agora** o `catalog-info.yaml` existe na `main` — o `acmeProvider` encontra e registra a `Resource` sozinho (até 1 min) | Backstage — catalog discovery | Backstage |

```
Usuário no Backstage
   → Scaffolder Template (aws-s3-bucket, aws-vpc, infra-repo-bootstrap)
      → [Backstage] gera .tf a partir de terraform-modules      (etapas 1-2)
      → [Backstage] abre PR (ou cria repo) em <repo-do-time>     (etapa 3)
         → [Humano] aprovação do PR                              (etapa 4)
            → [GitHub Actions] terraform plan (PR) / apply (manual)  (etapa 5)
               → [Backstage] discovery encontra catalog-info.yaml
                             na main e registra a Resource         (etapa 6)
```

### 5.2 Criar repositório (Scaffolder) vs. descobrir repositório (Catalog Discovery)

> Atualmente o Backstage está configurado para **criar** repositório. Como seria a configuração para **descobrir** repositório no GitHub? Qual a melhor prática?

Resposta curta: **as duas coisas coexistem**, em pontos diferentes do config, e são mecanismos diferentes.

| | "Criar" (action do Scaffolder) | "Descobrir" (`GithubEntityProvider`) |
|---|---|---|
| Onde vive | `steps` do `template.yaml` (`publish:github`, `publish:github:pull-request`) | `catalog.providers.github` no `app-config*.yaml` |
| Quando roda | Uma vez, no clique de "Create" | Continuamente, no schedule (1 min) |
| Fonte da verdade | O `step` do template | O conteúdo real do repositório na branch `main` |
| Repos cobertos | Só o que o Scaffolder acabou de tocar | **Toda** a org, incluindo repos criados fora do Backstage |
| Entidade removida se repo/arquivo some? | Não (`Location` fica órfã) — no caso de `catalog:register` | Sim, automaticamente |
| Timing vs. revisão humana | `catalog:register` registraria **antes** do merge | Só aparece **depois** do `catalog-info.yaml` estar na `main` |

```mermaid
flowchart LR
    subgraph criar["Criar (Scaffolder)"]
        A[Template step: publish:github / publish:github:pull-request] -->|push / PR| R1[Repo no GitHub]
    end
    subgraph descobrir["Descobrir (Catalog Discovery)"]
        P[GithubEntityProvider<br/>schedule a cada 1 min] -->|escaneia| ORG[(Org GitHub: acme)]
        ORG -->|encontra **/catalog-info.yaml na main| E2[Entidade no Catalog]
        ORG -.->|arquivo/repo removido| DEL[Entidade removida automaticamente]
    end
```

**Melhor prática (e o que já está certo aqui):**

1. **Prefira discovery a `catalog:register`.** Nenhum template da org usa `catalog:register`; o `acmeProvider` cobre todos. Resolve o problema de timing sem reordenar steps (impossível — o Scaffolder é síncrono).
2. **GitHub App, não PAT** (seção 4.1).
3. **Restrinja o escopo do scan** com `catalogPath` + `filters.branch` (feito). Em orgs maiores, `filters.repoSlug` ou topic.
4. **Schedule**: 1 min está adequado.
5. **Combine `githubOrg` + `github`** (seção 4.10).
6. **Atenção ao merge de arrays** entre os dois `app-config` (seção 4.8).

### 5.3 Backend de estado do Terraform

Todo skeleton gera um `backend.tf` apontando para o **mesmo** bucket S3, compartilhado por toda a org — o que muda por template é a `key`:

```hcl
# infra-cloud/aws/s3-bucket/skeleton/terraform/backend.tf
terraform {
  backend "s3" {
    bucket  = "acme-terraform-tfstate"
    key     = "infra/${{ values.resourceName }}/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true
  }
}
```

```hcl
# infra-cloud/aws/vpc/skeleton/terraform/backend.tf
terraform {
  backend "s3" {
    bucket  = "acme-terraform-tfstate"
    key     = "infra/vpc-${{ values.resourceName }}/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true
  }
}
```

- **A `key` precisa incluir algo que identifique o tipo de recurso, não só o `resourceName`.** Dois templates com a mesma convenção de `key` colidiriam no mesmo state. O de VPC já usa o prefixo `vpc-`; ao criar um novo template, replicar essa convenção é obrigatório.
- **Não há tabela DynamoDB de lock configurada hoje** (`dynamodb_table` ausente). É um ponto em aberto, não uma escolha documentada: avaliar lock (DynamoDB, ou o lock nativo via S3 do Terraform ≥ 1.10) antes de haver aplicações concorrentes.
- A IAM role do pipeline (seção 9.1) precisa de leitura/escrita nesse bucket, além das permissões do recurso em si.

### 5.4 Organização por provider

```
terraform-modules/
├── aws/
│   ├── s3-bucket/     # tag publicada: aws-s3-bucket/v0.1.0
│   └── vpc/           # tag publicada: aws-vpc/v0.1.0
└── azure/             # pasta reservada, sem conteúdo ainda
```

Os templates do Scaffolder seguem a mesma organização — o provider está implícito no path (`infra-cloud/aws/s3-bucket/`, `aws/vpc/`), em vez de ser um parâmetro dentro de um template genérico. Quando o Azure entrar: `infra-cloud/azure/<recurso>/` + `terraform-modules/azure/<recurso>/`, replicando `skeleton/terraform/` + `skeleton/workflow/`.

---

## 6. Modelo de repositórios: um repositório de infra por time

Os templates de recurso (`aws-s3-bucket`, `aws-vpc` e qualquer outro no mesmo padrão) **não criam um repositório por recurso**. Eles abrem um PR num repositório já existente, colocando o Terraform gerado numa subpasta:

```
infra/<nome-do-recurso>/
```

| Time | Repositório | Recurso criado | Caminho resultante |
|---|---|---|---|
| `infra-aws` | `acme/aws` | VPC `vpc-prod` | `aws/infra/vpc-prod/` |
| `infra-azure` | `acme/infra-azure` | VM `vm-web01` | `infra-azure/infra/vm-web01/` |

Cada repositório acumula quantos recursos o time criar, todos via PR e com o mesmo workflow de CI na raiz, compartilhado. É a razão de existir do `infra-repo-bootstrap` (seção 7.2): criar esse repositório com um commit inicial **uma vez por time** — porque `publish:github:pull-request` não consegue abrir PR num repositório totalmente vazio.

Hoje existe o repositório `acme/aws` (criado via bootstrap; um `AWS` maiúsculo foi criado antes e removido — ver seção 9.1).

### O ponto frágil: `owner` e `repoURL` são campos independentes

No formulário dos templates de recurso, `owner` (`OwnerPicker`) e `repoURL` (`RepoUrlPicker`) são campos separados, **sem validação cruzada**. Nada impede `owner: infra-aws` + `repoURL: acme/infra-azure` — um PR de recurso AWS cairia no repositório do time errado.

### Por que restringir permissão no GitHub não resolve

Quem abre o PR é a **GitHub App**, não a pessoa (seção 4.1). A App está instalada com acesso a todos os repos da org. Permissão individual do usuário é irrelevante — mesmo sem push no `infra-azure`, o PR seria aberto. Branch protection ajuda só como rede tardia: o PR errado já foi aberto, notificando o time errado; bloqueia o merge, não a abertura.

### O que resolve: tirar a escolha errada da tela

Derivar as opções de `repoURL` a partir do `owner`, usando o catálogo como fonte de verdade. Viável **depois** do `infra-repo-bootstrap`: cada repo de time sobe com um `catalog-info.yaml` `kind: System`, `spec.owner: <time>`, que o discovery registra. O `repoURL` poderia virar um `ui:field: EntityPicker` filtrado por `kind: System` **e** `spec.owner == parameters.owner` — a opção errada nem aparece.

**Não implementado.** Depende de ao menos um time ter passado pelo bootstrap e de decidir se o filtro por `owner` é obrigatório ou sugestão (times pequenos podem compartilhar repositório).

| Pergunta | Resposta |
|---|---|
| Um repo por time, N recursos dentro? | Sim — convenção (`targetPath: infra/<recurso>`), não imposta por schema |
| Dá pra restringir via permissão do GitHub? | Não — a automação roda como a App |
| O que evita PR no repo errado? | `EntityPicker` filtrado por `owner` — não implementado |
| Branch protection ajuda? | Só pós-fato (bloqueia merge, não abertura) |

---

## 7. Templates do `infra-cloud`

Cada template é uma pasta com `template.yaml` (`scaffolder.backstage.io/v1beta3`) + `skeleton/`.

| Template | Action de publicação | Cria repo? | Gera |
|---|---|---|---|
| `infra-repo-bootstrap/` | `publish:github` (commit direto na `main`) | **Sim** | README + `catalog-info.yaml` (`System`) + `.gitignore` |
| `aws/s3-bucket/` | `publish:github:pull-request` | Não | `infra/<nome>/{main.tf, backend.tf, provider.tf, catalog-info.yaml}` + workflow na raiz |
| `aws/vpc/` | `publish:github:pull-request` | Não | idem, consumindo `terraform-modules//aws/vpc` |

```
infra-cloud/
├── aws/s3-bucket/
│   ├── template.yaml                        # lido pelo Backstage (Scaffolder)
│   └── skeleton/
│       ├── terraform/                       # copiado para infra/<resourceName>/ no repo de destino
│       │   ├── main.tf                      # chama terraform-modules com ref fixo (seção 1.3)
│       │   ├── backend.tf                   # seção 5.3
│       │   ├── provider.tf
│       │   └── catalog-info.yaml            # entidade Resource, descoberta pelo acmeProvider
│       └── workflow/
│           └── .github/workflows/terraform-ci-cd.yml   # copiado para a raiz do repo de destino
├── aws/vpc/                                 # mesma estrutura
└── infra-repo-bootstrap/
    ├── template.yaml
    └── skeleton/
        ├── README.md
        ├── catalog-info.yaml                # entidade System do próprio repositório criado
        └── .gitignore
```

Nenhum usa `catalog:register` (seção 5.2).

Ao mover/renomear a pasta de um template, atualizar em conjunto: (1) os comentários de path internos ao `template.yaml`/skeleton; (2) as URLs nos dois `infra-portal/app-config*.yaml`; (3) os links na documentação.

### 7.1 Anatomia comum

#### Schema `v1beta3`: tudo dentro de `spec`

A raiz só aceita `apiVersion`, `kind`, `metadata`, `spec`. `parameters`, `steps` e `output` **precisam** estar dentro de `spec` — fora dela o catálogo rejeita com `Malformed envelope, <root> must NOT have additional properties` (seção 14, erro 14.2). Todos os três templates estão corretos hoje.

#### `fetch:template`, Nunjucks e `values` vs. `parameters`

O Scaffolder usa [Nunjucks](https://mozilla.github.io/nunjucks/) (port do Jinja2 pra Node) na action `fetch:template` pra renderizar **todo arquivo do skeleton** antes de copiá-lo. O delimitador é `${{ }}` (com `$`), não o `{{ }}` padrão — justamente pra reduzir colisão com outras sintaxes de chave dupla.

São dois contextos diferentes:

- **`parameters.*`** só existe dentro do `template.yaml` — o que o usuário preencheu no formulário.
- **`values.*`** só existe dentro dos arquivos do `skeleton/`. **Não é automático**: existe porque o step `fetch:template` copiou explicitamente cada `parameters.X` para `input.values.X`. Se um parâmetro não for listado ali, nunca vira `values.X` — e não há erro: o placeholder sai vazio no arquivo gerado (caso real com `costcenter`/`project`, seção 14, erro 14.4).

```yaml
- id: fetch-terraform
  action: fetch:template
  input:
    url: ./skeleton/terraform
    targetPath: infra/${{ parameters.resourceName }}   # onde cai no repo de destino
    values:
      resourceName: ${{ parameters.resourceName }}     # esquerda: nome em values; direita: contexto do template.yaml
      owner: ${{ parameters.owner }}
```

Fluxo: formulário → `parameters` no `template.yaml` → step copia o que interessa para `values` → Nunjucks renderiza o skeleton com `values` como único contexto. Ex.: para `resourceName = "relatorios"`, a linha `name = "${{ values.resourceName }}"` vira `name = "relatorios"` no arquivo publicado — não sobra nenhum `${{ }}`.

#### `copyWithoutTemplating` (não `copyWithoutRender`)

GitHub Actions também usa `${{ }}` (`${{ github.sha }}`, `${{ secrets.TOKEN }}`). Sem exclusão, o Nunjucks tenta interpretar essas expressões — e quebra, por exemplo, em `${{ inputs.x || 'plan' }}` (`Expected name, received "|"`: o `|` é lido como filtro Nunjucks). Por isso os workflows ficam num step separado:

```yaml
- id: fetch-workflow
  action: fetch:template
  input:
    url: ./skeleton/workflow
    copyWithoutTemplating:
      - .github/workflows/**
```

**Não usar `copyWithoutRender`**: no `@backstage/plugin-scaffolder-backend` instalado (4.0.3) esse campo é aceito pelo schema mas ignorado pelo handler (`templateActionHandler.ts`, `resolveTemplateActionSettings` só lê `ctx.input.copyWithoutTemplating`). Caso real na seção 14, erro 14.9.

#### Publicação via PR

```yaml
- id: publish
  action: publish:github:pull-request
  input:
    repoUrl: ${{ parameters.repoURL }}
    branchName: scaffolder/${{ parameters.resourceName }}
    title: 'infra: novo bucket S3 - ${{ parameters.resourceName }}'
    description: |
      PR gerado automaticamente pelo Backstage.
      Revise o `terraform plan` comentado pelo CI antes de aprovar.
```

- Cria branch `scaffolder/<resourceName>` a partir da branch padrão → o repo **precisa ter ao menos um commit** ("Git Repository is empty" caso contrário).
- Outputs disponíveis: `targetBranchName`, `remoteUrl`, `pullRequestNumber`. **Não produz `repoContentsUrl`** (só `publish:github` produz) — foi isso que quebrou o antigo step `catalog:register`.
- `output.links` mostra só "Ver Pull Request" (`steps.publish.output.remoteUrl`). Não há link "Ver no catálogo": a entidade não existe ainda nesse momento.

### 7.2 `infra-repo-bootstrap`

Fonte: `infra-cloud/infra-repo-bootstrap/template.yaml`.

**Por que existe:** os templates de recurso exigem repositório já existente **e com pelo menos um commit**. Este template cria o repositório de infra de um time já com commit inicial, uma vez por time.

```yaml
# infra-cloud/infra-repo-bootstrap/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: infra-repo-bootstrap
  title: Bootstrap de Repositório de Infra (por time)
  description: Cria o repositório de infraestrutura como código de um time, pronto para receber PRs dos templates de recurso (S3, VPC, ...)
  tags: [infra, bootstrap, github]
spec:
  owner: group:default/plataforma
  type: infrastructure

  parameters:
    - title: Dados do repositório
      required: [repoUrl, owner]
      properties:
        owner:
          title: Team owner repository
          type: string
          ui:field: OwnerPicker
          ui:options:
            catalogFilter:
              kind: Group
        description:
          title: Descrição curta (o que esse repositório vai hospedar)
          type: string
        repoUrl:
          title: Novo repositório de infra
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts: [github.com]
            allowedOwners: [acme]

  steps:
    - id: fetch-base
      name: Gerar README, catalog-info.yaml e .gitignore
      action: fetch:template
      input:
        url: ./skeleton
        values:
          repoName: ${{ (parameters.repoUrl | parseRepoUrl).repo }}
          owner: ${{ parameters.owner }}
          description: ${{ parameters.description }}

    - id: publish
      name: Criar repositório no GitHub
      action: publish:github
      input:
        repoUrl: ${{ parameters.repoUrl }}
        description: ${{ parameters.description }}
        defaultBranch: 'main'

  output:
    links:
      - title: Repositório
        url: ${{ steps.publish.output.remoteUrl }}
```

| Campo | Widget | Função |
|---|---|---|
| `owner` | `OwnerPicker` (`kind: Group`) | Time dono do **repositório** (não confundir com o dono de cada recurso criado depois) |
| `description` | texto livre (opcional) | Vai pra criação do repo e pro `catalog-info.yaml` |
| `repoUrl` | `RepoUrlPicker` (`allowedOwners: [acme]`) | Onde o **novo** repositório será criado — ele **não existe ainda** |

`parseRepoUrl` é filtro Nunjucks nativo do Scaffolder que decompõe `github.com?owner=acme&repo=infra-aws` em `{ host, owner, repo }` — evita pedir o nome do repo duas vezes. `publish:github` cria o repositório do zero e empurra direto na `main`, sem PR (não faz sentido revisar um README + skeleton inicial). É o único ponto na org que exercita a permissão **Administration (de repositório)** na App (seção 4.4).

**Skeleton gerado** — `README.md` (explica a convenção `infra/<recurso>/`), `.gitignore` (`.terraform/`, `*.tfstate`, `*.tfplan`) e:

```yaml
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: ${{ values.repoName }}
  description: ${{ values.description }}
spec:
  owner: ${{ values.owner }}
```

`System`, não `Component`/`Resource`: o repositório **agrupa** vários recursos. Os `catalog-info.yaml` dos templates de recurso **não** apontam `spec.system` para este `System` hoje — melhoria futura (seção 16).

Sem `catalog:register`: como o commit vai direto na `main`, o discovery registra em até 1 min. Adicionar `register` duplicaria a entidade.

**Fluxo completo do zero:**

1. **Create** → `Bootstrap de Repositório de Infra (por time)` → `owner`, descrição, `repoUrl` (ex. `github.com?owner=acme&repo=infra-azure`).
2. Repositório criado com README + `catalog-info.yaml` + `.gitignore`.
3. Em até 1 min aparece no catálogo como `System`.
4. **Create** → template do recurso (ex. `S3 Bucket (AWS)`) → em "Repositório de destino", selecionar o repo recém-criado.
5. Configurar no repo novo as GitHub Actions **variables** `AWS_ROLE_ARN` e `AWS_REGION` e garantir que a trust policy da IAM role cobre `repo:acme/<nome-do-repo>:*` (seção 9) — passo manual hoje.

**Limitação:** o nome do repo é livre — `infra-aws`/`infra-azure` são só exemplo na descrição; não há `pattern`. Isso já causou mismatch com um glob `infra-*` na trust policy (seção 9.1). E nada garante que o `owner` do bootstrap seja o mesmo time que depois selecionará o repo nos templates de recurso (seção 6).

### 7.3 `aws-s3-bucket`

Fonte: `infra-cloud/aws/s3-bucket/template.yaml`.

```yaml
# infra-cloud/aws/s3-bucket/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: aws-s3-bucket
  title: S3 Bucket (AWS)
  description: Cria um bucket S3 com o módulo padrão da organização, via PR revisado
  tags: [aws, terraform, storage]
spec:
  owner: group:default/plataforma     # quem mantém o template, não o bucket
  type: resource

  parameters:
    - title: Dados do recurso
      required: [resourceName, owner, environment, costcenter, project, repoURL]
      properties:
        resourceName:
          title: Nome do bucket (sem prefixo)
          type: string
          pattern: '^[a-z0-9-]+$'
        owner:
          title: Time do dono
          type: string
          ui:field: OwnerPicker
          ui:options:
            catalogFilter:
              kind: Group
        region:
          title: Região AWS
          type: string
          enum: [ca-central-1, us-east-1, sa-east-1]
        environment:
          title: Ambiente
          type: string
          enum: [dev, hml, prod]
        repoURL:
          title: Repositório de destino (já existente, do time dono)
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts: [github.com]
            allowedOwners: [acme]
        costcenter:
          title: Centro de custo do recurso
          type: string
          enum: [ti, financeiro, contabilidade]
        project:
          title: Projeto que usará esse storage
          type: string

  steps:
    - id: fetch-terraform
      name: Gerar arquivos terraform
      action: fetch:template
      input:
        url: ./skeleton/terraform
        targetPath: infra/${{ parameters.resourceName }}
        values:
          resourceName: ${{ parameters.resourceName }}
          region: ${{ parameters.region }}
          owner: ${{ parameters.owner }}
          environment: ${{ parameters.environment }}
          costcenter: ${{ parameters.costcenter }}
          project: ${{ parameters.project }}

    - id: fetch-workflow
      name: Copiar workflow do GitHub Actions
      action: fetch:template
      input:
        url: ./skeleton/workflow
        copyWithoutTemplating:
          - .github/workflows/**

    - id: publish
      name: Abrir Pull Request no repositório do time
      action: publish:github:pull-request
      input:
        repoUrl: ${{ parameters.repoURL }}
        branchName: scaffolder/${{ parameters.resourceName }}
        title: 'infra: novo bucket S3 - ${{ parameters.resourceName }}'
        description: |
          PR gerado automaticamente pelo Backstage.
          Revise o `terraform plan` comentado pelo CI antes de aprovar (seção 4.1).

  output:
    links:
      - title: Ver Pull Request
        url: ${{ steps.publish.output.remoteUrl }}
```

| Campo | Notas |
|---|---|
| `resourceName` | Nome do bucket (regras S3) |
| `owner` | Time dono do bucket, do catálogo |
| `region` | Passado a `values.region`; **não está em `required`** e nada no skeleton do S3 o consome hoje |
| `environment` | `[dev, hml, prod]` — o `aws-vpc` usa `prd`; inconsistência entre templates (seção 16) |
| `repoURL` | Repositório **já existente** do time |
| `costcenter` / `project` | → `tags.CostCenter` / `tags.Project` |

Note que **não há step `catalog:register`** — só `fetch-terraform`, `fetch-workflow` e `publish`.

**Skeleton (`skeleton/terraform/main.tf`, estado atual):**

```hcl
module "bucket" {
  source = "git::https://github.com/acme/terraform-modules.git//aws/s3-bucket?ref=aws-s3-bucket/v0.1.0"

  bucket = {
    name          = "${{ values.resourceName }}"
    force_destroy = true
  }

  versioning = {
    enable     = true          # ⚠️ módulo espera `enabled` — ver seção 16
    mfa_delete = false
  }

  encryption = {
    sse_algorithm     = "AES256"
    bucket_key_enable = true   # ⚠️ módulo espera `bucket_key_enabled` — ver seção 16
  }

  logging = {
    enable = false             # ⚠️ módulo espera `enabled` — ver seção 16
  }

  lifecycle_rules = []

  tags = {
    Environment        = "${{ values.environment }}"
    Owner              = "${{ values.owner }}"
    CostCenter         = "${{ values.costcenter }}"
    Project            = "${{ values.project }}"
    ManagedBy          = "backstage-scaffolder"
    DataClassification = "internal"
  }
}
```

Interface real do módulo (`terraform-modules/aws/s3-bucket/variables.tf`): `bucket { name, force_destroy }`, `versioning { enabled, mfa_delete }`, `encryption { sse_algorithm, kms_master_key_id?, bucket_key_enabled }`, `logging { enabled, target_bucket?, target_prefix? }`, `lifecycle_rules` (lista), `tags` (mapa com as 6 chaves obrigatórias `Environment`, `Owner`, `CostCenter`, `Project`, `ManagedBy`, `DataClassification`). `versioning`, `encryption`, `logging`, `lifecycle_rules` são defaults fixos no skeleton — o formulário não os coleta; pra torná-los configuráveis é preciso adicionar `parameters` e trocá-los por `${{ values.* }}`.

**`skeleton/terraform/catalog-info.yaml`:**

```yaml
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: ${{ values.resourceName }}
  description: Bucket S3 (${{ values.environment }}) provisionado via Backstage
spec:
  type: s3-bucket
  owner: ${{ values.owner }}
```

**Resultado no repositório de destino:**

```
<repo-do-time>/
├── .github/workflows/terraform-ci-cd.yml     ← raiz, compartilhado (fetch-workflow)
└── infra/<resourceName>/                       ← fetch-terraform
    ├── main.tf
    ├── backend.tf
    ├── provider.tf
    └── catalog-info.yaml
```

```mermaid
flowchart LR
    A[Usuário preenche formulário] --> B[fetch-terraform: renderiza skeleton/terraform]
    A --> B2[fetch-workflow: copia .github/workflows sem renderizar]
    B --> C[publish: cria branch + PR no GitHub]
    B2 --> C
    C --> D[Output: link para o PR]
    C -.depois do merge.-> E[discovery automático registra no catálogo]
```

Histórico de correções já aplicadas neste template (detalhes na seção 14): typo "S3 Bucke" no título; `parameters`/`steps`/`output` fora de `spec`; `CostCenter` vs `costcenter` (case); `costcenter`/`project` não repassados a `values`; `${{ parameters.* }}` usado dentro do skeleton; step `catalog:register` removido; `copyWithoutRender` → `copyWithoutTemplating`; `ref=v0.1.0` → `ref=aws-s3-bucket/v0.1.0`; `loggin` → `logging`; skeleton formatado com `terraform fmt`.

### 7.4 `aws-vpc`

Fonte: `infra-cloud/aws/vpc/template.yaml`. Criado a partir de uma cópia do `aws-s3-bucket`; os resquícios da cópia (descrição, título do PR, título de `project`, `region` fora de `required`, `enum` de `environment` incompatível com o módulo, CIDR único reaproveitado nas 4 sub-redes) já foram corrigidos.

```yaml
metadata:
  name: aws-vpc
  title: AWS VPC
  description: Cria uma VPC com o módulo padrão da organização, via PR revisado
  tags: [aws, vpc, terraform]
spec:
  owner: group:default/plataforma
  type: resource
```

**Parâmetros** — 12 campos, **todos** em `required`:

| Campo | Tipo/Widget | Notas |
|---|---|---|
| `resourceName` | string, `pattern: '^[a-z0-9-]+$'` | Nomeia o `module` e compõe os nomes das sub-redes (`<nome>-public-a`, …) |
| `vpcCidr` | string, regex de CIDR IPv4 | ex. `10.0.0.0/16` |
| `publicSubnetCidrA` / `publicSubnetCidrB` / `privateSubnetCidrA` / `privateSubnetCidrB` | string, mesmo regex | Um por sub-rede (substituíram o `subnetCidr` único em 2026-09-09). Escalares em vez de `type: array` pra evitar indexação por posição no `main.tf` |
| `owner` | `OwnerPicker` (`kind: Group`) | |
| `region` | `enum: [ca-central-1, us-east-1, sa-east-1]` | Obrigatório — usado sem fallback em `aws_region` e nas `availability_zone` |
| `environment` | `enum: [dev, hml, prd]` | Alinhado à `validation` do módulo |
| `repoURL` | `RepoUrlPicker` (`github.com`, `allowedOwners: [acme]`) | |
| `costcenter` | `enum: [ti, financeiro, contabilidade]` | |
| `project` | string | |

O regex de CIDR valida só **formato**; nem o JSON Schema nem a `validation` do módulo (que checa formato individual e unicidade de `name`) verificam **sobreposição** entre os 4 CIDRs ou se são sub-blocos de `vpcCidr` — isso só é pego pela AWS no `apply`.

**Steps:** iguais ao `aws-s3-bucket` (`fetch-terraform` com `targetPath: infra/${{ parameters.resourceName }}` e os 11 valores em `values`; `fetch-workflow` com `copyWithoutTemplating`; `publish` com título `infra: vpc - <resourceName>`).

**Skeleton (`skeleton/terraform/main.tf`, condensado):**

```hcl
module "networking_vpc" {
  source = "git::https://github.com/acme/terraform-modules.git//aws/vpc?ref=aws-vpc/v0.1.0"
  environment = "${{ values.environment }}"
  aws_region = "${{ values.region }}"

  vpc = {
    name = "${{ values.resourceName }}"
    cidr_block = "${{ values.vpcCidr}}"
    enable_dns_support = true
    enable_dns_hostnames = true
  }

  public_subnets = [
    { name = "${{ values.resourceName }}-public-a", cidr_block = "${{ values.publicSubnetCidrA }}", availability_zone = "${{ values.region}}a" },
    { name = "${{ values.resourceName }}-public-b", cidr_block = "${{ values.publicSubnetCidrB }}", availability_zone = "${{ values.region}}b" }
  ]

  private_subnets = [
    { name = "${{ values.resourceName }}-private-a", cidr_block = "${{ values.privateSubnetCidrA }}", availability_zone = "${{ values.region }}a" },
    { name = "${{ values.resourceName }}-private-b", cidr_block = "${{ values.privateSubnetCidrB }}", availability_zone = "${{ values.region }}b" }
  ]

  nat_gateway   = { strategy = "single", enabled = true }                              # baseline ADR-002, fixo
  flow_logs     = { enabled = true, traffic_type = "ALL", retention_in_days = 14 }     # baseline ADR-002, fixo
  vpc_endpoints = { enabled = true, gateway_services = ["s3", "dynamodb"] }            # baseline ADR-002, fixo

  tags = {
    Environment        = "${{ values.environment }}"
    Owner              = "${{ values.owner }}"
    CostCenter         = "${{ values.costcenter }}"
    Project            = "${{ values.project }}"
    ManagedBy          = "backstage-scaffolder-terraform"
    DataClassification = "internal"
  }
}
```

| Variável do módulo | Preenchida com | Origem |
|---|---|---|
| `environment`, `aws_region`, `vpc.name`, `vpc.cidr_block` | `values.*` | formulário |
| `vpc.enable_dns_*` | `true` fixo | hardcoded |
| `public_subnets[*]` / `private_subnets[*]` | `.name` composto de `resourceName`; `.cidr_block` de cada `values.*SubnetCidr*`; `.availability_zone` = `region` + `a`/`b` | formulário + composição; não há parâmetro de AZ |
| `nat_gateway`, `flow_logs`, `vpc_endpoints` | valores fixos | baseline `ADR-002` (mesmos de `terraform-modules/aws/vpc/examples/basic/main.tf`) — são `nullable = false` sem `default`, omiti-los quebrava o `plan` |
| `tags.*` | `values.*` + `ManagedBy`/`DataClassification` fixos | `tags.Environment` deve ser idêntico a `environment` (validation do módulo) |

**Known issue:** `nat_gateway.strategy` fica `"single"` mesmo pra `hml`/`prd` — o módulo suporta `"per_az"`, mas exigiria parâmetro novo ou lógica condicional (seção 16).

### 7.5 Organização do skeleton (`catalog-info.yaml`)

Depois de dividir o skeleton em `terraform/` e `workflow/`, avaliou-se se o `catalog-info.yaml` deveria ganhar pasta própria (`yamls/`). Cada subpasta é lida por um `fetch:template` separado, e é o `targetPath` de cada step — não a organização do skeleton — que decide onde o conteúdo cai.

**Opção A — dentro de `terraform/` (adotada):** convenção comum no Backstage: o `catalog-info.yaml` mora junto do código/IaC que descreve; os dois precisam cair no mesmo diretório de qualquer forma. Pra um único arquivo, um step extra não paga o custo.

**Opção B — `yamls/` irmã de `terraform/`:** exige terceiro step `fetch:template` com o **mesmo** `targetPath` do `fetch-terraform` (duplicado em dois lugares que precisam ficar sincronizados). Resultado final idêntico. **Variante não recomendada:** `yamls/` **dentro** de `terraform/` — gera `infra/<nome>/yamls/catalog-info.yaml`, pasta extra sem vantagem.

Revisitar se surgirem mais tipos de arquivo além de Terraform, workflow e catálogo.

### 7.6 O workflow `terraform-ci-cd.yml` gerado

O skeleton copia, na **raiz** do repositório de destino, um único workflow compartilhado por **todos** os recursos do time (gerado seguindo os padrões do subagente `gha-pipeline-specialist` do `terraform-modules`). A explicação linha a linha, os conceitos do GitHub Actions usados e as decisões estão em [`pipeline_terraform_scaffolder.md`](pipeline_terraform_scaffolder.md); aqui só o que importa pra integração com o Backstage.

Como cada recurso é um stack independente em `infra/<nome>/` (com `backend.tf` e state próprios), o workflow nunca tem diretório fixo:

| Evento | Job | Como acha o diretório |
|---|---|---|
| `pull_request` em `infra/**/*.tf` | `plan` | `git diff origin/<base>...HEAD` lista os `infra/<nome>/` alterados; `fmt -check`, `init`, `validate`, `plan` percorrem a lista com um `for` no mesmo job |
| `workflow_dispatch` | `apply` | Input `resource` (texto livre), validado por allowlist de caracteres e `test -d infra/$resource` antes de qualquer chamada à AWS. No `aws/vpc` ainda vale `find infra -mindepth 1 -maxdepth 1 -type d` (primeira subpasta) |

Decisões registradas:

- **Um job, um `for`, um comentário** — sem `strategy.matrix`. Todos os plans vão num único comentário no PR (marker `<!-- tf-plan -->`, atualizado a cada push), uma seção `#### infra/<nome>` por stack.
- **`fmt -check` reprova, não corrige** — os `.tf` do skeleton precisam estar `fmt`-limpos: se um repo gerado reprova sem ninguém ter editado, o defeito está no skeleton.
- **`apply` sem `type: choice`** — input `string` + validação (regex + `test -d`). O valor entra no shell via `env:`, nunca interpolado direto no `run:`.
- **`environment:` no job `apply`** — no `aws/s3-bucket` está **comentado** (`# environment: prd`, descomentar quando houver GitHub Environment com required reviewers); no `aws/vpc` está ativo (`environment: ${{ github.event.inputs.environment }}`). Sozinho, o YAML não bloqueia nada; quem bloqueia é a proteção do Environment (**Settings → Environments → Required reviewers**). Isso também muda o claim `sub` do token OIDC (seção 9.1).
- **Concurrency** — por branch no PR (push novo cancela o plan anterior) e por stack no apply (`terraform-infra/<resource>`, nunca cancelado).
- **Ações fixadas por SHA.**
- **Repos já gerados não recebem atualizações do workflow** — o arquivo só é recopiado na próxima execução do template naquele repo (e como o conteúdo é idêntico, o PR não mostra diff dele). Correção no skeleton exige regerar ou copiar à mão nos repos existentes.

**Atenção:** existe um `terraform-ci-cd.old` (versão anterior, versionado no git) ao lado do `.yml` em `aws/s3-bucket/skeleton/workflow/.github/workflows/`. Como o `fetch-workflow` copia a pasta `workflow/` inteira, esse arquivo **vai junto em todo PR gerado** pelo `aws-s3-bucket` (o GitHub ignora a extensão `.old` como workflow, mas é ruído no repositório do time). Ver seção 16.

### 7.7 Importando recursos AWS já existentes para o catálogo

Nem todo recurso de infraestrutura nasce pelo Backstage. KMS keys, buckets S3, bancos RDS e outros recursos criados manualmente, por outra pipeline ou antes da adoção do portal precisam estar no catálogo para serem referenciáveis — por exemplo, num `EntityPicker` de um template (seção 6) ou numa relação `dependsOn` de um `Component` (seção 12.2). Há duas abordagens, e a escolha depende do volume.

#### Opção A — Registro via `catalog-info.yaml` em repositório dedicado (recomendada para poucos recursos)

Criar um repositório na org (ex.: `acme/aws-resources-catalog`) com um `catalog-info.yaml` por recurso. O `acmeProvider` (seção 4.10) descobre tudo automaticamente pelo glob `**/catalog-info.yaml`.

Estrutura sugerida:

```
aws-resources-catalog/
├── catalog-info.yaml              # kind: System — agrupa os recursos importados
├── kms/
│   ├── kms-dados-dev.yaml         # kind: Resource, spec.type: kms-key
│   └── kms-dados-prd.yaml
└── rds/
    └── relatorios-prd.yaml        # kind: Resource, spec.type: rds-instance
```

Exemplo de arquivo por recurso:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: kms-dados-dev
  description: KMS key para encriptação de dados em dev
  annotations:
    aws.com/arn: "arn:aws:kms:us-east-1:123456789012:key/abc-123"
spec:
  type: kms-key
  owner: group:default/devops
  system: aws-resources
```

Com isso, um template de EC2 pode oferecer um dropdown de KMS keys existentes usando `EntityPicker`:

```yaml
# Trecho de parameters no template.yaml
kmsKey:
  title: KMS Key para encriptação
  type: string
  ui:field: EntityPicker
  ui:options:
    catalogFilter:
      kind: Resource
      spec.type: kms-key
```

**Vantagens:** zero código, zero credencial AWS no Backstage, funciona com o discovery que já está configurado. **Desvantagem:** manutenção manual — ao criar ou remover um recurso na AWS, alguém precisa atualizar o YAML correspondente.

#### Opção B — Entity provider via API da AWS (recomendada para muitos recursos)

O plugin `@backstage/plugin-catalog-backend-module-aws` consulta a API da AWS periodicamente e cria entidades `kind: Resource` automaticamente, com ARN, tags e região.

Configuração em `app-config.yaml`:

```yaml
catalog:
  providers:
    aws:
      account1:
        accountId: '123456789012'
        region: us-east-1
        schedule:
          frequency: { hours: 1 }
          timeout: { minutes: 3 }
```

Para serviços não cobertos pelo plugin (ex.: KMS), é possível escrever um **entity provider custom** — um módulo backend que implementa `EntityProvider`, chama o SDK da AWS (`KMS.ListKeys()` / `KMS.ListAliases()`) e emite entidades para o catálogo.

**Vantagens:** totalmente automático, sem risco de descompasso entre AWS e catálogo. **Desvantagens:** requer credenciais AWS configuradas no Backstage (IAM role ou access key com permissões de leitura), e código custom para serviços fora do plugin.

#### Qual escolher?

| | Opção A (YAML manual) | Opção B (Provider AWS) |
|---|---|---|
| **Esforço inicial** | Quase zero | Instalar plugin + configurar credenciais |
| **Atualização** | Manual (commit no repo) | Automática a cada schedule |
| **Poucos recursos (< 20)** | Ideal | Overkill |
| **Centenas de recursos** | Inviável | Ideal |
| **Credenciais AWS no Backstage** | Não | Sim |

Para a implantação atual, a Opção A é suficiente: criar o repositório `aws-resources-catalog`, adicionar os YAMLs dos recursos pré-existentes, e o discovery registra tudo em até 1 minuto. Se o volume crescer, migrar para a Opção B.

### 7.8 Testando templates localmente sem push na `main`

O discovery em produção lê templates da `main` via `type: url`. Para iterar no formulário de um template sem precisar fazer push, a estratégia é separar as locations entre os dois arquivos de config:

- **`app-config.yaml`** (dev, `yarn start`): `type: file` apontando para o path local do `infra-cloud`
- **`app-config.production.yaml`** (Docker): `type: url` apontando para o GitHub na `main`

```yaml
# app-config.yaml (dev) — locations locais
catalog:
  locations:
    - type: file
      target: ../../infra-cloud/aws/s3-bucket/template.yaml
      rules:
        - allow: [Template]
    - type: file
      target: ../../infra-cloud/aws/vpc/template.yaml
      rules:
        - allow: [Template]
    - type: file
      target: ../../infra-cloud/infra-repo-bootstrap/template.yaml
      rules:
        - allow: [Template]
```

```yaml
# app-config.production.yaml (Docker) — locations via GitHub
catalog:
  locations:
    - type: url
      target: https://github.com/acme/infra-cloud/blob/main/aws/s3-bucket/template.yaml
      rules:
        - allow: [Template]
    - type: url
      target: https://github.com/acme/infra-cloud/blob/main/aws/vpc/template.yaml
      rules:
        - allow: [Template]
    - type: url
      target: https://github.com/acme/infra-cloud/blob/main/infra-repo-bootstrap/template.yaml
      rules:
        - allow: [Template]
```

Funciona porque o Backstage substitui arrays inteiros no merge de config (seção 4.8): em produção, o array de `catalog.locations` do `app-config.production.yaml` sobrescreve completamente o do `app-config.yaml`. Em dev (`yarn start`), só o `app-config.yaml` é carregado, então valem os `type: file`.

O path relativo `../../infra-cloud/` resolve a partir de `packages/backend/` (diretório de trabalho do backend em dev). Cada vez que o `template.yaml` for salvo, o catálogo re-lê no próximo refresh cycle — não precisa reiniciar.

**Alternativas rápidas para testes pontuais:**

- **Register via UI** — clicar em "Register Existing..." e colar a URL do GitHub de uma feature branch (ex.: `https://github.com/acme/infra-cloud/blob/minha-feature/aws/novo/template.yaml`). Desregistrar manualmente ao terminar.
- **Template Editor** — acessar `/create/edit` no Backstage e colar o YAML. Renderiza o formulário sem registrar nada no catálogo (não executa os `steps`, só valida os `parameters`).

---

## 8. Passo a passo: expondo um módulo Terraform já existente como template

Cenário comum: o módulo (`terraform-modules/aws/<módulo>`) **já existe e já foi validado**, mas ainda não tem um `template.yaml` no `infra-cloud` — ou tem, mas não aparece em **Create**. Sequência completa, usando o `aws/vpc` como exemplo real.

**Pré-requisito**: o módulo já existe em `terraform-modules/aws/vpc/`, com `variables.tf`/`outputs.tf` prontos e `terraform validate` passando (skill `terraform-deploy` e subagente `aws-architect` do `terraform-modules`).

**Passo 0 — publicar a tag do módulo, se ainda não existir** (`terraform-modules`)

Todo `main.tf` de skeleton referencia o módulo por `ref` fixo (seção 1.3) — a tag precisa existir *antes*:

```bash
cd terraform-modules
git tag aws-vpc/v0.1.0
git push origin aws-vpc/v0.1.0
```

(A skill `git-terraform` automatiza add + commit + push + tag.)

**Passo 1 — criar a estrutura do template** (`infra-cloud`)

```
infra-cloud/aws/vpc/
├── template.yaml
└── skeleton/
    ├── terraform/
    │   ├── main.tf              # chama o módulo via ref fixo (Passo 0)
    │   ├── backend.tf           # state remoto compartilhado — key com prefixo do tipo (seção 5.3)
    │   ├── provider.tf
    │   └── catalog-info.yaml    # entidade Resource, descoberta pelo acmeProvider
    └── workflow/
        └── .github/workflows/terraform-ci-cd.yml
```

Não crie do zero: copie a estrutura de um template já existente (seção 7.3) e ajuste. Nenhum desses arquivos é compilado — o `infra-cloud` é lido em runtime, direto do GitHub.

**Passo 2 — mapear os parâmetros do formulário para as variáveis do módulo**

Cada variável **obrigatória** do módulo (`nullable = false` e sem `default`) precisa terminar preenchida no `main.tf` do skeleton — vinda de um `parameters` novo (propagado via `values:` no `fetch-terraform`), ou com valor fixo. Se uma variável obrigatória não aparecer em nenhum dos dois, `terraform plan` falha em **toda** VPC gerada — foi o que aconteceu com `nat_gateway`/`flow_logs`/`vpc_endpoints` (seção 7.4). Conferir também os **nomes dos atributos** dos `object(...)` contra o `variables.tf` — o skeleton do S3 tem hoje `enable` onde o módulo espera `enabled` (seção 16).

- [ ] Toda variável `nullable = false` sem `default` tem valor no `main.tf` do skeleton.
- [ ] Todo `parameters.X` usado no skeleton está listado em `values:` — um esquecido renderiza vazio, silenciosamente.
- [ ] Atributos dos objetos batem com o `variables.tf` na tag referenciada.
- [ ] `.tf` passam em `terraform fmt -check`.

**Passo 3 — commit e push do `infra-cloud`**

```bash
cd infra-cloud
git add aws/vpc/
git commit -m "feat: adiciona template aws-vpc"
git push
```

Sem build nem restart — o Scaffolder só vai ler este `template.yaml` quando a location do Passo 4 existir.

**Passo 4 — registrar a location, nos DOIS arquivos de config** (`infra-portal`)

```yaml
# infra-portal/app-config.yaml E infra-portal/app-config.production.yaml
catalog:
  locations:
    - type: url
      target: https://github.com/acme/infra-cloud/blob/main/aws/vpc/template.yaml
      rules:
        - allow: [Template]
```

**Precisa existir nos dois arquivos, sem exceção** (seção 4.8). Esquecer de replicar já aconteceu duas vezes na org.

**Passo 5 — aplicar a mudança de config**

Uma nova `location` é config, lida na inicialização, e `app-config*.yaml` é copiado para dentro da imagem no build — `docker compose restart` **não** basta:

```bash
cd infra-portal
docker compose build backstage
docker compose up -d backstage
```

**Passo 6 — validar**

```bash
docker logs backstage | grep -i "aws-vpc"
curl -s http://localhost:7007/api/catalog/entities/by-name/template/default/aws-vpc | jq .   # 200, não 404
```

Depois, `http://localhost:7007` → **Create** → o card "AWS VPC" deve aparecer. Se não, ou se o log mostrar `Policy check failed`/`Malformed envelope`, ver seção 14 (erros 14.1–14.3).

```
0. (só se o módulo ainda não tiver tag) git tag + push no terraform-modules
1. Criar template.yaml + skeleton/ no infra-cloud
2. Mapear parameters ↔ variables obrigatórias do módulo (values: no fetch-terraform)
3. git add + commit + push no infra-cloud (sem build)
4. Adicionar a location em app-config.yaml E app-config.production.yaml (infra-portal)
5. docker compose build backstage && docker compose up -d backstage
6. Validar: docker logs + curl na API do catálogo + conferir "Create" na UI
```

---

## 9. Pipeline Terraform nos repositórios gerados

O que acontece **depois** do Backstage: o PR aberto pelo template dispara o workflow da seção 7.6 no repositório do time. Detalhes do workflow em [`pipeline_terraform_scaffolder.md`](pipeline_terraform_scaffolder.md); aqui, a configuração externa (AWS, GitHub) que ele exige e o troubleshooting real.

### 9.1 OIDC com a AWS: trust policy da role

Role usada: `backstage-terraform-apply`, com `AmazonS3FullAccess`/`AmazonVPCFullAccess`. Configuração no repo: GitHub Actions **variables** `AWS_ROLE_ARN` e `AWS_REGION` (ex. `us-east-1`, baseline do ADR-002). Requer, uma vez por conta AWS: um Identity Provider OIDC para `token.actions.githubusercontent.com` e a IAM Role abaixo.

**Armadilha central:** as *permissions policies* da role (aba "Permissions") definem o que a role pode fazer **depois** de assumida. Quem decide se `AssumeRoleWithWebIdentity` é aceita é a **trust policy** (aba "Trust relationships"). Erro `Not authorized to perform sts:AssumeRoleWithWebIdentity` → olhar a trust policy, nunca as permissions.

Trust policy final (depois de três causas encadeadas, seção 14, erro 14.14):

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com" },
  "Action": [
    "sts:AssumeRoleWithWebIdentity",
    "sts:TagSession"
  ],
  "Condition": {
    "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
    "StringLike":   { "token.actions.githubusercontent.com:sub": ["repo:acme/aws:*"] }
  }
}
```

- **Owner é `acme`**, não a conta pessoal. O claim `sub` de um repo da org é `repo:acme/<repo>:...`.
- **Nome exato do repositório**, não glob `infra-*`: o `infra-repo-bootstrap` não impõe prefixo. Novo repo de time = nova entrada no array `sub` (ou uma role central com `repo:acme/*`).
- **`:*` no fim** cobre os claims diferentes que o mesmo workflow gera: `plan` em `pull_request` → `repo:acme/aws:pull_request`; `apply` via `workflow_dispatch` → `repo:acme/aws:environment:<dev|hml|prd>` se o job declara `environment:` (caso do `aws/vpc`), ou `repo:acme/aws:ref:refs/heads/main` se não declara (caso do `aws/s3-bucket` hoje).
- **`sts:TagSession` é obrigatório**: `aws-actions/configure-aws-credentials` anexa role session tags por padrão (`Repository`, `Actor`, `Commit`, `Branch`, `EventName`, `RunId`, `TriggeringActor`…). Sem essa action a chamada inteira é negada com a **mesma mensagem genérica**, mesmo com `sub`/`aud`/owner/nome corretos. Decisão: manter o tagging (em vez de `role-skip-session-tagging: true`) — a role aplica infra real e as tags dão rastreabilidade no CloudTrail.
- **Case**: nomes de repo no GitHub são case-insensitive pra unicidade, mas o claim `sub` preserva o nome como cadastrado, e `StringLike` no IAM é case-sensitive. Recriar `AWS` → `aws` exigiu atualizar a trust policy **e** recriar as Repository variables (ficam amarradas ao ID interno do repo, não ao nome).
- **Debug**: `Re-run jobs` → **Enable debug logging** mostra claims e session tags no step `configure-aws-credentials`.
- PR vindo de **fork** restringe `id-token` por padrão e causa o mesmo erro independente da trust policy.

### 9.2 `terraform init` com módulo em repositório privado

`acme/terraform-modules` é **private**. O `terraform init` baixa módulos `git::https://...` chamando o `git` do runner, que clona **sem credencial**:

```
fatal: could not read Username for 'https://github.com': No such device or address
```

**Armadilha:** o `GITHUB_TOKEN` automático do job **não resolve** — é escopado ao repositório onde o workflow roda; não dá acesso a outros repos da org mesmo com `permissions: contents: read`. A tag `aws-s3-bucket/v0.1.0` **não** é o problema (o clone falha antes de resolver a `ref`).

| Opção | Prós | Contras |
|---|---|---|
| A. Tornar `terraform-modules` público | Zero mudança no workflow | Expõe o código da org |
| B1. Reusar o GitHub App do Backstage | App já existe | Excesso de privilégio; chave privada em dois lugares; rotacionar derruba o login do Backstage |
| B2. PAT fine-grained como secret | Rápido | Credencial de longa duração atrelada a uma pessoa |
| **B3. GitHub App novo, dedicado e só-leitura** | Privilégio mínimo; token de instalação de 1h gerado no job; rotação independente; audit log separado | Um app a mais para manter |

**Decisão: B3.** Nunca reaproveitar o app do Backstage.

**Implementação (pendente, 2026-09-14):**

1. Criar o App `acme-tf-modules-reader` (sugestão): **Repository → Contents: Read-only**, nada mais; instalado só em `acme/terraform-modules`; webhook desativado. Guardar `App ID` como **variable** de org `TF_MODULES_APP_ID` e a chave `.pem` como **secret** de org `TF_MODULES_APP_PRIVATE_KEY`, com "Selected repositories" restrito aos repos de infra gerados pelo Backstage.
2. Adicionar antes do `Terraform init`, nos jobs `plan` e `apply` (primeiro em `aws/s3-bucket/skeleton/workflow/...`, depois `aws/vpc`; obter o SHA da release corrente de `actions/create-github-app-token`):

   ```yaml
         - name: Token para clonar terraform-modules (repo privado)
           id: app-token
           uses: actions/create-github-app-token@<sha>  # vX.Y.Z
           with:
             app-id: ${{ vars.TF_MODULES_APP_ID }}
             private-key: ${{ secrets.TF_MODULES_APP_PRIVATE_KEY }}
             owner: acme
             repositories: terraform-modules   # token escopado a este único repo

         - name: Autenticar git para módulos privados
           run: |
             git config --global \
               url."https://x-access-token:${{ steps.app-token.outputs.token }}@github.com/".insteadOf \
               "https://github.com/"
   ```

   O `insteadOf` reescreve toda URL `https://github.com/...` que o `git` usar — inclusive as do `terraform init`. O token expira em 1h e o `create-github-app-token` o revoga no `post` step.
3. Verificação: reabrir/pushar o PR; o `init` deve mostrar `- bucket in .terraform/modules/bucket/aws/s3-bucket` sem pedir credencial.

A versão genérica desse diagnóstico (independente do Backstage) está no manual de CI/CD, seção 14.7: `../cicd_github/manual-cicd-github-actions-terraform.md`.

### 9.3 Variáveis `AWS_REGION`/`AWS_ROLE_ARN` por repositório

Hoje são **Repository variables** configuradas manualmente, uma vez por repositório — trabalho repetido (e esquecível) a cada `infra-repo-bootstrap`.

- **Opção 1 — Variável de organização (recomendada, zero código):** `Organization acme → Settings → Secrets and variables → Actions → Variables → New organization variable`, "Repository access" = *All repositories*. Resolve `AWS_REGION` hoje. Só resolve `AWS_ROLE_ARN` se a role for central pra org (trust policy `repo:acme/*`).
- **Opção 2 — Scaffolder configura na criação:** custom scaffolder action no `infra-portal/packages/backend` chamando `POST /repos/{owner}/{repo}/actions/variables`, step novo nos templates com a região vinda do formulário, e permissão **"Actions: write"** na App. Só compensa se cada repo/recurso precisar de região ou role diferentes.

---

# Parte III — Operação

## 10. Plugins e integrações

Coluna "Status" reflete o que já está registrado em `packages/backend/src/index.ts`/`packages/app/src/App.tsx` hoje:

| Plugin | Status | Por que vale a pena |
|---|---|---|
| **Scaffolder** (`@backstage/plugin-scaffolder-backend` 4.0.3, `-module-github`, `-module-notifications`) | ✅ Instalado | Core do caso de uso — templates de Terraform (Parte II). |
| **TechDocs** (`@backstage/plugin-techdocs-backend`) | ✅ Instalado (builder local, seção 11) | Documentação viva de cada recurso/módulo. |
| **Catalog GitHub Org** (`@backstage/plugin-catalog-backend-module-github-org`) | ✅ Instalado e registrado | Sincroniza Teams/Users da org (seção 12). |
| **Catalog GitHub discovery** (`@backstage/plugin-catalog-backend-module-github`) | ✅ Instalado e registrado | Descobre `catalog-info.yaml` em toda a org (seção 4.10) — sem ele, nenhuma `Resource` provisionada pelo Scaffolder aparece no catálogo. |
| **Auth GitHub** (`@backstage/plugin-auth-backend-module-github-provider`) | ✅ Instalado e registrado | Login (seção 3.4). |
| **Notifications + Signals** | ✅ Instalado | Notificar o time quando um `apply` terminar ou um PR for aberto — ainda não configurado para esse fluxo. |
| **Permission framework** (`@backstage/plugin-permission-backend`) | ⚠️ Instalado com `-module-allow-all-policy` — `permission.enabled: true`, mas a policy libera tudo. | Controlar quem pode rodar quais templates (ex.: só `plataforma` cria VPC). Essencial num portal que cria infra real com custo — próximo passo natural. |
| **Kubernetes** (`@backstage/plugin-kubernetes-backend`) | ✅ Instalado, sem cluster | Gera o `warn` inofensivo da seção 3.3. |
| **Search** (`-backend`, `-module-pg`, `-module-catalog`, `-module-techdocs`) | ✅ Instalado | Importante à medida que o catálogo crescer. |
| **Home page** (widgets em `packages/app/src/modules/home/`) | ✅ Customizado (toolkit + world clock) | Expandir com "meus recursos"/"PRs de infra pendentes" no futuro. |
| **Cost Insights** (`@backstage-community/plugin-cost-insights`) | ❌ Não instalado | Custo de nuvem por Component/team. Requer backend de dados de custo (seção 10.2). |
| **GitHub Actions** (`@backstage-community/plugin-github-actions`) | ❌ Não instalado | Mostraria o status do `plan`/`apply` na página do recurso — fecharia o loop visual (seção 10.1). |

**Nota sobre "plugins dinâmicos":** o Backstage open source **não** carrega plugins em runtime — todo plugin novo exige rebuild (seção 3.7.2). Diferente do Red Hat Developer Hub, que tem *dynamic plugins*. Não é o caso do `infra-portal`.

### 10.1 Instalando o GitHub Actions (status de plan/apply na página do recurso)

Objetivo: mostrar o status das runs do GitHub Actions (o workflow `terraform-ci-cd.yml` gerado pelos templates do `infra-cloud`, seção 8) direto na página do `Resource` no catálogo.

Pacote `@backstage-community/plugin-github-actions` (v1.5.0 no momento desta escrita) — plugin só de **frontend**, sem backend próprio. Tem export `./alpha`, compatível com o novo frontend system que o `infra-portal` já usa (`packages/app/src/App.tsx`, `createApp`/`features`).

**Passo 1 — instalar o pacote**
```sh
cd infra-portal
yarn --cwd packages/app add @backstage-community/plugin-github-actions
```

**Passo 2 — registrar em `packages/app/src/App.tsx`** (mesmo padrão de `catalogPlugin`/`authPlugin` já existentes):
```tsx
import githubActionsPlugin from '@backstage-community/plugin-github-actions/alpha';

export default createApp({
  features: [catalogPlugin, authPlugin, githubActionsPlugin, navModule, homeModule, signInModule],
});
```

**Passo 3 — permissão na GitHub App** — o plugin consulta a REST API de Actions em nome do usuário logado, pelo mesmo `githubAuthApiRef` já usado no login (`packages/app/src/modules/signIn/SignInPage.tsx`, seção 3.4). A GitHub App configurada em `GITHUB_APP_*` precisa ter a permissão **"Actions: Read-only"** habilitada:
1. `https://github.com/settings/apps/<nome-da-app>/permissions` (conta dona da App) — marcar **Actions: Read-only** e salvar.
2. `https://github.com/organizations/acme/settings/installations` (configurações da **organização**, não da conta pessoal) — se aparecer um aviso de permissão pendente ("This app is requesting additional permissions"), **Review request** → **Accept new permissions**. Se dono da App e admin da org forem a mesma conta, essa aprovação pode já vir aplicada automaticamente ao salvar o passo 1.
3. **Logout/login de novo no Backstage** — o token OAuth do usuário já emitido não pega escopo novo sozinho.

Sem isso, a aba carrega vazia ou retorna `Error: Resource not accessible by integration` (403).

**Passo 4 — annotation `github.com/project-slug` nos skeletons** (`infra-cloud`) — hoje nenhum `catalog-info.yaml` tem bloco `annotations:`. É preciso:

1. Resolver `parseRepoUrl` **no `template.yaml`**, no bloco `values:` do step `fetch-terraform` (`aws/s3-bucket/template.yaml` e `aws/vpc/template.yaml`), e passar `repoOwner`/`repoName` já resolvidos pro skeleton — **não** passar a `repoURL` crua e chamar o filtro dentro do skeleton: isso falha com `Template filter "parseRepoUrl" failed: No matching integration configuration for host undefined`, porque a config de integrations usada pelo filtro só está disponível na renderização de nível do `template.yaml`, não na renderização interna dos arquivos do skeleton pelo `fetch:template`. Mesmo padrão já usado em `infra-repo-bootstrap/template.yaml:42`.
   ```yaml
   values:
     # ...existentes...
     repoOwner: ${{ (parameters.repoURL | parseRepoUrl).owner }}
     repoName: ${{ (parameters.repoURL | parseRepoUrl).repo }}
   ```
2. No `catalog-info.yaml` de cada skeleton (`aws/s3-bucket/skeleton/terraform/catalog-info.yaml`, `aws/vpc/skeleton/terraform/catalog-info.yaml`), referenciar só os valores já resolvidos, sem filtro:
   ```yaml
   metadata:
     name: ${{ values.resourceName }}
     annotations:
       github.com/project-slug: ${{ values.repoOwner }}/${{ values.repoName }}
   ```

**Importante:** recursos já provisionados antes dessa mudança **não** ganham a annotation retroativamente — só os PRs abertos depois do skeleton atualizado.

**Passo 5 — rebuild** (plugin novo é mudança de código, não de config — mesma regra de rebuild da seção 3.7.2). **Atenção**: `docker compose build backstage` sozinho **não** recompila nada — o `packages/backend/Dockerfile` só empacota o `bundle.tar.gz`/`skeleton.tar.gz` que já existem em `packages/backend/dist/` (está documentado no topo do próprio Dockerfile). Se você rodar `docker compose build` sem antes rodar `yarn build:backend`, a imagem sobe com o bundle **antigo** — plugin instalado no `package.json`/`App.tsx`, mas ausente do JS servido, e a aba simplesmente não aparece (sem erro nenhum, silencioso). Ordem correta:
```sh
cd infra-portal
yarn build:backend         # recompila packages/backend/dist/bundle.tar.gz (backend + frontend)
docker compose build backstage
docker compose up -d backstage
```
Depois de subir, se a aba ainda não aparecer, verifique se o navegador não está servindo o `index.html`/JS antigo em cache (hard refresh / aba anônima) antes de suspeitar de bug — o hash do `main.*.js` muda a cada build; conferir no DevTools → Network se bate com o hash do bundle novo.

**Passo 6 — validar** — abrir um `Resource` com a annotation `github.com/project-slug` e conferir se aparece a aba/seção de workflow runs do GitHub Actions.

**Vale a pena?** Na prática, o ganho é menor do que parece à primeira vista: como não há step `catalog:register` nos templates (seção acima), a entidade só existe no catálogo **depois do merge** — e o `plan` que importa pra decidir se aprova o PR já está comentado lá pelo CI (seção 4.1), antes de qualquer merge. A aba do GitHub Actions também só mostra **status** da run (sucesso/falha/link), não o conteúdo do log — pra ler o `plan`/`apply` de verdade, ainda é preciso abrir no GitHub. Ou seja: o plugin não ajuda na revisão do PR (que já funciona via comentário do CI); serve só como painel de saúde contínuo de recursos **já provisionados** (reruns, `apply` manual via `workflow_dispatch`, drift check). Avalie se esse ganho residual justifica a manutenção antes de replicar esse setup em outro template.

### 10.2 Instalando o Cost Insights (custo de nuvem por Component/team)

Diferente do GitHub Actions: o `@backstage-community/plugin-cost-insights` (v0.32.0, também com export `./alpha` compatível com o novo frontend system) **não vem com um backend oficial pronto**. É um plugin de frontend que define a interface `CostInsightsApi` — cabe a cada adopter implementá-la, ligada a uma fonte real de dados de custo. Não existe um `yarn add` que já deixa isso funcionando.

**Decisão de arquitetura antes de implementar** — já que o workspace é 100% AWS e os templates `aws/s3-bucket`/`aws/vpc` já geram as tags `CostCenter`/`Project` nos recursos (seção 7, linha ~1315), as opções de fonte de dados são:

1. **AWS Cost Explorer API** — consulta direta via SDK, agrupando por tag (`CostCenter`, `Project`) ou por conta. Mais simples de implementar; limites de granularidade/latência da API.
2. **Cost and Usage Report (CUR) + Athena** — CUR exportado pro S3, consultado via Athena/Glue. Mais trabalho de setup inicial; dados mais granulares e baratos de consultar em volume.

Recomendado abrir essa decisão como ADR via o subagente `aws-architect` (`terraform-modules/.claude/agents/`) antes de implementar — é escolha de arquitetura (fonte de dados, forma de autenticação do backend com a AWS), não só instalação de pacote.

**Passos de alto nível, depois de decidido:**

1. Adicionar o pacote no frontend: `yarn --cwd packages/app add @backstage-community/plugin-cost-insights`.
2. Escrever um backend plugin novo em `infra-portal/packages/backend` que implemente o contrato esperado (consultando Cost Explorer ou Athena, conforme a ADR) e registrá-lo em `packages/backend/src/index.ts`.
3. Implementar `CostInsightsApi` no frontend apontando pro endpoint desse backend novo, e registrar via `ApiBlueprint`/`createApiFactory` (novo frontend system) no `App.tsx` — junto com `costInsightsPlugin` (`/alpha`) no array `features`.
4. Mapear as tags `CostCenter`/`Project` dos recursos AWS pra `Component`/`System`/`Resource` do catálogo, pra que o custo apareça agrupado por entidade.
5. Rebuild (`yarn build:backend` / `docker compose build backstage`) e validar na página do recurso.

---

## 11. Documentação de recursos (TechDocs)

TechDocs usa **MkDocs**: cada entidade aponta para uma pasta com `docs/` + `mkdocs.yml`, renderizada dentro da entidade. Isso gera duas camadas de documentação distintas no fluxo do `infra-cloud`, que **não devem ser confundidas**:

1. **Docs do módulo** (técnica de construção + operacional de uso) — sobre o template em si (como o skeleton do `aws/s3-bucket` foi feito, quais inputs o form pede, como debugar o PR gerado). É conteúdo estático, igual pra qualquer instância provisionada a partir daquele template — por isso pode (e deve) ser **consolidado num único MkDocs dentro do `infra-cloud`**, cobrindo todos os módulos (`s3-bucket`, `vpc`, e os que forem adicionados).
2. **Docs da instância provisionada** (owner real, ARN, custo estimado daquele recurso específico, runbook daquela instância) — é conteúdo que varia por instância e só existe depois que o Terraform é aplicado. Como o `techdocs-ref` do Backstage é resolvido **por entidade**, relativo a onde o `catalog-info.yaml` daquela entidade foi registrado, esse conteúdo tem que continuar sendo gerado pelo `skeleton/` e publicado no repo de destino do PR — nunca pode ficar centralizado no `infra-cloud`, senão todas as instâncias daquele tipo mostrariam a mesma página estática.

### 11.1 Docs do módulo — consolidada em `infra-cloud`

Implementado para `s3-bucket` e `vpc` (2026-10-11), gerado/atualizado pela skill `document-module` (`backstage-org/.claude/skills/document-module/`) sempre que alguém pedir "documente o módulo X" — ela lê o módulo Terraform real, o template/skeleton e os ADRs relevantes antes de escrever. Estrutura:

```
infra-cloud/
├── mkdocs.yml              # único, nav agrupada por módulo
├── catalog-info.yaml       # registra esse MkDocs como entidade Component
├── docs/
│   ├── index.md
│   ├── s3-bucket/
│   │   ├── documentacao-tecnica.md   # como o módulo foi feito (decisões, terraform, skeleton)
│   │   └── guia-do-usuario.md        # como usar (inputs do form, o que é gerado, troubleshooting)
│   ├── vpc/
│   │   ├── documentacao-tecnica.md
│   │   └── guia-do-usuario.md
│   └── <recurso-futuro>/
│       ├── documentacao-tecnica.md
│       └── guia-do-usuario.md
├── aws/
│   ├── s3-bucket/template.yaml
│   └── vpc/template.yaml
└── infra-repo-bootstrap/template.yaml
```

`mkdocs.yml` na raiz do `infra-cloud` (nav agrupada por módulo, cada um com as duas páginas):

```yaml
site_name: infra-cloud
nav:
  - Início: index.md
  - S3 Bucket:
      - Documentação Técnica: s3-bucket/documentacao-tecnica.md
      - Guia do Usuário: s3-bucket/guia-do-usuario.md
  - VPC:
      - Documentação Técnica: vpc/documentacao-tecnica.md
      - Guia do Usuário: vpc/guia-do-usuario.md
plugins:
  - techdocs-core
markdown_extensions:
  - admonition
  - pymdownx.details
  - pymdownx.superfences
  - pymdownx.caret
  - pymdownx.tilde
  - attr_list
  - def_list
  - tables
```

`plugins: [techdocs-core]` é obrigatório — é o plugin que o `techdocs-cli`/gerador do Backstage injeta no build; sem ele o MkDocs builda normal via CLI solto, mas o TechDocs não reconhece a saída como compatível.

`markdown_extensions` com `admonition` é o que permite usar blocos `!!! info/note/tip/warning/question "título"` no conteúdo — a skill `document-module` usa esses blocos em vez de parágrafo denso ou tabela de 1 coluna (ex. metadados de última atualização, avisos de comportamento por ambiente, cada item do Troubleshooting). Sem essa extensão, o Markdown renderiza como texto cru, sem a caixa de destaque.

**Armadilha:** o `nav` referencia `docs/index.md` — crie esse arquivo (compartilhado por todos os módulos, não um por módulo) antes do primeiro build. Sem ele, o `mkdocs build` roda sem erro nenhum, mas a raiz do site TechDocs fica com `index.html` ausente (404 silencioso na aba Docs) — já aconteceu nesse workspace.

`catalog-info.yaml` na raiz do `infra-cloud`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: infra-cloud-docs
  annotations:
    backstage.io/techdocs-ref: dir:.
spec:
  type: documentation
  lifecycle: production
  owner: <group>
```

Esse `catalog-info.yaml` é descoberto automaticamente pelo `acmeProvider` (discovery dinâmico, seção 2) — **não** precisa editar `catalog.locations` nos `app-config*.yaml`. Os `template.yaml` continuam sendo a entidade `Template` usada pelo "Create" e não precisam de `techdocs-ref` próprio; opcionalmente, dá pra linkar a seção correspondente desse doc consolidado via `metadata.links` em cada `template.yaml`, pra quem está criando o recurso achar fácil o link da doc operacional daquele módulo.

### 11.1.1 Docs da instância provisionada — por repo de destino

Estrutura dentro do `skeleton/` de cada template, renderizada no repo onde o PR é aberto:

```
infra-cloud/aws/s3-bucket/skeleton/
├── terraform/
│   └── catalog-info.yaml   # precisa de techdocs-ref: dir:.
├── workflow/
└── docs/
    ├── index.md            # o que é, quando usar
    ├── arquitetura.md      # diagrama, decisões
    ├── runbook.md          # como debugar/recriar/destruir
    └── custo-estimado.md   # ordem de grandeza de custo
```

`catalog-info.yaml` precisa da annotation `backstage.io/techdocs-ref: dir:.`, resolvida relativa ao repo de destino onde essa instância foi registrada.

### 11.2 Modo de build: local (atual) vs. CI (recomendado para produção)

```yaml
# infra-portal/app-config.yaml (hoje)
techdocs:
  builder: 'local' # o próprio backend roda mkdocs build sob demanda
  generator:
    runIn: 'local' # ver armadilha "docker" abaixo — não funciona com backend containerizado
  publisher:
    type: 'local'
```

- **Builder local** (atual): simples, mas não escala e o build acontece no mesmo processo que serve a UI.
- **Generator `local`** (atual, desde 2026-10-11): o backend roda `mkdocs build` diretamente no próprio processo/container, sem container-irmão. Exige `mkdocs-techdocs-core` instalado na imagem do backend:
  ```dockerfile
  # packages/backend/Dockerfile, antes de USER node
  RUN apt-get install -y --no-install-recommends python3 python3-pip && \
      pip install --break-system-packages --no-cache-dir mkdocs-techdocs-core
  ```
  `mkdocs-techdocs-core` já traz `mkdocs` + `mkdocs-material` + `pymdown-extensions` como dependências — não precisa instalar cada um separado.

  **Armadilha — `generator.runIn: 'docker'` (default do `create-app`, não usar aqui):** esse modo faz o backend pedir pro **Docker do host** rodar um container-irmão com o `mkdocs`, repassando um bind mount do tipo `/tmp/backstage-XXXXX:/input`. Isso só funciona se o backend roda direto no host. Como o `infra-portal` roda o backend **dentro de um container** (`docker-compose.yml`), esse `/tmp/backstage-XXXXX` só existe no container `backstage`, não no host — então o Docker do host monta um diretório vazio e o build falha com `Config file 'mkdocs.yml' does not exist`, mesmo com o `mkdocs.yml` existindo e correto. Dá pra contornar montando o socket do Docker do host (`/var/run/docker.sock`) **e** compartilhando um diretório real do host como `/tmp` dos dois containers, mas isso é mais hack em cima de hack (e monta o socket do Docker no container, dando a ele controle efetivo de root na máquina host) — `runIn: 'local'` é mais simples e não tem esse problema.
- **Publisher (CI + storage externo)** — recomendado ao sair do uso local: `mkdocs build` no CI do repositório de origem, resultado publicado num bucket S3; o backend só **lê**:
  ```yaml
  techdocs:
    builder: "external"
    publisher:
      type: "awsS3"
      awsS3:
        bucketName: ${TECHDOCS_S3_BUCKET}
        region: us-east-1
  ```
  O mesmo pipeline que roda `terraform plan` pode rodar `techdocs-cli generate && techdocs-cli publish`.

### 11.3 Convenção de conteúdo mínimo por recurso

"Contrato" mínimo que todo template deve gerar: **o que é**, **como recriar/destruir**, **quem é o owner**, **custo esperado**. É convenção de time, não configuração de ferramenta.

---

## 12. Usuários e Grupos: donos e consumidores de recursos

### 12.1 Não crie Users/Groups manualmente — sincronize do GitHub

A estrutura (convidar membros, criar Teams, hierarquia) é feita no GitHub (seção 4.2). Com o `githubOrg` provider, o catálogo importa automaticamente **GitHub Teams → `Group`** e **membros → `User`** (com `memberOf`). Se algum time não existir como GitHub Team, crie o Team lá — não crie `Group` direto no Backstage.

### 12.2 Modelando "dono" vs "quem usa" um recurso

**Dono do recurso** — responsável, paga a conta, decide o ciclo de vida: `spec.owner`, sempre um `Group`. É o que o skeleton do `aws-s3-bucket` gera:

```yaml
# infra-cloud/aws/s3-bucket/skeleton/terraform/catalog-info.yaml, renderizado
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: <resourceName>
  description: Bucket S3 (<environment>) provisionado via Backstage
spec:
  type: s3-bucket
  owner: <owner>
```

**Quem consome/usa** — outros Components que dependem dele: `spec.dependsOn` no consumidor (cria a relação inversa `dependencyOf` automaticamente):

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: api-relatorios
spec:
  type: service
  owner: group:default/time-relatorios
  system: relatorios
  dependsOn:
    - resource:default/s3-bucket-relatorios-financeiro
```

Na UI, a página do bucket mostra "Owned by \<time\>" e "Depended on by api-relatorios".

### 12.3 Hierarquia de grupos

Modele departamento → time com `spec.parent`/`spec.children` nos `Group`, espelhando nested teams do GitHub — importado automaticamente pelo `githubOrg`.

### 12.4 Troubleshooting: o `OwnerPicker` só mostra "guest"

Sintoma: ao rodar um template, o campo `owner` só lista um grupo `guest`.

**Causa.** O `create-app` registra uma location de exemplo (`examples/org.yaml`, seção 4.7) que cria `User: guest` e `Group: guests` fictícios. Se só aparece "guest", nenhum grupo real foi sincronizado ainda. Causas possíveis: módulo `github-org` não registrado (seção 3.4), bundle desatualizado (seção 3.7.2), ou App não instalado na org (seção 4.11, item 2).

**Como confirmar que funcionou.** `http://localhost:7007/catalog?filters%5Bkind%5D=group` deve listar os times do GitHub.

**Para tirar o "guest" de vez.** Depois que os grupos reais sincronizarem, remova a location de `examples/org.yaml` — **nos dois arquivos** `app-config*.yaml` (seção 4.8).

---

## 13. O que reiniciar a cada alteração / hot-reload

| O que mudou | Precisa reiniciar/rebuildar? | Detalhe |
|---|---|---|
| `app-config.yaml` / `app-config.production.yaml` | **Sim — rebuild da imagem + up** | Config é lida só na inicialização, e os arquivos são copiados pra dentro da imagem no build (seção 3.7) — `restart` não basta. Em dev (`yarn start`), o CLI reinicia sozinho ao salvar. |
| Variáveis de ambiente (`.env`) | **Sim — `docker compose up -d`** (sem rebuild) | Envs são lidas na subida do processo. |
| Permissão nova no GitHub App | **Sim — `docker compose restart`** (sem rebuild) | Descarta o installation token cacheado (seção 4.5). |
| Novo plugin / código de `packages/app` ou `packages/backend` | **Sim — `yarn build:backend` + rebuild + up** | `docker compose build` sozinho **não recompila nada** (seção 3.7.2). Não existe carregamento dinâmico de plugin. |
| `catalog-info.yaml` em qualquer repositório da org | **Não** | O `acmeProvider` faz polling (até 1 min). Forçar via `POST /api/catalog/refresh`. |
| Novos membros/times na org (`githubOrg`) | **Não** | Segue o schedule (30s em dev, 30 min em prod). Forçar com o `UPDATE` no Postgres (seção 4.11, item 3). |
| Novo template no `infra-cloud` | **Não** — mas a nova entrada em `catalog.locations` é config, que exige rebuild | Um `template.yaml` **novo** só é lido depois de uma location apontar pra ele. Mudanças **dentro** de um template/skeleton já registrado são só mais uma entidade a reprocessar, sem restart. |
| Conteúdo do TechDocs (`docs/*.md`) | **Não** | Rebuild/republish independentes do backend. |
| Migração de banco (upgrade do Backstage) | **Sim — restart com a imagem nova** | Migrations rodam na subida; acompanhar `docker logs backstage`. |

**Regra prática:** tudo que é **dado do catálogo** (entidades, membros, docs) é dinâmico. Tudo que é **configuração do processo**, **nova location** ou **código do app** exige rebuild/restart.

```bash
curl -X POST http://localhost:7007/api/catalog/refresh \
  -H "Content-Type: application/json" \
  -d '{"entityRef": "resource:default/s3-bucket-relatorios-financeiro"}'
```

---

## 14. Troubleshooting: catálogo de erros

Tabela-índice (sintoma → causa → correção), com detalhes abaixo. Erros de login/sincronização estão na seção 4.11; `OwnerPicker` só com "guest" na seção 12.4; erros de boot na seção 3.7.

| # | Sintoma | Onde | Causa | Correção |
|---|---|---|---|---|
| 14.1 | Template não aparece em "Create", sem erro nos logs (só em Docker) | portal | `catalog.locations` só em `app-config.yaml`; array substituído por `app-config.production.yaml` | Replicar a location nos dois arquivos + rebuild |
| 14.2 | `Malformed envelope, <root> must NOT have additional properties` | portal (logs) | `parameters`/`steps`/`output` fora de `spec` no `template.yaml` | Aninhar em `spec` |
| 14.3 | `NotAllowedError: ... is not of an allowed kind for that location` | portal (UI "Register Existing Component") | `catalog.rules` global exclui `Template` | Location estática com `rules: [allow: Template]` |
| 14.4 | Tags `CostCenter`/`Project` vazias no `.tf` gerado, sem erro | scaffolder | Mismatch de case (`CostCenter`/`costcenter`); parâmetros não repassados a `values`; `${{ parameters.* }}` dentro do skeleton | Alinhar nomes; passar tudo em `values`; só `values.*` no skeleton |
| 14.5 | `403 Forbidden` em `GET .../commits/main/status` | scaffolder (fetch) | App sem "Commit statuses" | Permissão + 3 passos (seção 4.5) |
| 14.6 | `Git Repository is empty` | scaffolder (publish PR) | Repo de destino sem commit | Repo com ≥1 commit (→ `infra-repo-bootstrap`) |
| 14.7 | `Resource not accessible by integration` em `git/blobs` | scaffolder (publish PR) | "Contents" em Read-only | "Contents: Read and write" + 3 passos |
| 14.8 | `Resource not accessible by integration` em `git/trees` (só após incluir workflow) | scaffolder (publish PR) | Falta "Workflows" pra escrever em `.github/workflows/` | "Workflows: Read and write" + 3 passos |
| 14.9 | `Template error at line N: Expected name, received "|"` | scaffolder (fetch workflow) | `copyWithoutRender` ignorado; Nunjucks renderizou `||` do Actions | `copyWithoutTemplating` |
| 14.10 | `InputError: Invalid input passed to action catalog:register` | scaffolder (register) | `publish:github:pull-request` não produz `repoContentsUrl`; e registrar antes do merge é errado | Remover o step; usar discovery |
| 14.11 | `Failed to create the Organization repository acme/aws, Resource not accessible by integration` | scaffolder (`publish:github`) | Falta "Administration" **de repositório** (não de organização) | Repository permissions → Administration: Read and write + 3 passos |
| 14.12 | Nenhuma linha `github-provider:acmeProvider:refresh` nos logs; discovery não registra | portal (Docker) | `dist/bundle.tar.gz` mais antigo que `src/index.ts` | `yarn tsc && yarn build:backend` antes de `docker compose build` (seção 3.7.2) |
| 14.13 | `Policy check failed for component:default/${{ values.name \| dump }}` (warn) | portal (logs) | Discovery acha `catalog-info.yaml` não renderizado nos skeletons | Inofensivo (seção 4.10) |
| 14.14 | `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Actions (`plan`) | Trust policy: owner pessoal em vez de `acme`; glob `infra-*` ≠ nome real; falta `sts:TagSession` | Corrigir `sub`; `Action` como array com `sts:TagSession` (seção 9.1) |
| 14.15 | `could not read Username for 'https://github.com'` no `terraform init` | Actions (`plan`) | `terraform-modules` privado; clone anônimo; `GITHUB_TOKEN` não alcança outros repos | GitHub App dedicado só-leitura + `create-github-app-token` (seção 9.2, pendente) |
| 14.16 | AWS rejeita no `apply` por CIDRs duplicados na VPC | Actions (`apply`) | 4 sub-redes com o mesmo `subnetCidr` | 4 parâmetros escalares de CIDR (feito 2026-09-09) |
| 14.17 | `terraform plan` falha por variável obrigatória sem valor (VPC) | Actions | `nat_gateway`/`flow_logs`/`vpc_endpoints` ausentes (`nullable = false`, sem `default`) | Adicionados ao skeleton com baseline ADR-002 |

### 14.1 – 14.3 Template não aparece em "Create" (2026-08-26; recorrência 2026-09-09)

**14.1** — Causa, mecanismo e regra na seção 4.8. A recorrência de 2026-09-09 (`aws-vpc` adicionado só em `app-config.yaml`, ainda como mudança local não commitada) confirmou que conferir a location nos dois arquivos é passo obrigatório de qualquer PR que adicione template.

**14.2** — Log:

```
Policy check failed for template:default/aws-s3-bucket;
caused by Error: Malformed envelope, <root> must NOT have additional properties
```

A entidade era lida do GitHub mas rejeitada na validação de política antes de ser registrada. Corrigido no `infra-cloud` (commit `3955fa5`).

**14.3** — Seção 4.9. Uma primeira hipótese (extensão `.yml` vs `.yaml`) foi corrigida de passagem (`template.yml` → `template.yaml`) mas não era a causa raiz.

Verificação final: `GET /api/catalog/entities/by-name/template/default/<nome>` deve retornar `200`.

### 14.4 – 14.10 Falhas sucessivas ao executar o `aws-s3-bucket` (2026-08-27)

Primeira execução de ponta a ponta. Encontrados e corrigidos, em sequência:

**14.4 — Campos do formulário não chegavam ao `main.tf`.** Três bugs encadeados em `costcenter`: (1) `required` listava `costcenter` mas a property era `CostCenter`; (2) o step `fetch` não repassava `costcenter`/`project` a `values`; (3) typo `costcenterenter` e tentativa de usar `${{ parameters.* }}` dentro do `main.tf` (no skeleton só existe `values`). Sintoma: nenhum erro — as tags saíam vazias.

**14.5 — 403 ao buscar o skeleton:**

```
Error: Request failed for https://api.github.com/repos/acme/software-templates/commits/main/status?per_page=0, 403 Forbidden
    at GithubUrlReader.fetchResponse ...
    at async GithubUrlReader.getRepoDetails ...
```

(Log histórico — o repo se chamava `software-templates`; renomeado para `infra-cloud` em 2026-09-02.) Faltava "Commit statuses: Read-only". Corrigir a permissão sozinho não bastou — foi aqui que se descobriu o procedimento de 3 passos (seção 4.5).

**14.6 — `Pull request creation failed; caused by HttpError: Git Repository is empty.`** `publish:github:pull-request` ramifica a partir de um commit existente na branch padrão. "Repositório já existente" garante que o repo existe, não que tem conteúdo. Motivou o `infra-repo-bootstrap`.

**14.7 — `Resource not accessible by integration` (`rest/git/blobs#create-a-blob`).** Escrever arquivos via Git Data API exige "Contents: Read and write"; o App estava em Read-only.

**14.8 — `Resource not accessible by integration` (`rest/git/trees#create-a-tree`)**, só depois de o skeleton ser dividido em `terraform/` + `workflow/` e o `fetch-workflow` incluir um arquivo em `.github/workflows/`. O GitHub trata esse caminho como caso especial: exige a permissão **"Workflows"** além de "Contents".

**14.9 — `Template error at line 44, column 75: Expected name, received "|"`** ao renderizar `terraform-ci-cd.yml`: `${{ github.event.inputs.environment || 'plan' }}` interpretado pelo Nunjucks (`|` = filtro). O step já usava `copyWithoutRender: [.github/workflows/**]` — campo aceito pelo schema mas ignorado pelo handler em `@backstage/plugin-scaffolder-backend` 4.0.3. Trocado por `copyWithoutTemplating` no `aws/s3-bucket` **e** no `aws/vpc`.

**14.10 — `InputError: Invalid input passed to action catalog:register, instance is not any of [subschema 0],[subschema 1]`.** O step usava `repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}`, mas `publish:github:pull-request` só produz `targetBranchName`, `remoteUrl`, `pullRequestNumber` (inspecionado em `node_modules/@backstage/plugin-scaffolder-backend-module-github/dist/actions/githubPullRequest.cjs.js`). Além do problema de sintaxe, registrar antes do merge contradiz a proposta do template. Step `register` e link "Ver no catálogo" removidos; registro via discovery (seção 5.2).

### 14.11 `publish:github` não cria repositório (2026-09-02)

Primeira execução do `infra-repo-bootstrap`:

```
Error: Failed to create the Organization repository acme/aws, Resource not accessible by integration
```

Permissões da App na hora (via `GET /app`, seção 4.6): `checks: read, contents: write, members: read, metadata: read, pull_requests: write, statuses: read, workflows: write`.

Primeira tentativa (incompleta): concedida `organization_administration: write` — **não resolveu**, mesmo com restart. Causa real: existem **duas** permissões chamadas "Administration" (Repository vs. Organization). `POST /orgs/{org}/repos` está listado sob **Repository permissions → Administration** — coerente com `repository_selection: all` na instalação. Confirmado: `"administration" in permissions == False`. Corrigido concedendo **Repository → Administration: Read and write** + 3 passos. Nunca tinha bloqueado antes porque nenhum template criava repositório.

### 14.12 – 14.13 Módulo novo não aparece rodando no container (2026-09-02)

Detalhes, diagnóstico e correção na seção 3.7.2. O efeito colateral 14.13 (skeletons batendo no glob) está na seção 4.10.

### 14.14 OIDC negado no pipeline (2026-09-10)

Primeira execução do job `plan` no repositório `acme/AWS`:

```
Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

Três causas encadeadas na trust policy (seção 9.1 tem o resultado final):

1. `sub` com `repo:usuario-pessoal/infra-*:*` (conta pessoal) e uma segunda entrada com sintaxe inválida (`usuario-pessoal@*/infra-*@*`). Owner trocado para `acme`.
2. Glob `infra-*` não batia com o nome real `AWS` (o bootstrap não impõe prefixo). Trocado pelo nome exato.
3. Repo recriado como `aws` (minúsculo); trust policy e Repository variables refeitas. Erro persistiu. Debug logging expôs:

   ```text
   ##[debug]3 role session tags are being used:
   ##[debug][{"Key":"GitHub","Value":"Actions"},{"Key":"Repository","Value":"acme/aws"},
   {"Key":"Workflow","Value":"Terraform"},{"Key":"Action","Value":"__actions_configure-aws-credentials"},
   {"Key":"Actor","Value":"portal-infra..."},{"Key":"Commit","Value":"..."},
   {"Key":"Branch","Value":"refs/pull/1/merge"},{"Key":"EventName","Value":"pull_request"},
   {"Key":"BaseRef","Value":"main"},{"Key":"HeadRef","Value":"scaffolder/..."},
   {"Key":"RunId","Value":"..."},{"Key":"Job","Value":"plan"},{"Key":"TriggeringActor","Value":"usuario-pessoal"}]
   Assuming role with OIDC
   Retry AssumeRole: attempt 1 of 12 failed: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity.
   ```

   Faltava `sts:TagSession` no `Action`.

### 14.15 `terraform init` sem credencial (2026-09-14)

```
Initializing modules...
  Downloading git::https://github.com/acme/terraform-modules.git?ref=aws-s3-bucket/v0.1.0 for bucket...
  │ Error: Failed to download module
  │ ... /usr/bin/git exited with 128: Cloning into '.terraform/modules/bucket'...
  │ fatal: could not read Username for 'https://github.com': No such device or address
```

Falha depois de o OIDC já funcionar. Causa, alternativas e solução decidida (não implementada) na seção 9.2.

### 14.16 – 14.17 Skeleton do `aws-vpc` × módulo (2026-09-09)

Detalhes na seção 7.4. Ambos corrigidos; a limitação que resta (sem validação de sobreposição de CIDR no formulário) é estrutural.

---

## 15. Checklists

### Novo template (ou template movido/renomeado)

- [ ] `template.yaml` no schema `v1beta3`: `parameters`/`steps`/`output` dentro de `spec`.
- [ ] Location `type: url` com `rules: [allow: Template]` em **`app-config.yaml`** **e** **`app-config.production.yaml`**.
- [ ] Rebuild da imagem (config é copiada no build; restart não basta).
- [ ] Se o repo do template for privado, App instalada com leitura de conteúdo.
- [ ] Logs do backend sem `Policy check failed`/`Malformed envelope` pro nome do template.
- [ ] `GET /api/catalog/entities/by-name/template/default/<nome>` → `200`.
- [ ] Comentários de path internos e links na documentação atualizados.
- [ ] `key` do `backend.tf` com prefixo do tipo de recurso (seção 5.3).

### Template que abre PR (`publish:github:pull-request`)

- [ ] Todo `values.*` usado no skeleton está explicitamente em `input.values` do step `fetch` — não há fallback de `parameters` para `values`.
- [ ] Nomes de `property` batem (case-sensitive) com `required`.
- [ ] Só `${{ values.* }}` dentro do skeleton, nunca `${{ parameters.* }}`.
- [ ] **Sem** step `catalog:register` — o registro é via discovery após o merge.
- [ ] Repo de destino com ≥1 commit na branch padrão.
- [ ] Arquivos em `.github/workflows/` num step separado com `copyWithoutTemplating` (não `copyWithoutRender`), e App com permissão "Workflows".
- [ ] `.tf` do skeleton `terraform fmt`-limpos (o CI reprova no `fmt -check`).
- [ ] Nomes de variáveis e atributos de objeto do `main.tf` conferidos contra o `variables.tf` do módulo na tag referenciada.

### Nova permissão no GitHub App

- [ ] Salvar no App (conferir se o nome existe em **Repository** e **Organization** permissions — escolher a que o endpoint exige).
- [ ] Aceitar a atualização pendente em `github.com/organizations/acme/settings/installations`.
- [ ] `docker compose restart backstage` (descarta o installation token cacheado).
- [ ] Em dúvida, `GET /app` com JWT da App pra ver o que está concedido.

### Alteração em `packages/backend/src/index.ts`

- [ ] `yarn tsc && yarn build:backend` **antes** de `docker compose build`.
- [ ] `grep` do pacote no bundle novo.
- [ ] `docker logs backstage | grep "Registered scheduled task"` mostra a task esperada.

### Trust policy de role usada via OIDC

- [ ] Owner real do repo (`acme`), nome exato (case-sensitive no IAM).
- [ ] `Action` inclui `sts:AssumeRoleWithWebIdentity` **e** `sts:TagSession`.
- [ ] `sub` termina em `:*` (cobre `pull_request`, `environment:<x>`, `ref:refs/heads/main`).
- [ ] Olhar **Trust relationships**, não **Permissions**.
- [ ] PR não vem de fork.
- [ ] Ao recriar repo: refazer Repository variables (`AWS_ROLE_ARN`, `AWS_REGION`).
- [ ] Persistindo: `Re-run jobs` → Enable debug logging.

### Antes de confiar no sync de Users/Groups

- [ ] Todo mundo aceitou o convite da org.
- [ ] Teams existem com o slug que vai em `spec.owner`.
- [ ] App instalada na org (não só criada).
- [ ] `http://localhost:7007/catalog?filters%5Bkind%5D=group` lista os times.

---

## 16. Pendências e próximos passos

A instalação em si está concluída: app criado, GitHub App configurado, containers de pé, login validado, três templates em produção, módulos de backend e tela de login customizada no `infra-portal`. Concluído desde revisões anteriores: location do `aws/vpc` nos dois `app-config` (2026-09-09); tag `aws-vpc/v0.1.0` publicada e referenciada pelo skeleton.

| Item | Estado | Onde |
|---|---|---|
| `terraform init` falha por `terraform-modules` privado — GitHub App dedicado só-leitura + `create-github-app-token` nos skeletons de `aws/s3-bucket` e `aws/vpc` | **Decidido, não implementado** (2026-09-14) | seção 9.2 |
| Skeleton do `aws-s3-bucket` usa `versioning.enable`, `encryption.bucket_key_enable` e `logging.enable`, mas `terraform-modules/aws/s3-bucket/variables.tf` exige `enabled` / `bucket_key_enabled` / `enabled` → `terraform plan` vai falhar por atributo obrigatório ausente assim que o `init` passar | **Bug latente, detectado na consolidação (2026-09-14)**, ainda não exercitado porque o `init` falha antes | seção 7.3 |
| `terraform-ci-cd.old` versionado dentro de `aws/s3-bucket/skeleton/workflow/.github/workflows/` é copiado em todo PR gerado | **Detectado na consolidação (2026-09-14)** — mover pra fora do skeleton ou remover | seção 7.6 |
| `environment` do `aws-s3-bucket` é `[dev, hml, prod]`, do `aws-vpc` é `[dev, hml, prd]` — padronizar | Não tratado | seções 7.3 / 7.4 |
| `region` no `aws-s3-bucket` fora de `required` e não consumido pelo skeleton | Não tratado | seção 7.3 |
| Trocar a policy do Permission framework de `allow-all` para uma regra real (ex.: só `plataforma` roda o template de VPC) | Não implementado | seção 10 |
| `repoURL` como `EntityPicker` filtrado por `owner` (evitar PR no repo do time errado) | Não implementado; depende de ≥1 time bootstrapado | seção 6 |
| `catalog-info.yaml` dos recursos apontar `spec.system` para o `System` do repositório | Não implementado | seção 7.2 |
| `nat_gateway.strategy` fixo em `"single"` mesmo em `hml`/`prd` | Não implementado (parâmetro ou lógica por ambiente) | seção 7.4 |
| Validação de sobreposição entre os 4 CIDRs de sub-rede no formulário | Limitação estrutural do JSON Schema; só a AWS pega no `apply` | seção 7.4 |
| Lock do state remoto (`dynamodb_table` ou lock nativo S3 do Terraform ≥ 1.10) | Ponto em aberto | seção 5.3 |
| `AWS_REGION`/`AWS_ROLE_ARN` como variável de organização (ou custom action) | Recomendação: variável de org | seção 9.3 |
| `environment:` no job `apply` do `aws/s3-bucket` (required reviewers) | Comentado no skeleton; descomentar quando houver GitHub Environment | seção 7.6 |
| Adicionar `docs/` aos skeletons e migrar `techdocs.builder` para `external` + `awsS3` | Não implementado | seção 11 |
| Repetir o padrão S3/VPC para Azure (`infra-cloud/azure/<recurso>/` + `terraform-modules/azure/<recurso>/`) | Pasta `azure/` reservada, vazia | seção 5.4 |
| Ruído `Policy check failed` dos skeletons no discovery | Inofensivo; restringir por topic se incomodar | seção 4.10 |
| Duplicação de `catalog.locations` nos dois `app-config` — `app-config.templates.yaml` dedicado | Só se virar dor recorrente | seção 4.8 |
| `organization_administration: write` concedida à App sem necessidade | Decidir se mantém | seção 4.4 |
| Remover location de `examples/org.yaml` (grupo `guest`) dos dois `app-config` | Quando os grupos reais estiverem sincronizando | seção 12.4 |
| Repos já gerados não recebem atualização do workflow do skeleton | Copiar à mão ou regerar | seção 7.6 |
| Plugins `GitHub Actions` e `Cost Insights` | Não instalados — passo a passo de instalação documentado | seção 10.1 / 10.2 |
| Criar repo `aws-resources-catalog` para importar recursos AWS pré-existentes ao catálogo | Não implementado | seção 7.7 |

---

## 17. Referências

**Documentos relacionados neste repositório**

- [`pipeline_terraform_scaffolder.md`](pipeline_terraform_scaffolder.md) — o workflow `terraform-ci-cd.yml` linha a linha: triggers, jobs, conceitos do GitHub Actions, decisões, pré-requisitos e diferenças entre `aws/s3-bucket` e `aws/vpc`.
- `../cicd_github/manual-cicd-github-actions-terraform.md` — manual geral de CI/CD com Terraform.

**Arquivos-fonte (workspace `backstage-org`)**

- `infra-portal/app-config.yaml`, `infra-portal/app-config.production.yaml`
- `infra-portal/packages/backend/src/index.ts`, `infra-portal/packages/backend/Dockerfile`, `infra-portal/docker-compose.yml`
- `infra-portal/packages/app/src/App.tsx`, `infra-portal/packages/app/src/modules/signIn/`
- `infra-cloud/aws/s3-bucket/template.yaml` + `skeleton/{terraform,workflow}/`
- `infra-cloud/aws/vpc/template.yaml` + `skeleton/{terraform,workflow}/`
- `infra-cloud/infra-repo-bootstrap/template.yaml` + `skeleton/`
- `terraform-modules/aws/s3-bucket/variables.tf`, `terraform-modules/aws/vpc/variables.tf`
- `terraform-modules/.claude/agents/gha-pipeline-specialist.md`, `.claude/skills/{terraform-deploy,git-terraform}/`

**ADRs (design dos módulos Terraform — não incorporados)**

- `docs/adrs/ADR-001-modulo-terraform-s3-bucket.md`
- `docs/adrs/ADR-002-vpc-baseline-dev-us-east-1.md` (baseline hardcoded no skeleton do `aws-vpc`)

**Externas**

- [Backstage — TechDocs configuration](http://backstage.io/docs/features/techdocs/configuration)
- [Permissões exigidas por endpoint da API do GitHub (GitHub Apps)](https://docs.github.com/en/rest/overview/permissions-required-for-github-apps)
- [Nunjucks](https://mozilla.github.io/nunjucks/)
