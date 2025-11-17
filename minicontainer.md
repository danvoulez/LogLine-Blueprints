Blueprint — MiniContainer (Wallet ✚ Docker, TDLN-first)

MiniContainer — Blueprint v1.0

Resumo

MiniContainer é um artefato portátil, assinado e auditável que unifica:
	•	Wallet (identidade, chaves, provas, grants/segredos)
	•	Docker/Podman (deploy declarativo)

Tudo descrito em JSON✯Atomic, validado por TDLN Gate e operado via Universal API.
Entrega 3 modos de uso (CLI, SDK, API direta) — sempre passando pelo TDLN.

⸻

1) Objetivos
	•	Um arquivo, dois mundos: mesmo manifesto para identidade/segredos e execução de container.
	•	LLM/CLI-first: manifesto legível, fácil de gerar e revisar.
	•	Privacidade ativa: segredos via grant://, materializados só no runner (edge).
	•	Auditável: proveniência com Ed25519+BLAKE3, recibos e prova de inclusão (Merkle, opcional).
	•	Portável: adapters de target (Railway/Podman/…).

⸻

2) Forma do artefato
	•	Arquivo: minicontainer.llc (zip) com:
	•	manifest.json (obrigatório)
	•	artifacts/* (opcional; cifrados)
	•	Assinatura de proveniência dentro do manifest.json.

⸻

3) Manifesto (campos canônicos — v1)

{
  "type": "ll.minicontainer.v1",
  "version": "1.0.0",
  "owner": { "user_id": "usr_123", "tenant": "voulezvous" },

  "slots": {
    "signing": [{ "kid":"kid_sign_1", "pub":"hex...", "status":"active" }],
    "api":     [{ "token_id":"tok_cli", "scopes":["deploy:*","prove:*"], "status":"active" }],
    "crypto":  [{ "kid":"kid_sym_1", "alg":"xchacha20", "status":"active" }]
  },

  "artifacts": [
    { "id":"secret_db_url", "type":"secret.url.v1", "hash":"blake3:abcd...", "enc":"xchacha20" }
  ],

  "grants": [
    { "id":"grt_db", "artifact_id":"secret_db_url", "scope":["runtime:read"], "ttl":"P30D", "aud":"railway", "sig":"ed25519:..." }
  ],

  "deploy": {
    "engine":"container",
    "image":"ghcr.io/vv/app:1.0.0",
    "entrypoint": ["node","server.js"],
    "env": { "DB_URL":"grant://grt_db", "PORT":"8080" },
    "ports":[{ "host":0, "container":8080 }],
    "resources": { "memory_bytes": 536870912 },
    "security": { "read_only_rootfs": true },
    "targets": [
      { "provider":"railway", "project_id":"rw_proj", "service":"web" }
    ]
  },

  "ledger_refs": ["s3://ledger/vv/2025/11/"],
  "ts": "2025-11-17T13:37:00Z",

  "provenance": { "kid":"kid_sign_1", "alg":"ed25519-blake3", "sig":"BASE64..." }
}

Observações
	•	env aceita grant://... (segredos nunca em claro).
	•	targets[] seleciona adapter (Railway/Podman/…).

⸻

4) Segurança & Canon
	•	Canon JSON: ordenação determinística de chaves, arrays preservam ordem; hash BLAKE3; assinatura Ed25519.
	•	Headers de request (quando enviar a API): X-Nonce, X-Timestamp (±120s), Idempotency-Key, X-Payload-Hash.
	•	TDLN Gate valida: schema, assinatura de provenance, consistência de campos.

⸻

5) Operação (intents do ecossistema)

Todos os fluxos passam pelo LogLine Universal API e TDLN.

5.1 Declarar/atualizar MiniContainer
	•	Intent: minicontainer.apply
	•	Subject: urn:minicontainer:<tenant>/<name>
	•	Payload: manifest (inline ou ref)

5.2 Provar autorização (Wallet)
	•	Intent: wallet.proof.submit (ex.: intent:"deploy.allow")
	•	TDLN verifica ll.proof.v1 e emite wallet.proof.verified

5.3 Deploy (adapter)
	•	Intent: deploy.apply
	•	Payload: { ref, target{ provider, project_id, service }, options? }
	•	Evolução de estado: deploy.planned → deploy.applied → run_state.*

5.4 Materialização de grants (edge)
	•	Intent: grant.materialize (gerado pelo runner)
	•	Nunca expõe o segredo; apenas hash/refs.

⸻

6) Modos de uso (3 caminhos oficiais)

Modo A — CLI (recomendado)
	•	Comandos:
	•	logline minicontainer verify manifest.json
	•	logline minicontainer apply manifest.json
	•	logline deploy up --ref mc:vv/web@blake3:... --target railway:proj=rw1,svc=web --watch
	•	O que a CLI faz:
	•	Canon+sign, prova automática (wallet.prove deploy.allow), idempotência, transporte (SSE/polling/webhook), TDLN pré-checado.
	•	Chegada: POST /v1/spans com spans perfeitos (TDLN só confere e aceita).

Modo B — SDK (JS/TS)

const sdk = new LogLineSDK({ baseUrl, apiKey, tenant, signer });
await sdk.append({
  intent: "minicontainer.apply",
  subject: "urn:minicontainer:vv/web",
  payload: { manifest }          // inline ou ref
});
await sdk.append({
  intent: "wallet.proof.submit",
  subject: "urn:minicontainer:vv/web",
  payload: { proof }             // ll.proof.v1 emitido pelo Wallet
});
await sdk.append({
  intent: "deploy.apply",
  subject: "urn:minicontainer:vv/web",
  payload: {
    ref: "mc:vv/web@blake3:...",
    target: { provider:"railway", project_id:"rw1", service:"web" }
  }
});

	•	Chegada: igual ao CLI (spans perfeitos).
	•	SSE/webhooks para run_state.*.

Modo C — API direta (sem SDK)
	•	Duas portas:
	1.	POST /v1/spans — envie o span (perfeito ou não)
	2.	POST /v1/universal — envie parágrafo NL com manifest/ref
	•	TDLN Gate:
	•	se completo → ok + recibo;
	•	se faltar → need_more/invalid com suggested_reply.

⸻

7) Adapters de Target
	•	Railway (v1): plano → set env (grant:// → segredo no runner) → build/deploy → run_state.*
	•	Podman/Docker (v1): toPodmanArgs() a partir do deploy.*
	•	Outros: mesma interface (plan/apply/status), sempre emitindo spans de estado.

⸻

8) Estados e Eventos
	•	minicontainer.applied (ok/erro + por quê)
	•	wallet.proof.verified (allow/deny)
	•	deploy.planned → deploy.applied
	•	run_state.pending|running|exited|failed (exit_code, tempos, node/provider)
	•	grant.materialized (hash, nunca segredo)

Transporte: SSE (padrão), polling (fallback), webhooks (parceiros), WS (opcional).

⸻

9) Erros canônicos
	•	422_SCHEMA_INVALID (manifest malformado)
	•	422_SIGNATURE_INVALID (proveniência/assinatura inválida)
	•	428_PROOF_REQUIRED (faltou prova para deploy)
	•	409_IDEMPOTENCY_CONFLICT, 409_NONCE_REUSED
	•	403_FORBIDDEN (intent/tenant/policy)
	•	500_INTERNAL

⸻

10) Quick Start (de ponta a ponta)

1) Verificar e aplicar

logline minicontainer verify manifest.json
logline minicontainer apply manifest.json

2) Provar e deployar

logline wallet prove deploy.allow --tenant voulezvous
logline deploy up --ref mc:voulezvous/web@blake3:abcd --target railway:proj=rw1,svc=web --watch

3) Status

logline status urn:minicontainer:voulezvous/web --watch
# ou polling:
logline status urn:minicontainer:voulezvous/web --no-stream


⸻

11) Roadmap v1.1
	•	Healthchecks no deploy.lifecycle (timeouts, retries)
	•	Policies declarativas por tenant (quotas, imagens permitidas, privileged)
	•	Cosign/SBOM: verificação de imagem (intents supply.verify)
	•	Views materializadas (estado direto em /v1/state/view)

⸻

Conclusão: MiniContainer pega sua visão de junho (wallet ✚ deploy) e a entrega redonda: um artefato único, TDLN-first, três modos de uso e operação universal pelos mesmos intents.
Se estiver ok, parto para [3/3] Blueprint — LogLine CLI.
