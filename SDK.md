LogLine SDK — Blueprint v1.0

Objetivo

Fornecer clientes oficiais (TS primeiro; Python/Go gerados) que:
	•	Assinam e idempotizam chamadas (Ed25519+BLAKE3, Nonce/Timestamp, Idempotency-Key).
	•	Lidam com transportes universais (ACK, Polling, SSE, Webhooks, WS opcional).
	•	Expõem APIs de alto nível (registry, deploy, minicontainer, wallet) e baixo nível (append(), query(), stream()).

⸻

Pacotes
	•	@logline/sdk (TypeScript, fonte da verdade)
	•	@logline/canon-sign (canon JSON, BLAKE3, Ed25519)
	•	@logline/tdln-client (opcional: NL→span no servidor via /v1/universal)
	•	Bindings gerados: logline-sdk-py, logline-sdk-go (OpenAPI 3.1)

⸻

Núcleo (TS)

Tipos essenciais

export type EntityType = "contract" | "function" | "decision" | "agent" | "law" | "file" | "test";

export interface Span {
  schema_version: "1.1.0";
  entity_type: EntityType;
  this: string;
  prev?: string;
  did: { actor: string; action: "append" };
  intent: string;            // ex: "deploy.apply"
  subject: string;           // ex: "urn:minicontainer:vv/web"
  input?: { args?: any[]; env?: Record<string, any>; content?: string; bytes_b64?: string };
  payload?: any;
  output?: { stdout?: string; stderr?: string; exit_code?: number; [k: string]: any };
  ts: string;
  signature?: { kid: string; alg: "ed25519-blake3"; sig: string };
}

export type AckOk = { status: "ok"; this: string; trace_id: string; receipt: { ts: string; hash: string; signature: string; merkle?: string } };
export type AckNeedMore = { status: "need_more"; trace_id: string; missing_fields: string[]; why?: Record<string,string>; suggested_reply: string };
export type AckInvalid = { status: "invalid"; trace_id: string; errors: string[]; suggested_reply: string };
export type Ack = AckOk | AckNeedMore | AckInvalid;

export interface QueryRequest { subject: string; intent?: string; since?: string; cursor?: string; limit?: number; }
export interface QueryResponse { items: Span[]; cursor?: string; }

Cliente base

export class LogLineSDK {
  constructor(private cfg: {
    baseUrl: string;
    apiKey: string;
    tenant: string;
    signer: { kid: string; sign(bytes: Uint8Array): Promise<Uint8Array> }; // Ed25519
    fetchImpl?: typeof fetch; // injeta polyfill em Node
    defaultTransport?: "sse" | "polling" | "websocket";
  }) {}

  // baixo nível
  async append(spans: Span | Span[]): Promise<Ack[]> { /* assina headers, idempotiza, POST /v1/spans */ }
  async query(req: QueryRequest): Promise<QueryResponse> { /* POST /v1/spans/query */ }
  stream(params: {subject: string; intent?: string}, onSpan:(s:Span)=>void): () => void { /* GET /v1/stream (SSE) */ }

  // webhooks
  async registerWebhook(url: string, events: string[]): Promise<{ id: string }> { /* POST /v1/webhooks */ }

  // alto nível (conveniências)
  registry = new RegistryAPI(this);
  deploy   = new DeployAPI(this);
  wallet   = new WalletAPI(this);
  mini     = new MiniContainerAPI(this);
  universal = new UniversalAPI(this); // /v1/universal (TDLN server-side)
}

Assinatura & cabeçalhos (automático no append)
	•	Authorization: ApiKey
	•	X-Tenant-Id
	•	X-Signature-Alg: ed25519-blake3
	•	X-Signature-KeyId, X-Signature
	•	X-Timestamp (±120s), X-Nonce (uuid)
	•	Idempotency-Key (uuid)
	•	X-Payload-Hash: blake3-256=<hex>

⸻

Módulos de Alto Nível

Registry

class RegistryAPI {
  constructor(private sdk: LogLineSDK) {}
  async personCreate(p: { name: string; email: string; country: string; consent: boolean; subject: string }): Promise<Ack> {
    const span: Span = {
      schema_version: "1.1.0",
      entity_type: "contract",
      this: genSpanUrn(),
      did: { actor: "sdk", action: "append" },
      intent: "registry.person.create",
      subject: p.subject,
      payload: { kind:"registry/person.v1", ...p },
      ts: new Date().toISOString()
    };
    return (await this.sdk.append(span))[0];
  }
}

Deploy & MiniContainer

class MiniContainerAPI {
  constructor(private sdk: LogLineSDK) {}
  async apply(subject: string, manifest: any): Promise<Ack> {
    return (await this.sdk.append({
      schema_version: "1.1.0",
      entity_type: "contract",
      this: genSpanUrn(),
      did: { actor: "sdk", action: "append" },
      intent: "minicontainer.apply",
      subject,
      payload: { manifest },
      ts: new Date().toISOString()
    }))[0];
  }
}

class DeployAPI {
  constructor(private sdk: LogLineSDK) {}
  async apply(subject: string, ref: string, target: {provider:string; project_id:string; service:string}, options?: any): Promise<Ack> {
    return (await this.sdk.append({
      schema_version: "1.1.0",
      entity_type: "function",
      this: genSpanUrn(),
      did: { actor: "sdk", action: "append" },
      intent: "deploy.apply",
      subject,
      payload: { ref, target, options },
      ts: new Date().toISOString()
    }))[0];
  }
  status(subject: string, onSpan:(s:Span)=>void) { return this.sdk.stream({subject, intent:"run_state.*"}, onSpan); }
}

Wallet (provas & grants)

class WalletAPI {
  constructor(private sdk: LogLineSDK) {}
  async submitProof(subject: string, proof: any): Promise<Ack> {
    return (await this.sdk.append({
      schema_version: "1.1.0",
      entity_type: "decision",
      this: genSpanUrn(),
      did: { actor: "sdk", action: "append" },
      intent: "wallet.proof.submit",
      subject,
      payload: { proof },
      ts: new Date().toISOString()
    }))[0];
  }
}

Universal (NL → span no servidor, opcional)

class UniversalAPI {
  constructor(private sdk: LogLineSDK) {}
  async parse(paragraph: string, intent_hint?: string, dry_run = true) {
    return this.sdk._post("/v1/universal", { paragraph, intent_hint, dry_run }) as Promise<Ack>;
  }
}


⸻

Uso rápido (TS)

const sdk = new LogLineSDK({
  baseUrl: "https://api.logline.world",
  apiKey: process.env.LOGLINE_API!,
  tenant: "voulezvous",
  signer: myEd25519Signer // { kid, sign(bytes) }
});

// MiniContainer → Deploy
await sdk.mini.apply("urn:minicontainer:voulezvous/web", manifest);
await sdk.wallet.submitProof("urn:minicontainer:voulezvous/web", proof);
const ack = await sdk.deploy.apply("urn:minicontainer:voulezvous/web", "mc:voulezvous/web@blake3:abcd", {provider:"railway", project_id:"rw1", service:"web"});
if (ack.status === "ok") {
  const stop = sdk.deploy.status("urn:minicontainer:voulezvous/web", span => console.log("RUN:", span.intent, span.payload));
  // … depois
  stop();
}


⸻

Polling (fallback) e SSE

// polling
const q1 = await sdk.query({ subject:"urn:minicontainer:voulezvous/web", intent:"run_state.*", since:new Date(Date.now()-60_000).toISOString() });

// SSE
const stop = sdk.stream({ subject:"urn:minicontainer:voulezvous/web", intent:"run_state.*" }, s => console.log("event:", s.intent));
// stop() para encerrar


⸻

Retentativas & Idempotência
	•	Retries com backoff exponencial para 5xx/transiente.
	•	Idempotency-Key gerado por request; conflito (409) → o SDK retorna o recibo anterior.
	•	Nonce/Timestamp: renovados por request; tolerância de 120s.

⸻

Erros canônicos (mapeados para exceções)
	•	UnauthorizedError (401), ForbiddenError (403)
	•	RateLimitedError (429) (expondo retryAfter)
	•	SchemaInvalid (422_TDLN_INVALID|SCHEMA_INVALID)
	•	ProofRequired (428)
	•	IdempotencyConflict (409) (com recibo anterior)
	•	Internal (5xx)

⸻

Browser & Node
	•	Node: usa node:crypto por padrão (ou libsodium se quiser).
	•	Browser: usa WebCrypto (Ed25519 via Subtle/Polyfill), SSE nativo via EventSource.
	•	React/Next: hook useStream(subject, intent) no pacote opcional @logline/react.

⸻

Geração Python/Go
	•	Expor OpenAPI 3.1 com 3 endpoints (/v1/spans, /v1/spans/query, /v1/stream + /v1/universal, /v1/webhooks).
	•	Python: classe LogLineSDK com append, query, stream_sse, register_webhook e helpers (registry/deploy/mini/wallet).
	•	Go: cliente com contexto e backoff, canal Stream(ctx) para SSE.

⸻

Testes
	•	Golden: o mesmo span → mesmo hash via @logline/canon-sign.
	•	Round-trip: append() → ACK → stream() entrega → verifica recibo.
	•	Idempotência: reenvio com a mesma Idempotency-Key não duplica.
	•	Erro: 428 sem prova → 200 após submitProof.

⸻

Roadmap v1.1
	•	Middleware (before/after request) para métricas e tracing.
	•	Batch compressor (NDJSON + zstd) no append.
	•	Views helpers: sdk.views.registry.people({since}) (projeções frequentes).
	•	Cosign/SBOM helpers (supply chain).

⸻

TL;DR

O SDK empacota assinatura + idempotência + transportes + helpers de alto nível e fala fluentemente com a Universal API. TS é a fonte; Python/Go vêm do OpenAPI. Ele é o par perfeito da CLI: onde a CLI orquestra experiência, o SDK dá power programático com o mesmo contrato mínimo.
