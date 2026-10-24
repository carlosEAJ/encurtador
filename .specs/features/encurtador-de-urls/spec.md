# Especificação: Encurtador de URLs (V1)

## Problem Statement

É preciso de uma aplicação fullstack simples para encurtar URLs longas. O objetivo é gerar um código curto único para cada URL cadastrada, permitir o redirecionamento para a URL original e contabilizar a quantidade de cliques. Além disso, deve existir uma forma restrita de visualizar essas estatísticas.

## Goals

- [ ] Permitir que o usuário insira uma URL longa e obtenha um link encurtado.
- [ ] Gerar um `shortCode` único para cada URL cadastrada e persistí-lo.
- [ ] Redirecionar o acesso via `/{shortCode}` para a URL original com status `302 Found`.
- [ ] Incrementar o contador de cliques (`clickCount`) a cada redirecionamento bem-sucedido.
- [ ] Disponibilizar uma página de analytics isolada para listar todas as URLs cadastradas com seus códigos e cliques.
- [ ] Permitir que o usuário informe um `shortCode` customizado (opcional) na criação do link.
- [ ] Permitir definir uma data/hora de expiração (opcional) para o link encurtado.
- [ ] Aplicar limite de requisições (Rate Limit) aos endpoints da API.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Sistema de login/autenticação | Segurança por omissão. A página de analytics será de uso restrito por acesso direto à rota. |
| Página intersticial com anúncios (Google AdSense) | Previsto apenas para a V2 (RF05). |
| Painel administrativo público | Não haverá. |

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Geração do `shortCode` | Alfanumérico curto, gerado pelo backend (ex: Base62) | Padrão para encurtadores e garante unicidade. | y |
| Banco de dados | Suporte a H2 (desenvolvimento) ou PostgreSQL (ambiente mais próximo de produção) | Mencionado nos requisitos não-funcionais. | y |
| Status de redirecionamento | `302 Found` | Explicitamente definido em RF02. | y |
| Acesso à rota de analytics | Rota isolada no frontend, sem proteção por login | "Segurança por omissão" conforme especificação. | y |
| Incremento de cliques | Feito no momento do redirecionamento bem-sucedido | Explicitamente definido em RF03. | y |
| Customização do `shortCode` | Campo opcional `customCode` no `POST /api/urls`, alfanumérico de 3 a 10 caracteres; se omitido, o backend gera automaticamente. Conflito retorna `409 Conflict`. | Atende ao requisito trazido para a V1 mantendo compatibilidade com a geração automática. | y |
| Expiração de links | Campo opcional `expiresAt` (ISO-8601, data/hora futura). Link expirado responde `410 Gone` e não incrementa cliques. Ausência do campo = link permanente. | Atende ao requisito trazido para a V1 sem afetar links já existentes. | y |
| Limite de requisições (Rate Limit) | Limite padrão configurável via propriedades da aplicação (ex: 60 requisições/minuto por IP), aplicado a `POST /api/urls` e `GET /api/urls`. Excedente retorna `429 Too Many Requests` com header `Retry-After`. | Atende ao requisito trazido para a V1 sem limitar o redirecionamento público. | y |

**Open questions:** none - all resolved or logged above.

## User Stories

### P1: Encurtar URL

**User Story:** Como usuário, eu quero inserir uma URL longa em um campo simples e obter um link encurtado para copiar e compartilhar.

**Why P1:** É o fluxo principal da aplicação. Sem ele, não existe objeto a ser redirecionado.

**Acceptance Criteria:**

1. WHEN o usuário enviar uma URL válida no frontend e acionar o encurtamento THEN o sistema SHALL enviar uma requisição `POST /api/urls` com o corpo `{"url": "..."}`.
2. WHEN o backend receber uma requisição válida para encurtar uma URL THEN o sistema SHALL gerar um `shortCode` único, persistir a `UrlEntity` com `clickCount = 0` e `createdAt` preenchido automaticamente.
3. WHEN a URL for criada com sucesso THEN o sistema SHALL retornar `201 Created` com os dados da URL gerada (`id`, `originalUrl`, `shortCode`, `clickCount`, `createdAt`).
4. IF a URL enviada for inválida ou vazia THEN o sistema SHALL retornar um erro adequado (`400 Bad Request`).
5. WHEN o frontend receber a resposta de sucesso THEN o sistema SHALL exibir o link encurtado pronto para cópia.

**Independent Test:** Inserir uma URL válida na tela inicial, obter o link encurtado e validar que os dados foram salvos no backend.

### P1: Redirecionamento para URL original

**User Story:** Como usuário, eu quero acessar um link encurtado e ser redirecionado para a URL original.

**Why P1:** É a funcionalidade principal de um encurtador de URLs.

**Acceptance Criteria:**

1. WHEN for feita uma requisição `GET /{shortCode}` com um `shortCode` existente THEN o sistema SHALL buscar a `UrlEntity` correspondente no banco.
2. WHEN o `shortCode` for encontrado THEN o sistema SHALL retornar `302 Found` com o header `Location` apontando para `originalUrl`.
3. IF o `shortCode` não existir THEN o sistema SHALL retornar `404 Not Found`.
4. WHEN o redirecionamento for bem-sucedido THEN o sistema SHALL incrementar `clickCount` em +1 para aquela URL.

**Independent Test:** Acessar `http://localhost:8080/{shortCode}` válido e verificar o redirecionamento para a URL original, além do incremento do contador.

### P1: Página de Analytics Privada

**User Story:** Como alguém com acesso à rota isolada, eu quero visualizar todas as URLs encurtadas com seus códigos e quantidade de cliques.

**Why P1:** Permite acompanhar o uso dos links sem criar um painel administrativo público.

**Acceptance Criteria:**

1. WHEN for feita uma requisição `GET /api/urls` THEN o sistema SHALL retornar `200 OK` com um array JSON contendo todas as `UrlEntity` cadastradas.
2. The system SHALL exibir no frontend, em uma rota isolada, para cada item: `shortCode`, `originalUrl`, `clickCount` e `createdAt`.
3. The system SHALL listar as URLs ordenadas de forma coerente com o que for retornado pelo backend (sem necessidade de filtros complexos na V1).
4. The frontend SHALL acessar essa rota de forma isolada, sem expor links para ela na navegação principal.

**Independent Test:** Cadastrar ao menos duas URLs, acessar a rota de analytics isolada e conferir se ambas aparecem com seus respectivos `clickCount`.

### P1: Customização de `shortCode`

**User Story:** Como usuário, eu quero poder escolher o código do meu link encurtado para que ele seja memorizável e faça sentido para o meu contexto.

**Why P1:** Aumenta o valor percebido do produto, mantendo a geração automática como fallback.

**Acceptance Criteria:**

1. WHEN o usuário enviar `POST /api/urls` com o campo opcional `customCode` THEN o sistema SHALL validar que ele é alfanumérico, com 3 a 10 caracteres, e usá-lo como `shortCode`.
2. IF o `customCode` informado já estiver em uso THEN o sistema SHALL retornar `409 Conflict` com uma mensagem indicando que o código está indisponível.
3. IF o `customCode` informado violar o formato permitido THEN o sistema SHALL retornar `400 Bad Request`.
4. WHEN o `customCode` não for informado THEN o sistema SHALL gerar o `shortCode` automaticamente conforme o comportamento padrão.

**Independent Test:** Criar um link com `customCode` inédito, acessar `/{customCode}` e confirmar o redirecionamento; repetir com o mesmo código e verificar `409`.

### P1: Expiração de links

**User Story:** Como usuário, eu quero definir uma data/hora de expiração para que links temporários deixem de funcionar automaticamente.

**Why P1:** Evita que links sensíveis ou temporários permaneçam válidos indefinidamente.

**Acceptance Criteria:**

1. WHEN o usuário enviar `POST /api/urls` com o campo opcional `expiresAt` (data/hora futura em ISO-8601) THEN o sistema SHALL persistir esse valor na `UrlEntity`.
2. IF o `expiresAt` informado for inválido ou estiver no passado THEN o sistema SHALL retornar `400 Bad Request`.
3. WHEN for feito `GET /{shortCode}` de um link cujo `expiresAt` já passou THEN o sistema SHALL retornar `410 Gone` sem incrementar `clickCount`.
4. WHEN o `expiresAt` não for informado THEN o sistema SHALL tratar o link como permanente (nunca expira).

**Independent Test:** Criar um link com `expiresAt` no passado e verificar `400`; criar com expiração curta, aguardar o vencimento, acessar e confirmar `410` e que o contador não mudou.

### P1: Limite de requisições (Rate Limit)

**User Story:** Como operador do sistema, eu quero limitar a quantidade de requisições por cliente para proteger a API de abuso e sobrecarga.

**Why P1:** Protege a disponibilidade do serviço e evita criação massiva de links.

**Acceptance Criteria:**

1. WHEN um mesmo IP exceder o limite configurado de requisições em uma janela de tempo THEN o sistema SHALL retornar `429 Too Many Requests` com o header `Retry-After`.
2. WHEN o número de requisições estiver dentro do limite THEN o sistema SHALL processar a requisição normalmente.
3. The system SHALL aplicar o limite a `POST /api/urls` e `GET /api/urls`, sem limitar o redirecionamento público `GET /{shortCode}`.
4. The system SHALL permitir configurar o limite e a janela de tempo via propriedades da aplicação, sem alteração de código.

**Independent Test:** Disparar requisições acima do limite configurado e verificar `429` com `Retry-After`; confirmar que o redirecionamento continua funcionando e que, dentro da janela, as requisições são aceitas.

## Edge Cases

1. IF a URL já foi encurtada anteriormente THEN o sistema SHALL seguir o comportamento definido: criar um novo registro ou reutilizar conforme implementação, desde que o `shortCode` gerado seja único.
2. IF houver tentativa de gerar um `shortCode` que já exista (colisão) THEN o sistema SHALL gerar um novo `shortCode` até obter um único.
3. IF a requisição `POST /api/urls` vier com body inválido ou sem o campo `url` THEN o sistema SHALL retornar `400 Bad Request`.
4. IF for acessado um `shortCode` com caracteres inválidos ou inexistente THEN o sistema SHALL retornar `404 Not Found`.
5. IF a `originalUrl` não possuir um formato válido de URL THEN o sistema SHALL rejeitar a requisição com `400 Bad Request`.
6. IF o `customCode` solicitado já existir THEN o sistema SHALL retornar `409 Conflict`, sem sobrescrever o registro existente.
7. IF o `customCode` contiver caracteres não alfanuméricos ou tiver menos de 3 / mais de 10 caracteres THEN o sistema SHALL retornar `400 Bad Request`.
8. IF o `expiresAt` estiver no passado ou em formato inválido THEN o sistema SHALL retornar `400 Bad Request`.
9. IF um `shortCode` existente já estiver expirado THEN o sistema SHALL retornar `410 Gone`, sem redirecionar e sem incrementar `clickCount`.
10. IF o limite de requisições for excedido THEN o sistema SHALL retornar `429 Too Many Requests` com header `Retry-After`, sem processar a operação.

## Modelo de Dados (UrlEntity)

| Campo | Tipo | Regras |
| --- | --- | --- |
| `id` | `Long` | Primary Key, Auto-incremento |
| `originalUrl` | `String` (TEXT) | Obrigatória |
| `shortCode` | `String` | Obrigatória, Única, VARCHAR(10) |
| `clickCount` | `Long` | Padrão inicial = 0 |
| `createdAt` | `LocalDateTime` | Preenchido automaticamente na criação |
| `expiresAt` | `LocalDateTime` | Opcional; quando preenchido, define o fim da validade do link |

## Contrato de Endpoints (API REST)

| Método | Rota | Body | Response | Observação |
| --- | --- | --- | --- | --- |
| `POST` | `/api/urls` | `{"url": "https://...", "customCode": "opcional", "expiresAt": "opcional (ISO-8601)"}` | `201 Created` + objeto `UrlEntity` | Criação do link encurtado; `400` para dados inválidos, `409` para `customCode` em uso, `429` se exceder o rate limit |
| `GET` | `/{shortCode}` | - | `302 Found` + Header `Location: [URL Original]` | Redirecionamento e incremento de cliques; `404` se inexistente e `410` se expirado |
| `GET` | `/api/urls` | - | `200 OK` + Array JSON de `UrlEntity` | Listagem para página de analytics (restrita); `429` se exceder o rate limit |

## Arquitetura (V1)

- **Backend:** Java 17+, Spring Boot, Spring Data JPA. Camadas: `Controller`, `Service`, `Repository`, `Model/Entity`.
- **Frontend:** Angular (versão recente), estilização minimalista com Bootstrap ou Tailwind CSS.
- **Banco:** Relacional. H2 para desenvolvimento local, PostgreSQL como opção para outros ambientes.
- **Separação:** Projetos `backend/` e `frontend/` totalmente separados.
- **Containerização:** Opcional para desenvolvimento local e recomendada para execução integrada. Ver seção **Containerização (Docker)**.

## Containerização (Docker)

A aplicação SHALL poder ser executada de forma integrada via Docker Compose, sem depender de instalações locais de Java, Node ou PostgreSQL.

### Serviços

| Serviço | Imagem / Build | Função |
| --- | --- | --- |
| `db` | `postgres:16-alpine` | Banco PostgreSQL com volume persistente |
| `backend` | build multi-stage (`maven:3.9-eclipse-temurin-17` → `eclipse-temurin:17-jre`) | API Spring Boot em `:8080` |
| `frontend` | build multi-stage (`node:20-alpine` → `nginx:alpine`) | SPA Angular + proxy reverso em `:80` |

### Estrutura esperada

```text
docker-compose.yml
backend/Dockerfile
frontend/Dockerfile
frontend/nginx.conf
.env.example
```

### Requisitos

1. WHEN `docker compose up --build` for executado THEN o sistema SHALL subir `db`, `backend` e `frontend` na ordem correta, com healthcheck do PostgreSQL antes de iniciar o backend.
2. The system SHALL configurar o backend para usar PostgreSQL no perfil `docker` (via variáveis `SPRING_DATASOURCE_*`), mantendo H2 como padrão em desenvolvimento local.
3. The frontend nginx SHALL encaminhar `/api/*` e `/{shortCode}` para o backend, garantindo que criação, analytics e redirecionamento funcionem atrás do proxy.
4. The system SHALL persistir os dados do PostgreSQL em volume Docker nomeado.
5. The system SHALL expor as configurações sensíveis (credenciais do banco e parâmetros de rate limit) via variáveis de ambiente, sem hardcode na imagem.
6. The system SHALL manter um arquivo `.env.example` documentando as variáveis necessárias.
7. The rate limit SHALL usar armazenamento em memória por instância na V1; cenários com múltiplas réplicas exigirão Redis (fora do escopo da V1).

### Observações

- H2 permanece disponível para execução local sem Docker; o Compose usa PostgreSQL.
- A exposição das portas `8080` (backend) e `80` (frontend) é suficiente para o ambiente local. Em produção, apenas o frontend deve ficar exposto publicamente.
- O redirect precisa passar pelo proxy do nginx para preservar `/{shortCode}` na mesma origem do frontend.

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| --- | --- | --- | --- |
| URL-01 | P1: Encurtar URL | Implementing | Pending |
| URL-02 | P1: Redirecionamento para URL original | Implementing | Pending |
| URL-03 | P1: Redirecionamento para URL original (incremento de cliques) | Implementing | Pending |
| URL-04 | P1: Página de Analytics Privada | Implementing | Pending |
| URL-05 | P1: Validações e tratamento de erros (inválido/inexistente) | Implementing | Pending |
| URL-06 | P1: Customização de `shortCode` | Implementing | Pending |
| URL-07 | P1: Expiração de links | Implementing | Pending |
| URL-08 | P1: Limite de requisições (Rate Limit) | Implementing | Pending |
| URL-09 | Infra: Containerização (Docker Compose) | Implementing | Pending |

## Success Criteria

- [ ] Consegue-se encurtar uma URL válida pelo frontend e receber o link pronto para cópia.
- [ ] Ao acessar `/{shortCode}` válido, ocorre redirecionamento `302 Found` para `originalUrl`.
- [ ] A cada acesso bem-sucedido ao redirecionamento, `clickCount` é incrementado corretamente.
- [ ] A rota isolada de analytics exibe todas as URLs com `shortCode`, `originalUrl`, `clickCount` e `createdAt`.
- [ ] Códigos inexistentes retornam `404` e URLs inválidas retornam `400` conforme esperado.
- [ ] É possível criar um link com `customCode` inédito; códigos repetidos retornam `409` e formatos inválidos retornam `400`.
- [ ] Links com `expiresAt` deixam de redirecionar após o vencimento (`410 Gone`) sem incrementar cliques.
- [ ] Requisições acima do rate limit configurado retornam `429` com `Retry-After`, sem bloquear o redirecionamento público.
- [ ] A aplicação completa sobe com `docker compose up --build`, com dados persistidos no PostgreSQL e frontend acessível pelo nginx.
- [ ] Criação, analytics e redirecionamento funcionam através do proxy do frontend, sem erros de CORS.
