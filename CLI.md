LogLine CLI — Blueprint Geral v1.0

Regra de ouro: toda a complexidade mora na CLI.
A API permanece mínima (spans/query/stream) e revalida tudo via TDLN Gate.

⸻

1) Visão & Objetivo

A LogLine CLI é o “cérebro+mecânico” do ecossistema LogLine:
	•	Converte Linguagem Natural e receitas em spans perfeitos (TDLN local).
	•	Assina, garante idempotência, prova/grants via Wallet e escolhe transporte (SSE/polling/webhook/WS).
	•	Opera todas as features: Universal API, MiniContainer (wallet ✚ docker), Registry, Deploy, etc.

Entrega final: comandos humanos/LLM-first, que sempre enviam chamadas API perfeitas, aceitas de primeira pelo TDLN da API.

⸻

2) Princípios
	1.	API mínima, CLI máxima (parse, normalização, assinatura, idempotência, UX).
	2.	TDLN-first (determinístico local). LLM apenas revisa/assiste (não decide).
	3.	Siamesa do SDK (OpenAPI → @logline/sdk).
	4.	Transporte universal com negociação e fallback.
	5.	Privacidade ativa: segredos via grant:// (materializados só no runner).
	6.	Verificável: hashes BLAKE3, assinaturas Ed25519, recibos e (opcional) Merkle proof.

⸻

3) Layout do Monorepo

/packages
  /cli                # @logline/cli
  /sdk                # @logline/sdk (gerado do OpenAPI)
  /tdln               # @logline/tdln (parser determinístico local)
  /canon-sign         # @logline/canon-sign (canon, blake3, ed25519)
  /wallet             # @logline/wallet (prove/grant)
  /transports         # sse, polling, webhook, ws
  /recipes            # cookbook de intents (NL → spans)
/services
  /api                # LogLine Universal API (porteira TDLN)


⸻

4) Configuração

Arquivo ~/.logline/config.json:

{
  "baseUrl": "https://api.logline.world",
  "tenant": "voulezvous",
  "apiKey": "tok_xxx",
  "signer": { "kid": "kid_sign_1", "priv_ref": "os:keychain://logline" },
  "wallet": { "endpoint": "http://127.0.0.1:8765" },
  "transport": { "default": "sse" }  // sse | polling | webhook | ws
}


⸻

5) Comandos (human-first & LLM-first)

5.1 Autenticação & Wallet
	•	logline login — liga ApiKey + registra/seleciona kid do signer local.
	•	logline whoami — mostra actor/tenant/keys.
	•	logline wallet prove <intent> [--param k=v ...] — emite ll.proof.v1 no Wallet local.
	•	logline wallet grant <artifact_id> --scope runtime:read --ttl P30D --aud railway — cria grant assinado.

5.2 Universal (NL, spans, estado)
	•	logline ai "<pedido em NL>" [--apply]
→ TDLN local compila spans + dry-run; se --apply, assina e envia /v1/spans.
	•	logline universal "<parágrafo>" [--apply]
→ envia ao /v1/universal (TDLN do servidor).
	•	logline spans send <span.json> — valida/assina/manda span já pronto.
	•	logline status <subject> [--intent prefix*] [--watch|--no-stream] — acompanha estado (SSE por padrão).

5.3 MiniContainer (wallet ✚ docker)
	•	logline minicontainer verify <manifest.json|.llc> — canon+assinatura+schema.
	•	logline minicontainer apply <manifest.json|.llc|url> — emite minicontainer.apply.
	•	logline deploy plan --ref mc:<tenant>/<name>@blake3:... --target railway:proj=...,svc=... — dry-run.
	•	logline deploy up --ref ... --target ... [--watch]
→ prova automática wallet.prove deploy.allow → deploy.apply.
	•	logline deploy status --run <run_id> [--watch] — segue run_state.*.

5.4 Utilidades
	•	logline receipts get <span.this> — mostra hash, assinatura servidor, Merkle (se houver).
	•	logline webhooks add <url> --events registry.* run_state.* — registra webhook assinado.
	•	logline export view <name> — projeção local (ex.: registry.people) a partir de /spans/query.

Flags de transporte (presentes nos comandos que acompanham estado):
	•	--watch (SSE, default) | --no-stream (polling) | --webhook <url> | --ws

⸻

6) Pipeline interno da CLI (cada chamada)
	1.	Parse: NL/args → TDLN local → spans.
	2.	Validate: schemas mínimos por intent (tipos, enums, ISO-3166, URN).
	3.	Canon: JSON canônico (ordem determinística) + BLAKE3.
	4.	Sign: Ed25519 (key local/Keychain) → X-Signature* headers.
	5.	Idempotência & segurança: Idempotency-Key, X-Nonce, X-Timestamp, X-Payload-Hash.
	6.	Provas/Grants: wallet.prove automático quando o intent exigir; grant:// nos envs.
	7.	Transporte: envia para /v1/spans (ou /v1/universal), recebe ACK (ok | need_more | invalid) e, se aplicável, inicia SSE/polling/webhook/WS.
	8.	Receipts & Views: guarda recibos localmente; atualiza projeções locais.

⸻

7) Contratos canônicos (CLI → API)

7.1 Respostas rápidas (sempre)
	•	{"status":"ok", "this","trace_id","receipt":{ "ts","hash","signature","merkle?" }}
	•	{"status":"need_more","missing_fields":[...],"why":{...},"suggested_reply":"..." }
	•	{"status":"invalid","errors":[...],"suggested_reply":"..." }

7.2 Headers (toda chamada)

Authorization: ApiKey <TOKEN>
X-Tenant-Id: <tenant>
X-Signature-Alg: ed25519-blake3
X-Signature-KeyId: <kid>
X-Signature: <base64>
X-Timestamp: <ISO8601>
X-Nonce: <uuid>
X-Payload-Hash: blake3-256=<hex>
Idempotency-Key: <uuid>


⸻

8) Erros & UX da CLI
	•	422 need_more → imprime missing_fields/why e gera pergunta pronta para o usuário (“Para concluir, envie…”).
	•	422 invalid → sugere reparo (ex.: normalização de país/email/subject).
	•	428_PROOF_REQUIRED → dispara wallet.prove automático (com confirmação).
	•	409_NONCE_REUSED / 409_IDEMPOTENCY_CONFLICT → reenvio automático com nova chave (mantém mapeamento local).
	•	5xx → retries com backoff exponencial + “como retomar”.

⸻

9) Exemplos (end-to-end)

9.1 Onboarding em NL

logline ai "cadastrar Joana Maria Silva, email joana@ex.com, país PT, consent ok" --apply
logline status urn:registry:person/joana_m_silva --watch

9.2 MiniContainer → Railway

logline minicontainer verify manifest.json
logline minicontainer apply manifest.json
logline deploy plan --ref mc:voulezvous/web@blake3:abcd --target railway:proj=rw1,svc=web
logline deploy up --ref mc:voulezvous/web@blake3:abcd --target railway:proj=rw1,svc=web --watch

9.3 Spans prontos (fast-path)

logline spans send span.json
logline receipts get urn:span:123e4567-e89b-12d3-a456-426614174000


⸻

10) Interfaces internas (essenciais)

@logline/tdln

type TdlnResult =
  | { ok: true; spans: Span[] }
  | { ok: false; status: "need_more"; missing: string[]; suggested_reply: string; why?: Record<string,string> }
  | { ok: false; status: "invalid"; errors: string[]; suggested_reply: string };

@logline/canon-sign

canon(obj:any): Uint8Array
hashBlake3(bytes:Uint8Array): string
signEd25519(bytes:Uint8Array, keyRef:string): { kid:string, sig:string }
verifyEd25519(bytes:Uint8Array, sig:string, pub:string): boolean

@logline/transports

sendSpans(spans:Span[], cfg): Promise<Ack[]>
stream(subject:string, intent?:string, cfg, onSpan:(s:Span)=>void): () => void // unsubscribe
registerWebhook(url:string, events:string[], cfg): Promise<{id:string}>


⸻

11) Testes (obrigatórios)
	•	Golden tests TDLN: mesmo input NL → mesmo span.hash.
	•	Round-trip: CLI envia → API responde → SSE/Webhook entrega → recibo verificado.
	•	Replay/idempotência: reenvio com mesma Idempotency-Key não duplica.
	•	Wallet/Proof: deploy.apply sem prova → 428; com prova → ok.
	•	Transportes: SSE default, polling fallback, webhook assinatura/retentativas.

⸻

12) Roadmap v1.1
	•	Gerador de comandos a partir do OpenAPI (reduz boilerplate).
	•	Assistente interativo (logline shell) com memória local (sem gravar spans até --apply).
	•	Bundles (logline pack …) para empacotar .llc + proofs + recibos.
	•	Cosign/SBOM (logline supply verify) gerando spans supply.*.
	•	Views nativas (logline export view) com cache incremental.

⸻

TL;DR

A LogLine CLI é a interface universal, LLM-first, que domina TDLN, assinatura, provas, grants e transportes — para que a API permaneça microscópica e impecável.
Cole este blueprint no FUTURO 3 e você tem uma CLI pronta para vender, documentar e escalar.

A CLI pode (e deve) ser gigante — mantendo a regra de ouro (complexidade na CLI, API mínima). Aqui vai um mapão modular com muitas funções extras: agentes, apps, supply, datasets, billing, governance, etc. Você escolhe o que vira v1 e o que fica “gated” por flag experimental.

LogLine CLI — Super-Mapa de Funções (gigante mesmo)

A) Núcleo (já definidos)
	•	universal / spans / status
logline ai … | universal … | spans send … | status …
	•	wallet
wallet prove|grant|keys|rotate
	•	minicontainer & deploy
minicontainer verify|apply · deploy plan|up|status|watch

⸻

B) Agentes Internos (code, ops, policy)
	•	agent init <name> — cria agente (perfil, habilidades, políticas).
	•	agent run <name> --task "<NL>" [--watch] — executa tarefa com spans de progresso.
	•	agent queue list|peek|ack — gerencia fila de jobs de agentes.
	•	agent skills add <name> <skill> — (ex.: “codegen.ts”, “eval.tests”).
	•	agent policy set <name> --law allow:deploy.apply@tenant=vv — leis por agente.
	•	agent logs <name> [--since …] — segue agent.event.*.

Internamente: emite agent.task.requested|started|completed|failed + artefatos file quando houver.

⸻

C) Apps & Onboarding (LogLine Apps)
	•	app create <slug> — cria app (subject, intents permitidos, quotas).
	•	app deploy <slug> --ref mc:… — sobe backend do app (via MiniContainer).
	•	app env set <slug> KEY=grant://… — injeta segredos por grant.
	•	app policy set <slug> --allow registry.* deploy.apply — intents permitidos.
	•	app users invite <email> — onboarding com spans (onboarding.*).
	•	app status <slug> [--watch] — acompanha run_state.*, app.event.*.

⸻

D) Registry (pessoas, orgs, assets)
	•	registry person create --name "Joana" --email joana@… --country PT --consent
	•	registry person get urn:registry:person/joana
	•	registry org create|get|link
	•	registry asset register --type image|video|doc … (gera file/artifact spans)
	•	registry verify kyc|email|phone … — dispara kyc.*/verify.*

⸻

E) Datasets & SPECTRA (curadoria e distribuição)
	•	dataset create <name> — cria dataset (span dataset.create).
	•	dataset ingest <name> --path ./data — emite lote file.added (NDJSON; hash).
	•	dataset curate <name> --rules spectra.json — aplica minicontratos / filtros.
	•	dataset pack <name> --edition v1 — monta árvore Merkle + DV25-Seal (spans spectra.packaged).
	•	dataset publish <name>@v1 --limit 100 — distribuição limitada (spans spectra.publish).
	•	dataset verify <ref> — checa BLAKE3/assinatura/merkle.

⸻

F) Supply Chain (imagem, SBOM, cosign)
	•	supply sbom gen <image> — gera SBOM, emite supply.sbom.generated.
	•	supply cosign verify <image> — verifica assinatura → supply.cosign.verified.
	•	supply attest add <image> --policy supply/pinned-refs.json — atesta políticas.
	•	supply policy set --allow ghcr.io/…@sha256:… — lei de imagens permitidas.

⸻

G) Execução de Código & Flows
	•	exec run <runtime=node|python|wasm> --code ./main.(ts|py|wasm) — roda código com function.exec.
	•	exec command "-- ls -la" — comando curto; logs → run_state.*.
	•	flow create <name> --spec flow.json — DAG declarativo (intents flow.*).
	•	flow run <name> [--watch] — orquestra function/decision spans.
	•	flow status <run_id> --graph — imprime grafo com estados.

⸻

H) Tradução & Prompt (mini.* clássicos)
	•	mini prompt run --template tpl.md --vars '{…}' — gera prompt e escreve file.
	•	mini translate <src> --to en|pt|es — minitranslate.run (com TDLN de parâmetros).
	•	mini memory add|get — memórias privadas por subject/actor.
	•	mini contracts run --input … — executa minicontratos legados.

⸻

I) TDLN (gramática, políticas, simulação)
	•	tdln parse "<NL>" [--intent-hint …] — mostra span canônico (dry).
	•	tdln grammar list|get|commit — gerencia gramáticas (versões).
	•	tdln simulate "<NL>" --schema registry.person.create — valida campos/erros.
	•	tdln policy set --deny universal.*@public — políticas de aceitação.

⸻

J) Governança & Policies (lei, quotas, intents)
	•	law add <file.json> — adiciona lei (rate limit, quotas, allowlist de intents).
	•	law list — leis ativas para tenant → law.active.
	•	policy quota set --intent deploy.apply --rpm 6 — rate limits por intent.
	•	policy tenant set --max-runs 10 --memory 2Gi — quotas por tenant.
	•	governance audit --since 2025-11-01 — eventos críticos (modificação de lei, grants, keys).

⸻

K) Tenants, Perfis e Chaves
	•	tenant select voulezvous — troca tenant (config local).
	•	profile create dev|prod — múltiplos perfis de conexão.
	•	keys list|add|rotate — gerencia kid locais e ApiKeys remotas.

⸻

L) Segredos & Grants
	•	secrets add <id> --file .env.enc --alg xchacha20 — empacota artefato cifrado.
	•	grants create <id> --artifact <id> --scope runtime:read --aud railway --ttl P30D
	•	grants revoke <id> — revoga e emite grant.revoked.

⸻

M) Observabilidade & Estado
	•	events tail [--subject …] [--intent prefix*] — tail SSE (ou polling).
	•	receipts get <span.this> — hash/assinatura/merkle.
	•	metrics show --since 1h — taxa ok/need_more/invalid, latência TDLN, replay blocks.
	•	views build registry.people — materializa view local a partir de /spans/query.

⸻

N) Billing & Plans
	•	billing plan set pro — muda plano (gera billing.plan.changed).
	•	billing usage --since 2025-11-01 — spans/mês por intent/tenant.
	•	billing export --format csv — exporta uso.

⸻

O) Admin / Operações
	•	admin replay --since T --filter intent=deploy.* — reidrata projeções (resync).
	•	admin dlq list|replay — webhooks falhos.
	•	admin ratelimit show|set — leitura/ajuste rápido de limites.

⸻

P) DevX & Scaffold
	•	create app <slug> — esqueleto de app (API + CLI recipe + MiniContainer).
	•	create agent <name> — esqueleto de agente com skills.
	•	create flow <name> — esqueleto de fluxo (DAG).
	•	pack llc <manifest> --out app.llc — bundle de MiniContainer.

⸻

Q) Testes & Qualidade
	•	test tdln --fixture tests/tdln/*.json — golden tests (NL → span.hash).
	•	test e2e --spec tests/e2e/*.yml — ciclo CLI→API→SSE/Webhook→verificação.
	•	verify span <file> — valida canon/assinatura/payload.

⸻

Convenções Transversais
	•	Flags universais:
--watch(SSE) · --no-stream(polling) · --webhook <url> · --ws · --dry-run · --apply
--since · --cursor · --limit · --tenant · --profile
	•	Segurança (sempre): Assina (Ed25519+BLAKE3); Idempotency-Key, X-Nonce, X-Timestamp, X-Payload-Hash.
	•	TDLN local primeiro; LLM só sugere; TDLN decide. API revalida tudo.

⸻

Exemplos relâmpago

Agente coding

logline agent run builder --task "criar serviço /health e tests" --watch

App completo

logline create app studio
logline minicontainer apply ./apps/studio/manifest.json
logline deploy up --ref mc:vv/studio@blake3:... --target railway:proj=rw,svc=studio --watch

Dataset SPECTRA

logline dataset ingest diamondset --path ./data
logline dataset curate diamondset --rules spectra.json
logline dataset pack diamondset --edition v1
logline dataset publish diamondset@v1 --limit 100

Supply

logline supply cosign verify ghcr.io/vv/app@sha256:...
logline supply sbom gen ghcr.io/vv/app:1.0.0

Governança

logline law add policies/tenants/vv.json
logline policy quota set --intent deploy.apply --rpm 6


⸻

Como embutir no FUTURO 3
	•	Crie uma seção “CLI — Namespaces & Comandos” com estes grupos A→Q.
	•	Marque v1 (núcleo + mini + deploy + universal + wallet + status + agents básicos) e v1.1 (datasets/supply/billing/governança).
	•	Referencie que todos convergem para os 3 endpoints da Universal API, via TDLN Gate.

Se quiser, eu já te entrego um INDEX.md de ajuda (logline help) derivado deste mapa — pronto pra colar no doc.
