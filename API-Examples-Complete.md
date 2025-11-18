# LogLine Universal API — Complete Examples

**Version:** 1.0.0
**Date:** 2025-11-17

This document provides end-to-end examples of using the LogLine Universal API, including request/response pairs for all major workflows.

---

## Table of Contents

1. [Authentication](#1-authentication)
2. [Person Registry Workflow](#2-person-registry-workflow)
3. [Deploy Workflow](#3-deploy-workflow)
4. [Wallet & Proofs](#4-wallet--proofs)
5. [MiniContainer Workflow](#5-minicontainer-workflow)
6. [Natural Language (Universal) Workflow](#6-natural-language-universal-workflow)
7. [Streaming & Webhooks](#7-streaming--webhooks)
8. [Error Handling Examples](#8-error-handling-examples)
9. [Mini Suite Examples](#9-mini-suite-examples)

---

## 1. Authentication

All requests require an API key in the `Authorization` header.

### Request Headers (All Endpoints)

```http
Authorization: ApiKey tok_abc123xyz
X-Tenant-Id: voulezvous
X-Env: prd
```

For signed requests (recommended):

```http
Authorization: ApiKey tok_abc123xyz
X-Tenant-Id: voulezvous
X-Env: prd
X-Signature-Alg: ed25519-blake3
X-Signature-KeyId: kid_sign_1
X-Signature: SGVsbG8gV29ybGQ=
X-Timestamp: 2025-11-17T12:34:56Z
X-Nonce: 7c9e6679-7425-40de-944b-e07fc1f90ae7
X-Payload-Hash: blake3-256=abcd1234567890abcdef...
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

---

## 2. Person Registry Workflow

### 2.1 Create Person (Complete Span)

**Request:**

```http
POST /v1/spans
Content-Type: application/json
Authorization: ApiKey tok_abc123xyz
X-Tenant-Id: voulezvous
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440001
```

```json
{
  "schema_version": "1.1.0",
  "entity_type": "contract",
  "this": "urn:span:550e8400-e29b-41d4-a716-446655440001",
  "did": {
    "actor": "usr_abc123",
    "action": "append"
  },
  "intent": "registry.person.create",
  "subject": "urn:registry:person/joana_m_silva",
  "payload": {
    "kind": "registry/person.v1",
    "name": "Joana Maria Silva",
    "email": "joana@example.com",
    "country": "PT",
    "consent": true,
    "phone": "+351912345678"
  },
  "ts": "2025-11-17T12:34:56Z"
}
```

**Response (200 OK):**

```json
{
  "status": "ok",
  "this": "urn:span:550e8400-e29b-41d4-a716-446655440001",
  "trace_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "receipt": {
    "ts": "2025-11-17T12:34:56.123Z",
    "hash": "blake3:abcd1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab",
    "signature": "SGVsbG8gV29ybGQgZnJvbSBMb2dMaW5lIQ==",
    "merkle": "blake3:merkle_root_hash..."
  }
}
```

### 2.2 Create Person (Partial Span - Need More)

**Request:**

```http
POST /v1/spans
Content-Type: application/json
```

```json
{
  "schema_version": "1.1.0",
  "entity_type": "contract",
  "this": "urn:span:550e8400-e29b-41d4-a716-446655440002",
  "did": {
    "actor": "usr_abc123",
    "action": "append"
  },
  "intent": "registry.person.create",
  "subject": "urn:registry:person/joana",
  "payload": {
    "kind": "registry/person.v1",
    "name": "Joana Silva"
  },
  "ts": "2025-11-17T12:35:00Z"
}
```

**Response (200 OK - need_more):**

```json
{
  "status": "need_more",
  "trace_id": "8d0e7780-8536-41ef-a827-f18fc2g01bf8",
  "missing_fields": ["email", "country", "consent"],
  "why": {
    "email": "Email address is required for person registration",
    "country": "Country code (ISO-3166 alpha-2) is required",
    "consent": "Privacy consent must be explicitly provided"
  },
  "suggested_reply": "To complete this request, please provide:\n- email: Email address is required for person registration\n- country: Country code (ISO-3166 alpha-2) is required\n- consent: Privacy consent must be explicitly provided"
}
```

### 2.3 Create Person (Invalid Fields)

**Request:**

```json
{
  "schema_version": "1.1.0",
  "entity_type": "contract",
  "this": "urn:span:550e8400-e29b-41d4-a716-446655440003",
  "did": {"actor": "usr_abc123", "action": "append"},
  "intent": "registry.person.create",
  "subject": "urn:registry:person/joana",
  "payload": {
    "kind": "registry/person.v1",
    "name": "Joana",
    "email": "not-an-email",
    "country": "XX",
    "consent": true
  },
  "ts": "2025-11-17T12:36:00Z"
}
```

**Response (200 OK - invalid):**

```json
{
  "status": "invalid",
  "trace_id": "9e1f8891-9647-42f0-b938-g29gd3h12cg9",
  "errors": [
    "Invalid email format: not-an-email. Expected: user@domain.com",
    "Invalid country code: XX. Use ISO-3166 alpha-2 (e.g., PT, US, BR)"
  ],
  "suggested_reply": "Please fix the following:\n- Use valid email format: user@domain.com\n- Use ISO-3166 alpha-2 country code (e.g., PT, US, BR)"
}
```

### 2.4 Query Person Spans

**Request:**

```http
POST /v1/spans/query
Content-Type: application/json
```

```json
{
  "subject": "urn:registry:person/joana_m_silva",
  "intent": "registry.person.*",
  "limit": 10
}
```

**Response:**

```json
{
  "items": [
    {
      "schema_version": "1.1.0",
      "entity_type": "contract",
      "this": "urn:span:550e8400-e29b-41d4-a716-446655440001",
      "did": {"actor": "usr_abc123", "action": "append"},
      "intent": "registry.person.create",
      "subject": "urn:registry:person/joana_m_silva",
      "payload": {
        "kind": "registry/person.v1",
        "name": "Joana Maria Silva",
        "email": "joana@example.com",
        "country": "PT",
        "consent": true,
        "phone": "+351912345678"
      },
      "ts": "2025-11-17T12:34:56Z"
    }
  ],
  "cursor": null
}
```

---

## 3. Deploy Workflow

### 3.1 Deploy (Requires Proof)

**Request (without proof):**

```http
POST /v1/spans
```

```json
{
  "schema_version": "1.1.0",
  "entity_type": "function",
  "this": "urn:span:650e8400-e29b-41d4-a716-446655440010",
  "did": {"actor": "usr_abc123", "action": "append"},
  "intent": "deploy.apply",
  "subject": "urn:minicontainer:vv/web",
  "payload": {
    "kind": "deploy/apply.v1",
    "ref": "mc:vv/web@blake3:abcd1234567890abcdef1234567890abcdef1234567890abcdef1234567890ab",
    "target": {
      "provider": "railway",
      "project_id": "rw_proj_123",
      "service": "web"
    },
    "options": {
      "auto_scale": true,
      "replicas": 3
    }
  },
  "ts": "2025-11-17T13:00:00Z"
}
```

**Response (428 Proof Required):**

```json
{
  "code": "PROOF_REQUIRED",
  "message": "deploy.apply requires wallet.proof for intent deploy.allow",
  "trace_id": "a12b3456-c789-4def-0123-456789abcdef",
  "details": {
    "required_proof_intent": "deploy.allow",
    "subject": "urn:minicontainer:vv/web"
  }
}
```

### 3.2 Submit Proof

**Request:**

```http
POST /v1/spans
```

```json
{
  "schema_version": "1.1.0",
  "entity_type": "decision",
  "this": "urn:span:750e8400-e29b-41d4-a716-446655440011",
  "did": {"actor": "usr_abc123", "action": "append"},
  "intent": "wallet.proof.submit",
  "subject": "urn:wallet:vv",
  "payload": {
    "kind": "wallet/proof.v1",
    "proof": {
      "type": "ll.proof.v1",
      "intent": "deploy.allow",
      "result": "allow",
      "kid": "kid_sign_1",
      "sig": "U2lnbmF0dXJlIG9mIGRlcGxveS5hbGxvdw==",
      "ts": "2025-11-17T12:59:00Z",
      "nonce": "b23c4567-d890-5ef0-1234-567890abcdef"
    }
  },
  "ts": "2025-11-17T12:59:30Z"
}
```

**Response:**

```json
{
  "status": "ok",
  "this": "urn:span:750e8400-e29b-41d4-a716-446655440011",
  "trace_id": "c34d5678-e901-6fg0-2345-678901bcdefg",
  "receipt": {
    "ts": "2025-11-17T12:59:30.456Z",
    "hash": "blake3:proof_hash...",
    "signature": "UHJvb2Ygc2lnbmF0dXJl"
  }
}
```

### 3.3 Deploy (With Proof)

Now retry the deploy from 3.1:

**Response (200 OK):**

```json
{
  "status": "ok",
  "this": "urn:span:650e8400-e29b-41d4-a716-446655440010",
  "trace_id": "d45e6789-f012-7gh0-3456-789012cdefgh",
  "receipt": {
    "ts": "2025-11-17T13:00:01.789Z",
    "hash": "blake3:deploy_hash...",
    "signature": "RGVwbG95IHNpZ25hdHVyZQ=="
  }
}
```

### 3.4 Stream Deploy Status (SSE)

**Request:**

```http
GET /v1/stream?subject=urn:minicontainer:vv/web&intent=run_state.*
X-Tenant-Id: voulezvous
Authorization: ApiKey tok_abc123xyz
```

**Response (text/event-stream):**

```
id: 1
event: span
data: {"schema_version":"1.1.0","entity_type":"function","this":"urn:span:850e8400...","intent":"run_state.pending","subject":"urn:minicontainer:vv/web","payload":{"kind":"run_state/pending.v1","run_id":"run_abc123","ref":"mc:vv/web@blake3:abcd...","target":{"provider":"railway","project_id":"rw_proj_123","service":"web"}},"ts":"2025-11-17T13:00:02Z"}

id: 2
event: span
data: {"schema_version":"1.1.0","entity_type":"function","this":"urn:span:950e8400...","intent":"run_state.running","subject":"urn:minicontainer:vv/web","payload":{"kind":"run_state/running.v1","run_id":"run_abc123","started_at":"2025-11-17T13:00:15Z","node":"railway-node-42"},"ts":"2025-11-17T13:00:15Z"}

id: 3
event: span
data: {"schema_version":"1.1.0","entity_type":"function","this":"urn:span:a50e8400...","intent":"run_state.exited","subject":"urn:minicontainer:vv/web","payload":{"kind":"run_state/exited.v1","run_id":"run_abc123","exit_code":0,"exited_at":"2025-11-17T13:05:30Z","duration_seconds":315},"ts":"2025-11-17T13:05:30Z"}
```

---

## 4. Wallet & Proofs

### 4.1 Create Grant

**Request:**

```json
{
  "schema_version": "1.1.0",
  "entity_type": "decision",
  "this": "urn:span:b60e8400-e29b-41d4-a716-446655440020",
  "did": {"actor": "usr_abc123", "action": "append"},
  "intent": "wallet.grant.create",
  "subject": "urn:wallet:vv",
  "payload": {
    "kind": "wallet/grant.v1",
    "artifact_id": "secret_db_url",
    "scope": ["runtime:read"],
    "ttl": "P30D",
    "aud": "railway"
  },
  "ts": "2025-11-17T14:00:00Z"
}
```

**Response:**

```json
{
  "status": "ok",
  "this": "urn:span:b60e8400-e29b-41d4-a716-446655440020",
  "trace_id": "e56f7890-g123-8hi0-4567-890123defghi",
  "receipt": {
    "ts": "2025-11-17T14:00:00.234Z",
    "hash": "blake3:grant_hash...",
    "signature": "R3JhbnQgc2lnbmF0dXJl"
  }
}
```

### 4.2 Revoke Grant

**Request:**

```json
{
  "schema_version": "1.1.0",
  "entity_type": "decision",
  "this": "urn:span:c70e8400-e29b-41d4-a716-446655440021",
  "did": {"actor": "usr_abc123", "action": "append"},
  "intent": "wallet.grant.revoke",
  "subject": "urn:wallet:vv",
  "payload": {
    "kind": "wallet/grant.v1",
    "grant_id": "grt_db_20251117",
    "reason": "Credential rotation"
  },
  "ts": "2025-11-17T14:30:00Z"
}
```

**Response:**

```json
{
  "status": "ok",
  "this": "urn:span:c70e8400-e29b-41d4-a716-446655440021",
  "trace_id": "f67g8901-h234-9ij0-5678-901234efghij",
  "receipt": {
    "ts": "2025-11-17T14:30:00.567Z",
    "hash": "blake3:revoke_hash...",
    "signature": "UmV2b2tlIHNpZ25hdHVyZQ=="
  }
}
```

---

## 5. MiniContainer Workflow

### 5.1 Apply MiniContainer Manifest

**Request:**

```json
{
  "schema_version": "1.1.0",
  "entity_type": "contract",
  "this": "urn:span:d80e8400-e29b-41d4-a716-446655440030",
  "did": {"actor": "usr_abc123", "action": "append"},
  "intent": "minicontainer.apply",
  "subject": "urn:minicontainer:vv/web",
  "payload": {
    "kind": "minicontainer/manifest.v1",
    "manifest": {
      "type": "ll.minicontainer.v1",
      "version": "1.0.0",
      "owner": {
        "user_id": "usr_abc123",
        "tenant": "vv"
      },
      "slots": {
        "signing": [
          {
            "kid": "kid_sign_1",
            "pub": "abcd1234...",
            "status": "active"
          }
        ]
      },
      "deploy": {
        "engine": "container",
        "image": "ghcr.io/vv/app:1.0.0",
        "entrypoint": ["node", "server.js"],
        "env": {
          "DB_URL": "grant://grt_db",
          "PORT": "8080"
        },
        "ports": [{"host": 0, "container": 8080}],
        "resources": {"memory_bytes": 536870912}
      },
      "ts": "2025-11-17T15:00:00Z",
      "provenance": {
        "kid": "kid_sign_1",
        "alg": "ed25519-blake3",
        "sig": "TWFuaWZlc3Qgc2lnbmF0dXJl"
      }
    }
  },
  "ts": "2025-11-17T15:00:00Z"
}
```

**Response:**

```json
{
  "status": "ok",
  "this": "urn:span:d80e8400-e29b-41d4-a716-446655440030",
  "trace_id": "g78h9012-i345-0jk0-6789-012345fghijk",
  "receipt": {
    "ts": "2025-11-17T15:00:00.890Z",
    "hash": "blake3:manifest_hash...",
    "signature": "TWluaWNvbnRhaW5lciBzaWc="
  }
}
```

---

## 6. Natural Language (Universal) Workflow

### 6.1 NL to Person Registration

**Request:**

```http
POST /v1/universal
Content-Type: application/json
```

```json
{
  "paragraph": "Cadastrar pessoa Joana Maria Silva, email joana@example.com, país Portugal, consent ok",
  "dry_run": false
}
```

**Response:**

```json
{
  "status": "ok",
  "this": "urn:span:e90e8400-e29b-41d4-a716-446655440040",
  "trace_id": "h89i0123-j456-1kl0-7890-123456ghijkl",
  "compiled_span": {
    "schema_version": "1.1.0",
    "entity_type": "contract",
    "this": "urn:span:e90e8400-e29b-41d4-a716-446655440040",
    "did": {"actor": "system", "action": "append"},
    "intent": "registry.person.create",
    "subject": "urn:registry:person/joana_m_silva",
    "payload": {
      "kind": "registry/person.v1",
      "name": "Joana Maria Silva",
      "email": "joana@example.com",
      "country": "PT",
      "consent": true
    },
    "ts": "2025-11-17T16:00:00Z"
  },
  "receipt": {
    "ts": "2025-11-17T16:00:00.123Z",
    "hash": "blake3:nl_span_hash...",
    "signature": "Tkwgc3BhbiBzaWduYXR1cmU="
  }
}
```

### 6.2 NL Ambiguous Input

**Request:**

```json
{
  "paragraph": "criar algo",
  "dry_run": true
}
```

**Response:**

```json
{
  "status": "invalid",
  "trace_id": "i90j1234-k567-2lm0-8901-234567hijklm",
  "errors": [
    "Cannot determine intent from input: 'criar algo'",
    "Ambiguous request - insufficient information"
  ],
  "suggested_reply": "Please clarify what you want to create. Examples:\n- 'criar pessoa [name]'\n- 'criar organização [name]'\n- 'deploy container [ref]'"
}
```

### 6.3 TDLN Assist (Advisory)

**Request:**

```http
POST /v1/tdln/assist
```

```json
{
  "paragraph": "person João, living in Brazil, email joao@example.com"
}
```

**Response:**

```json
{
  "draft": {
    "schema_version": "1.1.0",
    "entity_type": "contract",
    "intent": "registry.person.create",
    "subject": "urn:registry:person/joao",
    "payload": {
      "kind": "registry/person.v1",
      "name": "João",
      "email": "joao@example.com",
      "country": "BR"
    }
  },
  "missing": ["consent"],
  "confidence": 0.75,
  "advisor": {
    "model": "qwen2.5-7b-instruct",
    "provider": "vllm",
    "prompt_hash": "blake3:prompt_hash..."
  }
}
```

---

## 7. Streaming & Webhooks

### 7.1 Register Webhook

**Request:**

```http
POST /v1/webhooks
Content-Type: application/json
```

```json
{
  "url": "https://example.com/webhooks/logline",
  "events": ["run_state.*", "registry.person.*", "deploy.*"]
}
```

**Response:**

```json
{
  "id": "wh_abc123",
  "url": "https://example.com/webhooks/logline",
  "events": ["run_state.*", "registry.person.*", "deploy.*"],
  "created_at": "2025-11-17T17:00:00Z"
}
```

### 7.2 Webhook Delivery (What Your Endpoint Receives)

**Request to Your Webhook:**

```http
POST https://example.com/webhooks/logline
Content-Type: application/json
X-LogLine-Signature: ed25519:SGVsbG8gV29ybGQ=
X-LogLine-Timestamp: 2025-11-17T17:05:00Z
X-LogLine-Event: run_state.running
```

```json
{
  "event": "run_state.running",
  "span": {
    "schema_version": "1.1.0",
    "entity_type": "function",
    "this": "urn:span:f01e8400-e29b-41d4-a716-446655440050",
    "intent": "run_state.running",
    "subject": "urn:minicontainer:vv/web",
    "payload": {
      "kind": "run_state/running.v1",
      "run_id": "run_xyz789",
      "started_at": "2025-11-17T17:05:00Z",
      "node": "railway-node-42"
    },
    "ts": "2025-11-17T17:05:00Z"
  },
  "delivered_at": "2025-11-17T17:05:00.234Z"
}
```

---

## 8. Error Handling Examples

### 8.1 Unauthorized (401)

**Request (missing or invalid API key):**

```http
POST /v1/spans
Authorization: ApiKey invalid_key
```

**Response:**

```json
{
  "code": "UNAUTHENTICATED",
  "message": "Invalid API key",
  "trace_id": "j01k2345-l678-3mn0-9012-345678ijklmn"
}
```

### 8.2 Forbidden (403)

**Request (valid key but insufficient permissions):**

**Response:**

```json
{
  "code": "FORBIDDEN",
  "message": "Tenant 'voulezvous' is not allowed to execute intent 'law.create'",
  "trace_id": "k12l3456-m789-4no0-0123-456789jklmno",
  "details": {
    "tenant": "voulezvous",
    "intent": "law.create",
    "required_permission": "admin:law:write"
  }
}
```

### 8.3 Rate Limited (429)

**Response:**

```json
{
  "code": "RATE_LIMITED",
  "message": "Rate limit exceeded for intent 'deploy.apply': 10 requests per minute",
  "trace_id": "l23m4567-n890-5op0-1234-567890klmnop",
  "details": {
    "limit": "10 req/min",
    "retry_after_seconds": 45
  }
}
```

### 8.4 Nonce Reused (409)

**Response:**

```json
{
  "code": "NONCE_REUSED",
  "message": "Nonce has already been used: 7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "trace_id": "m34n5678-o901-6pq0-2345-678901lmnopq",
  "details": {
    "nonce": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "first_used_at": "2025-11-17T12:34:56Z"
  }
}
```

### 8.5 Idempotency Conflict (409)

**Response:**

```json
{
  "code": "IDEMPOTENCY_CONFLICT",
  "message": "Idempotency key already used with different request body",
  "trace_id": "n45o6789-p012-7qr0-3456-789012mnopqr",
  "details": {
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440001",
    "original_request_hash": "blake3:original...",
    "current_request_hash": "blake3:current...",
    "original_receipt": {
      "this": "urn:span:550e8400-e29b-41d4-a716-446655440001",
      "hash": "blake3:abcd...",
      "ts": "2025-11-17T12:34:56.123Z"
    }
  }
}
```

---

## 9. Mini Suite Examples

### 9.1 miniPrompt - Compile & Run

**Request (Compile):**

```http
POST /v1/prompt/compile
```

```json
{
  "kernel": "# Greeting Prompt\n\nHello, {{name}}! Welcome to {{product}}."
}
```

**Response:**

```json
{
  "hash": "blake3:kernel_hash_abc123...",
  "normalized": "# Greeting Prompt\n\nHello, {{name}}! Welcome to {{product}}."
}
```

**Request (Run):**

```http
POST /v1/prompt/run
```

```json
{
  "kernel": "# Greeting Prompt\n\nHello, {{name}}! Welcome to {{product}}.",
  "vars": {
    "name": "Joana",
    "product": "LogLine"
  }
}
```

**Response:**

```json
{
  "result": "# Greeting Prompt\n\nHello, Joana! Welcome to LogLine.",
  "hash": "blake3:result_hash_xyz789..."
}
```

### 9.2 miniMemory - Put, Search

**Request (Put):**

```http
POST /v1/memory/put
```

```json
{
  "key": "user_prefs_joana",
  "value": {
    "theme": "dark",
    "language": "pt",
    "notifications": true
  }
}
```

**Response:**

```json
{
  "key": "user_prefs_joana",
  "hash": "blake3:memory_hash..."
}
```

**Request (Search):**

```http
POST /v1/memory/search
```

```json
{
  "text": "dark theme preferences",
  "topk": 5
}
```

**Response:**

```json
{
  "results": [
    {
      "key": "user_prefs_joana",
      "value": {
        "theme": "dark",
        "language": "pt",
        "notifications": true
      },
      "score": 0.92
    },
    {
      "key": "user_prefs_pedro",
      "value": {
        "theme": "dark",
        "language": "en"
      },
      "score": 0.78
    }
  ]
}
```

### 9.3 miniWorkflows - Start & Step

**Request (Start):**

```http
POST /v1/workflows/start
```

```json
{
  "flow": "order-approval",
  "input": {
    "order_id": "ord_12345",
    "amount": 1500.00,
    "customer": "joana@example.com"
  }
}
```

**Response:**

```json
{
  "id": "wf_abc123",
  "flow": "order-approval",
  "status": "running",
  "current_step": "risk_check",
  "created_at": "2025-11-17T18:00:00Z",
  "updated_at": "2025-11-17T18:00:00Z"
}
```

**Request (Step):**

```http
POST /v1/workflows/step
```

```json
{
  "id": "wf_abc123",
  "step": "approve",
  "input": {
    "approved": true,
    "approver": "manager@example.com"
  }
}
```

**Response:**

```json
{
  "id": "wf_abc123",
  "flow": "order-approval",
  "status": "completed",
  "current_step": "complete",
  "created_at": "2025-11-17T18:00:00Z",
  "updated_at": "2025-11-17T18:05:30Z"
}
```

### 9.4 miniTools - Invoke HTTP Tool

**Request:**

```http
POST /v1/tools/invoke
```

```json
{
  "tool": "http",
  "args": {
    "url": "https://api.example.com/webhook",
    "method": "POST",
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "event": "order.created",
      "order_id": "ord_12345"
    }
  }
}
```

**Response:**

```json
{
  "result": {
    "status": 200,
    "body": {
      "received": true,
      "event_id": "evt_xyz789"
    },
    "headers": {
      "content-type": "application/json"
    }
  }
}
```

### 9.5 miniContainer - Wallet Sign & Verify

**Request (Sign):**

```http
POST /v1/container/wallet/sign
```

```json
{
  "kid": "kid_sign_1",
  "payload": {
    "action": "deploy.apply",
    "target": "urn:minicontainer:vv/web",
    "timestamp": "2025-11-17T19:00:00Z"
  }
}
```

**Response:**

```json
{
  "signature": "U2lnbmVkIHBheWxvYWQgd2l0aCBFZDI1NTE5",
  "kid": "kid_sign_1",
  "hash": "blake3:payload_hash_abc..."
}
```

**Request (Verify):**

```http
POST /v1/container/wallet/verify
```

```json
{
  "payload": {
    "action": "deploy.apply",
    "target": "urn:minicontainer:vv/web",
    "timestamp": "2025-11-17T19:00:00Z"
  },
  "signature": "U2lnbmVkIHBheWxvYWQgd2l0aCBFZDI1NTE5",
  "kid": "kid_sign_1"
}
```

**Response:**

```json
{
  "valid": true
}
```

### 9.6 miniContratos - Register Contract

**Request:**

```http
POST /v1/contratos/register
```

```json
{
  "template": "nda@v3",
  "parties": [
    {
      "name": "Empresa A",
      "role": "disclosing_party"
    },
    {
      "name": "Empresa B",
      "role": "receiving_party"
    }
  ],
  "effective_date": "2025-11-20",
  "jurisdiction": "PT"
}
```

**Response:**

```json
{
  "id": "ctr_nda_20251117_001",
  "template": "nda@v3",
  "parties": [
    {
      "name": "Empresa A",
      "role": "disclosing_party"
    },
    {
      "name": "Empresa B",
      "role": "receiving_party"
    }
  ],
  "status": "draft",
  "created_at": "2025-11-17T19:30:00Z"
}
```

---

## 10. Batch Operations

### 10.1 Batch Append (Multiple Spans)

**Request:**

```http
POST /v1/spans
```

```json
[
  {
    "schema_version": "1.1.0",
    "entity_type": "contract",
    "this": "urn:span:batch-001",
    "did": {"actor": "usr_abc", "action": "append"},
    "intent": "registry.person.create",
    "subject": "urn:registry:person/alice",
    "payload": {
      "kind": "registry/person.v1",
      "name": "Alice",
      "email": "alice@example.com",
      "country": "US",
      "consent": true
    },
    "ts": "2025-11-17T20:00:00Z"
  },
  {
    "schema_version": "1.1.0",
    "entity_type": "contract",
    "this": "urn:span:batch-002",
    "did": {"actor": "usr_abc", "action": "append"},
    "intent": "registry.person.create",
    "subject": "urn:registry:person/bob",
    "payload": {
      "kind": "registry/person.v1",
      "name": "Bob",
      "email": "bob@example.com",
      "country": "GB",
      "consent": true
    },
    "ts": "2025-11-17T20:00:01Z"
  }
]
```

**Response (Array of ACKs):**

```json
[
  {
    "status": "ok",
    "this": "urn:span:batch-001",
    "trace_id": "trace-001",
    "receipt": {
      "ts": "2025-11-17T20:00:00.123Z",
      "hash": "blake3:alice_hash...",
      "signature": "YWxpY2Ugc2lnbmF0dXJl"
    }
  },
  {
    "status": "ok",
    "this": "urn:span:batch-002",
    "trace_id": "trace-002",
    "receipt": {
      "ts": "2025-11-17T20:00:01.234Z",
      "hash": "blake3:bob_hash...",
      "signature": "Ym9iIHNpZ25hdHVyZQ=="
    }
  }
]
```

---

**End of Examples Document**

For implementation guides, see:
- `openapi-specification.yaml` for complete API reference
- `TDLN-Grammar-Specification.md` for parsing and validation logic
- `span-schemas-catalog.yaml` for all intent schemas
