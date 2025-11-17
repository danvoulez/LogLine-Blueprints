LogLine Mini Suite — Rebrand Blueprint v1.0

Objetivo: consolidar o rebranding e a arquitetura dos 10 produtos mini em torno do novo mini (ex–minicore), com API/CLI/SDK unificados, TDLN Gate e governança determinística.

⸻

0) North Star
	•	mini = motor geral JSON✯Atomic para criptografia, execução de contratos/spans e orquestração LLM‑friendly.
	•	Produtos mini = ferramentas plugáveis que enriquecem projetos internos ou de clientes via api.logline.world.
	•	TDLN + LLM Council na frente de todas as entradas em linguagem natural; TDLN decide.
	•	LogLine App = The ultimate frontend: Chat + Ledger + Apps mini integrados.

⸻

1) Mapa de Produtos

1.1. mini (ex–minicore)
	•	O que é: runtime JSON✯Atomic (contratos, spans, prova, políticas) + criptografia (Ed25519/BLAKE3).
	•	O que faz: executa spans, aplica policies, assina/valida recibos, expõe API universal.
	•	Tagline: “Nosso computador na nuvem.”

1.2. miniPrompt
	•	Geração/transformação determinística de prompts e prompt-kernels (hash + versionamento + cookbook).

1.3. miniMemory
	•	Store semântico + embeddings + chaves de privacidade (local-first opcional) com grants por tenant/usuário.

1.4. miniWorkflows
	•	Orquestrador de passos declarativos (spans → steps → policies), com replays e DLQ.

1.5. miniTools
	•	Execução segura de tools (HTTP, DB, code runners) a partir de spans com capabilities e quotas.

1.6. miniContainer (deploy • wallet • vault)
	•	deploy: sobe artefatos/containers (Railway/Podman) a partir de specs em spans.
	•	wallet: custodiante de chaves (API/Ed25519) + assinaturas + KID/rotação.
	•	vault: segredos/artefatos com escopo (tenant/ambiente) e prova de acesso.

Observação: podemos expor como submódulos: mini-container.deploy, mini-container.wallet, mini-container.vault.

1.7. miniContratos
	•	Produto para pequenas empresas: registro de acordos fora do código, com templates JSON✯Atomic e Flows nativos.
	•	Volta à lógica da antiga tela New do minicontratos — agora 100% ledger-first e disponível via API.

1.8. TDLN Gate + LLM Council (obrigatório)
	•	Entrada NL → LLM Conselheiro local → (opcional LLM premium) → TDLN decide → span perfeito.

1.9. LogLine App
	•	Frontend unificado: Chat AI + Acesso ao Ledger + painel dos produtos mini.
	•	LLM Engine com DRI do TDLN (mensagens organizadas, forte capacidade de ação).

⸻

2) Naming & Pacotes
	•	Serviços (HTTP): mini-* (ex.: mini-workflows, mini-container), sob api.logline.world.
	•	NPM/TS: @logline/mini-* (core, prompt, memory, workflows, tools, container, contratos, sdk, cli).
	•	CLI: logline <produto>:<ação> (ex.: logline container:deploy).
	•	SDK: client.<produto>.<método>() (ex.: client.container.deploy.apply(...)).

⸻

3) API Universal (envelope padrão)

Endpoint: POST /v1/commands (ou recursos específicos por produto).

{
  "action": "<produto>.<ação>",
  "resource": "urn:<domínio>/<id>",
  "body": { /* payload canônico */ },
  "ctx": {"tenant": "vv", "env": "prd"}
}

ACK: ok | need_more | invalid + receipt (hash/assinatura) + trace_id.

⸻

4) Superfícies por Produto (API/CLI/SDK)

4.1 mini (core)
	•	API
	•	POST /v1/spans/execute — executa/valida span.
	•	POST /v1/policies/evaluate — aplica conjunto de policies.
	•	CLI logline span:execute --file span.json | logline policy:test --law laws.yaml
	•	SDK client.core.executeSpan(span) | client.core.evaluatePolicies(input)

4.2 miniPrompt
	•	API POST /v1/prompt/compile, POST /v1/prompt/run
	•	CLI logline prompt:run --kernel kernel.md --vars vars.json
	•	SDK client.prompt.compile(...), client.prompt.run(...)

4.3 miniMemory
	•	API POST /v1/memory/put|get|search|grant
	•	CLI logline memory:put --key k --file data.json
	•	SDK client.memory.put/get/search/grant

4.4 miniWorkflows
	•	API POST /v1/workflows/start, POST /v1/workflows/step, GET /v1/workflows/:id
	•	CLI logline wf:start --flow order-approval
	•	SDK client.workflows.start/step/get

4.5 miniTools
	•	API POST /v1/tools/invoke (tool registry + caps)
	•	CLI logline tools:invoke --tool http --url ...
	•	SDK client.tools.invoke(tool, args)

4.6 miniContainer
	•	deploy: POST /v1/container/deploy | CLI logline container:deploy --ref app@1.0 --target railway
	•	wallet: POST /v1/container/wallet/sign|verify | CLI logline wallet:sign --file payload.json
	•	vault: POST /v1/container/vault/put|get | CLI logline vault:put --name SECRET --from .env
	•	SDK client.container.deploy.apply(...), client.wallet.sign(...), client.vault.put/get(...)

4.7 miniContratos
	•	API POST /v1/contratos/register, POST /v1/contratos/flow, GET /v1/contratos/:id
	•	CLI logline contratos:new --template nda --party A --party B
	•	SDK client.contratos.register/flow/get

4.8 TDLN Gate
	•	API POST /v1/tdln/decide|assist|compile
	•	CLI logline tdln:decide --text "..." --prefer-premium
	•	SDK client.tdln.decide/assist/compile

⸻

5) Spans — Templates (exemplos)

5.1 Deploy (miniContainer.deploy)

{
  "action": "container.deploy.apply",
  "resource": "urn:app/web",
  "body": {
    "ref": "web@1.2.3",
    "target": {"provider": "railway", "project_id": "rw1", "service": "web"},
    "env": {"NODE_ENV": "production"}
  }
}

5.2 Contrato (miniContratos.register)

{
  "action": "contratos.register",
  "resource": "urn:agreement/nda/2025-0001",
  "body": {
    "template": "nda@v3",
    "parties": [{"name": "Empresa A"}, {"name": "Empresa B"}],
    "effective_date": "2025-11-17",
    "jurisdiction": "PT"
  }
}


⸻

6) Governança rápida
	•	Tenants/Ambientes: obrigatório; quotas + rate limits por ação.
	•	Assinaturas: ApiKey (transporte) + Ed25519 (recibos, onde aplicável).
	•	Observabilidade: métricas ok/need_more/invalid, P50/P95, throughput.
	•	TDLN decide: LLMs aconselham, nunca decidem.

⸻

7) Migração (ex–minicore → mini)
	•	Aliases de ação por 60 dias: core.* → mini.* (deprecado em X‑Deprecation: true).
	•	SDK exporta ambos namespaces na v1; remove core.* na v2.
	•	CLI: comandos antigos avisam e redirecionam.

minicontratos (antigo) → miniContratos (novo)
	•	Reaproveitar templates/ledger; padronizar endpoints.
	•	Reativar Flows como workflows declarativos.

⸻

8) Pricing & Packaging (visão)
	•	Starter: mini + 2 produtos (ex.: container.deploy + contratos) — limites baixos.
	•	Pro: mini + 5 produtos, TDLN Gate incluso (LLM local) — SSE/Webhooks prontos.
	•	Enterprise: todos os produtos, LLM Premium, SSO/SCIM, auditoria avançada.

⸻

9) Backlog de Prioridades (Futuro 3)
	1.	mini (core) pronto + v1/commands estável.
	2.	miniContainer.deploy, miniContratos (MVP mercado).
	3.	TDLN Gate (Mini/Max) com LLM local.
	4.	miniWorkflows + miniTools (capabilities) → habilitar automações.
	5.	miniPrompt + miniMemory → elevar o LLM Engine.
	6.	LogLine App (frontend unificado).

⸻

10) Roadmap v0 → v1 (4–6 semanas)
	•	S1: mini+API universal; container.deploy; contratos.register; CLI/SDK básicos.
	•	S2: TDLN Gate (Mini) + LLM local; Workflows (mínimo) + Tools (HTTP runner).
	•	S3: Vault/Wallet; Prompt/Memory; App shell; métricas & policies endurecidas.

⸻

11) Integração com LogLine App
	•	App consome o ledger e chama API mini; TDLN Gate inspecciona NL.
	•	Painéis por produto, histórico por trace_id, reexecução de spans e explain.

⸻

12) Anexos
	•	OpenAPI (universal + produtos), CLI help, SDK stubs, exemplos de spans e policies.


LogLine Mini Suite — API + CLI Stubs v1

Scope: OpenAPI 3.1 skeleton (consolidado por produto) + ajuda da CLI + exemplos de policies e spans. Segurança: ApiKey (transporte) + headers opcionais de recibo/assinatura.

⸻

0) OpenAPI — Header & Security (comum)

openapi: 3.1.0
info:
  title: LogLine Mini Suite — API v1
  version: 1.0.0
servers:
  - url: https://api.logline.world
components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: Authorization
  parameters:
    Tenant:
      in: header
      name: X-Tenant
      schema: { type: string }
    Env:
      in: header
      name: X-Env
      schema: { type: string, enum: [dev, stg, prd] }
security:
  - ApiKeyAuth: []


⸻

1) mini (core)

paths:
  /v1/spans/execute:
    post:
      tags: [mini]
      summary: Executa/valida um span JSON✯Atomic
      parameters: [ { $ref: '#/components/parameters/Tenant' }, { $ref: '#/components/parameters/Env' } ]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                span: { type: object }
      responses:
        '200': { description: OK }
  /v1/policies/evaluate:
    post:
      tags: [mini]
      summary: Avalia policies sobre um input
      responses:
        '200': { description: OK }


⸻

2) miniPrompt

paths:
  /v1/prompt/compile:
    post:
      tags: [miniPrompt]
      summary: Normaliza kernel e gera hash
      responses: { '200': { description: OK } }
  /v1/prompt/run:
    post:
      tags: [miniPrompt]
      summary: Executa kernel com variáveis
      responses: { '200': { description: OK } }


⸻

3) miniMemory

paths:
  /v1/memory/put:
    post: { tags: [miniMemory], summary: Armazena item, responses: { '200': { description: OK } } }
  /v1/memory/get:
    post: { tags: [miniMemory], summary: Recupera item, responses: { '200': { description: OK } } }
  /v1/memory/search:
    post: { tags: [miniMemory], summary: Busca semântica, responses: { '200': { description: OK } } }
  /v1/memory/grant:
    post: { tags: [miniMemory], summary: Concede acesso, responses: { '200': { description: OK } } }


⸻

4) miniWorkflows

paths:
  /v1/workflows/start:
    post: { tags: [miniWorkflows], summary: Inicia um flow, responses: { '200': { description: OK } } }
  /v1/workflows/step:
    post: { tags: [miniWorkflows], summary: Avança passo, responses: { '200': { description: OK } } }
  /v1/workflows/{id}:
    get:  { tags: [miniWorkflows], summary: Consulta estado, parameters: [ { in: path, name: id, required: true, schema: { type: string } } ], responses: { '200': { description: OK } } }


⸻

5) miniTools

paths:
  /v1/tools/invoke:
    post:
      tags: [miniTools]
      summary: Invoca tool registrada com capabilities
      responses: { '200': { description: OK } }


⸻

6) miniContainer (deploy • wallet • vault)

paths:
  /v1/container/deploy:
    post: { tags: [miniContainer], summary: Aplica deploy (Railway/Podman), responses: { '200': { description: OK } } }
  /v1/container/wallet/sign:
    post: { tags: [miniContainer], summary: Assina payload, responses: { '200': { description: OK } } }
  /v1/container/wallet/verify:
    post: { tags: [miniContainer], summary: Verifica assinatura, responses: { '200': { description: OK } } }
  /v1/container/vault/put:
    post: { tags: [miniContainer], summary: Armazena segredo/artefato, responses: { '200': { description: OK } } }
  /v1/container/vault/get:
    post: { tags: [miniContainer], summary: Recupera segredo/artefato, responses: { '200': { description: OK } } }


⸻

7) miniContratos

paths:
  /v1/contratos/register:
    post: { tags: [miniContratos], summary: Registra contrato (template + parties), responses: { '200': { description: OK } } }
  /v1/contratos/flow:
    post: { tags: [miniContratos], summary: Executa fluxo do contrato (assinatura, anexos, etc.), responses: { '200': { description: OK } } }
  /v1/contratos/{id}:
    get:  { tags: [miniContratos], summary: Consulta contrato, parameters: [ { in: path, name: id, required: true, schema: { type: string } } ], responses: { '200': { description: OK } } }


⸻

8) TDLN Gate + LLM Council

paths:
  /v1/tdln/decide:
    post: { tags: [tdln], summary: NL/span → decisão (ok/need_more/invalid), responses: { '200': { description: OK } } }
  /v1/tdln/assist:
    post: { tags: [tdln], summary: Apenas aconselhamento (LLM local/premium), responses: { '200': { description: OK } } }
  /v1/tdln/compile:
    post: { tags: [tdln], summary: Compila/valida span perfeito, responses: { '200': { description: OK } } }


⸻

9) CLI — Ajuda (consolidada)

logline login --apiKey <token>
logline whoami
logline status --resource <urn> [--watch]

# mini (core)
logline span:execute --file span.json
logline policy:test --law laws.yaml

# miniPrompt
logline prompt:compile --kernel kernel.md
logline prompt:run --kernel kernel.md --vars vars.json

# miniMemory
logline memory:put --key k --file data.json
logline memory:get --key k
logline memory:search --text "..." --topk 5
logline memory:grant --principal user:123 --key k --perm read

# miniWorkflows
logline wf:start --flow order-approval --input input.json
logline wf:step --id wf_123 --step review --input step.json
logline wf:get --id wf_123

# miniTools
logline tools:invoke --tool http --url https://... --method POST --data @body.json

# miniContainer (deploy • wallet • vault)
logline container:deploy --ref app@1.2.3 --target railway --proj rw1 --svc web
logline wallet:sign --file payload.json --kid default
logline wallet:verify --file payload.json --sig @sig.txt --kid default
logline vault:put --name SECRET --from .env
logline vault:get --name SECRET > .env

# miniContratos
logline contratos:new --template nda@v3 --party "Empresa A" --party "Empresa B"
logline contratos:flow --id nda-2025-0001 --action sign --actor user:123
logline contratos:get --id nda-2025-0001

# TDLN
logline tdln:assist  --text "criar cliente Joana…" --watch
logline tdln:decide  --text "refund do pedido 123 por fraude" --prefer-premium
logline tdln:compile --file span.json


⸻

10) Policies — Exemplos (YAML)

# allowlist por ação/tenant
allow:
  - tenant: vv
    actions: [ 'container.deploy.apply', 'contratos.register', 'tdln.decide' ]

# quotas e rate limits
limits:
  - tenant: vv
    env: prd
    action: 'tdln.decide'
    rate_per_minute: 300
    burst: 100

# imagens permitidas (deploy)
images:
  allow: [ 'docker.io/library/nginx:*', 'ghcr.io/vv/*' ]


⸻

11) Spans — Exemplos úteis

{ "action": "memory.search", "resource": "urn:mem/global", "body": { "text": "contrato NDA", "topk": 5 } }
{ "action": "tools.invoke", "resource": "urn:tool/http", "body": { "url": "https://example.com/hook", "method": "POST", "json": {"ping": true} } }
{ "action": "workflows.start", "resource": "urn:flow/order-approval", "body": { "input": {"order_id": "123"} } }
{ "action": "container.wallet.sign", "resource": "urn:wallet/default", "body": { "kid": "default", "payload": {"hello": "world"} } }


⸻

12) Próximos passos
	•	Congelar os esquemas por produto, gerar OpenAPI completo e SDK TS.
	•	Implementar CLI conforme ajuda acima (com --json, --no-stream, retries/idempotência).
	•	Integrar TDLN Gate antes dos endpoints que aceitarem NL.
