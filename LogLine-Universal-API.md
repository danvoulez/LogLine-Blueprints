LogLine Universal API — Blueprint v1.0

Resumo

A LogLine Universal API é uma superfície mínima e estável para operar tudo como spans: você envia fatos/intents assinados, e o sistema orquestra agentes, políticas e projeções de estado. Toda a complexidade fica do lado do cliente/CLI/SDK; a API é porteira com TDLN (parser determinístico) que valida e normaliza entradas — inclusive quando chegam “cagadas”.

Proposta de valor
	•	Uma única API para tudo (registry, deploy, wallet/provas, billing, etc.) via intents.
	•	Determinística & auditável (JSON✯Atomic + recibos assinados + prova de inclusão opcional).
	•	Universal nos transportes: polling, SSE, webhooks, (opcional) WebSocket.
	•	LLM-friendly sem risco: TDLN decide; LLM apenas sugere.

Princípios
	1.	API mínima; complexidade no cliente (CLI/SDK/recipes).
	2.	TDLN Gate antes do ledger: aceita perfeita, conserta quando possível, rejeita com orientação.
	3.	Append-only com intents e subjects canônicos.
	4.	Privacidade ativa: dados sensíveis por grants (grant://) materializados no edge.
	5.	Cripto-agilidade: padrão Ed25519+BLAKE3, com espaço para algoritmos futuros.

⸻

Superfície de API (v1)

1) POST /v1/spans — Append de span
	•	Entrada: 1 ou N spans (lote ≤ 100). Podem estar perfeitos ou “brutos”.
	•	Fluxo: TDLN valida/normaliza → aceita ou retorna orientação.
	•	Resposta (por item):
	•	ok: { status:"ok", this, trace_id, receipt{ ts, hash, signature, merkle? } }
	•	need_more: { status:"need_more", missing_fields[], why{}, suggested_reply }
	•	invalid: { status:"invalid", errors[], suggested_reply }
	•	Headers de segurança:
	•	Authorization: ApiKey <TOKEN>
	•	X-Tenant-Id, X-Signature-Alg: ed25519-blake3, X-Signature-KeyId, X-Signature
	•	X-Timestamp (±120s), X-Nonce, Idempotency-Key, X-Payload-Hash: blake3-256=<hex>

2) POST /v1/universal — NL→Span (porta “louca”, opcional)
	•	Entrada: { paragraph, span?, intent_hint?, dry_run? }
	•	Fluxo: TDLN tenta compilar parágrafo → se completo, gera span perfeito; se não, devolve need_more/invalid.
	•	Resposta: idem ao /v1/spans. Se dry_run=true, não anexa ao ledger (somente compiled_span).

3) POST /v1/spans/query — Consulta (polling)
	•	Entrada: { subject, intent:"prefix*"?, since?, cursor?, limit? }
	•	Resposta: { items:[spans], cursor } (idempotente, ordenação temporal).

4) GET /v1/stream — SSE (tempo real)
	•	Query: ?subject=urn:...&intent=prefix*
	•	Resposta: eventos data: <span-json>; reconecta com Last-Event-Id.

5) POST /v1/webhooks — Registro de webhooks
	•	Entrada: { url, events:[ "run_state.*","registry.*", ... ] }
	•	Entrega: POST assinado (Ed25519), retries com backoff + DLQ.

Opcional: WebSocket por Accept-Transport=websocket. Não altera payloads.

⸻

TDLN Gate (porteiro)
	•	Entrada: parágrafo NL e/ou span bruto.
	•	Decisão:
	•	Perfeito → normaliza (ordenação, tipos) e aceita.
	•	Reparável → preenche/renomeia campos; se ficar completo, aceita.
	•	Faltando → need_more com missing_fields, why e suggested_reply.
	•	Ambíguo → invalid e proposta de esclarecimento.
	•	LLM-fallback: só redige resposta amigável e sugestões; não decide.

Schemas mínimos por intent (exemplos)
	•	registry.person.create → name, email, country (ISO-3166), consent, subject
	•	deploy.apply → ref (mc:…), target{ provider, project_id, service }
	•	wallet.proof.submit → proof{ type="ll.proof.v1", intent, result, kid, sig, ts, nonce }

⸻

Modelo de dados (Span canônico — extrato)

{
  "schema_version": "1.1.0",
  "entity_type": "contract|function|decision|agent|law|file|test",
  "this": "urn:span:uuid",
  "prev": "urn:span:prev-uuid",
  "did": { "actor": "usr_123", "action": "append" },
  "intent": "registry.person.create",
  "subject": "urn:registry:person/joana_m_silva",
  "input": { "args": [], "env": {}, "content": "" },
  "payload": { "kind": "registry/person.v1", "name": "..." },
  "ts": "2025-11-17T12:34:56Z",
  "signature": { "kid": "kid_sign_1", "alg": "ed25519-blake3", "sig": "..." }
}


⸻

Transports (universais)
	•	Síncrono (ACK): ok | need_more | invalid + trace_id + receipt.
	•	Assíncrono: SSE (recomendado), webhooks assinados, polling (cursor) e WS (opcional).

⸻

Segurança
	•	Assinatura de request (headers acima).
	•	Anti-replay: X-Timestamp ±120s, X-Nonce único, Idempotency-Key.
	•	Policies: allowlist de intents por tenant, rate limits por intent.
	•	Grants: segredos como grant://… (materializados só no edge/runner).
	•	Recibos: hash, assinatura do servidor e Merkle proof opcional.

⸻

Erros canônicos
	•	401_UNAUTHENTICATED, 403_FORBIDDEN
	•	409_NONCE_REUSED, 409_IDEMPOTENCY_CONFLICT
	•	422_TDLN_INVALID, 422_SCHEMA_INVALID, 422_SIGNATURE_INVALID
	•	428_PROOF_REQUIRED
	•	429_RATE_LIMITED
	•	500_INTERNAL

⸻

Observabilidade
	•	Spans de auditoria: universal.accepted|repaired|rejected, run_state.*, registry.completed.
	•	Métricas: taxa ok/need_more/invalid, latência TDLN, replays bloqueados.
	•	trace_id em tudo (correlação ponta-a-ponta).

⸻

Empacotamento do Produto

Planos (exemplo):
	•	Starter: 100k spans/mês, SSE + polling, 1 webhook, intents básicos (registry, deploy, wallet.proof).
	•	Pro: 5M spans/mês, 10 webhooks, WS opcional, grants, Merkle receipts.
	•	Enterprise: limites sob demanda, BYO S3/Postgres para espelhamento, HA multi-região.

SLAs:
	•	Ingestão p99 < 150ms; SSE first event p99 < 800ms; webhook retry 24h + DLQ.

⸻

Roadmap v1.1
	•	Views nativas (/v1/state/view) para projeções canônicas (opcional; evita reprocessamento do cliente).
	•	Rotas batch otimizadas com compressão NDJSON + zstd.
	•	Criptografia de payload campo-a-campo (envelope XChaCha20 com KDF BLAKE3).

⸻

Exemplos rápidos

NL → Universal (cadastro)

Req:

{ "paragraph": "Quero cadastrar pessoa Joana, email joana@ex.com, país PT, consent ok" }

Resp ok:

{ "status":"ok", "intent":"registry.person.create", "subject":"urn:registry:person/joana_m_silva", "this":"urn:span:..." }

Deploy (span perfeito via CLI/SDK)

Req /v1/spans:

{
  "intent":"deploy.apply",
  "subject":"urn:minicontainer:vv/web",
  "payload":{
    "ref":"mc:vv/web@blake3:abcd",
    "target":{"provider":"railway","project_id":"rw1","service":"web"}
  }
}

Resp ok + SSE de run_state.running → exited.

⸻

Pronto — produto fechado: LogLine Universal API v1.0.
Se estiver ok, na próxima mensagem envio o Blueprint MiniContainer (2/3) e depois LogLine CLI (3/3).
