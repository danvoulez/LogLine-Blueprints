# TDLN — Formal Grammar & Decision Specification v1.0

**Version:** 1.0.0
**Date:** 2025-11-17
**Status:** Complete

---

## Table of Contents

1. [Overview](#1-overview)
2. [Principles](#2-principles)
3. [Formal Grammar (EBNF)](#3-formal-grammar-ebnf)
4. [Parsing Algorithm](#4-parsing-algorithm)
5. [Normalization Rules](#5-normalization-rules)
6. [Validation Logic](#6-validation-logic)
7. [Intent Schema Catalog](#7-intent-schema-catalog)
8. [Decision Flow](#8-decision-flow)
9. [LLM Council Integration](#9-llm-council-integration)
10. [Error Handling](#10-error-handling)
11. [Examples & Test Cases](#11-examples--test-cases)
12. [Implementation Notes](#12-implementation-notes)

---

## 1. Overview

**TDLN (Textual Deterministic Ledger Notation)** is the deterministic parser and validator that sits at the gate of the LogLine Universal API. It converts natural language, partial spans, or complete spans into validated, canonical spans ready for ledger append.

### 1.1 Responsibilities

- **Parse** natural language and structured input into span candidates
- **Normalize** fields (country codes, emails, URNs, dates, etc.)
- **Validate** against intent schemas and policies
- **Decide** whether to accept (ok), request more info (need_more), or reject (invalid)
- **Produce** deterministic output (same input → same hash)

### 1.2 Key Properties

- **Deterministic**: Identical inputs produce identical outputs
- **Transparent**: All decisions explained with `why` and `suggested_reply`
- **Composable**: Works with LLM Council (advisory) or standalone
- **Versioned**: Grammar and schemas are versioned separately

---

## 2. Principles

1. **TDLN Decides**: LLMs advise, TDLN is the final arbiter
2. **Fail-Safe Defaults**: Unknown fields → need_more; ambiguous → invalid
3. **Canonical Ordering**: JSON keys sorted lexicographically
4. **Type Safety**: Strict type checking per intent schema
5. **Privacy-First**: Never log or expose secrets; use grant:// URIs
6. **Idempotency**: Same span + same nonce → same receipt

---

## 3. Formal Grammar (EBNF)

### 3.1 Core Span Grammar

```ebnf
(* TDLN Span Grammar v1.0 *)

Span            = "{" SpanFields "}" ;
SpanFields      = SchemaVersion "," EntityType "," This ","
                  Prev? "," Did "," Intent "," Subject ","
                  Input? "," Payload? "," Output? ","
                  Timestamp "," Signature? ;

SchemaVersion   = "\"schema_version\"" ":" "\"" Version "\"" ;
Version         = Digit+ "." Digit+ "." Digit+ ;

EntityType      = "\"entity_type\"" ":" "\""
                  ("contract" | "function" | "decision" | "agent" | "law" | "file" | "test")
                  "\"" ;

This            = "\"this\"" ":" "\"" URN "\"" ;
Prev            = "\"prev\"" ":" "\"" URN "\"" ;

URN             = "urn:" Segment (":" Segment)* ("/" Segment)* ;
Segment         = (Letter | Digit | "-" | "_" | ".")+ ;

Did             = "\"did\"" ":" "{" "\"actor\"" ":" String ","
                                   "\"action\"" ":" "\"append\"" "}" ;

Intent          = "\"intent\"" ":" "\"" IntentPath "\"" ;
IntentPath      = IntentSegment ("." IntentSegment)* ;
IntentSegment   = Letter (Letter | Digit | "_")* ;

Subject         = "\"subject\"" ":" "\"" URN "\"" ;

Input           = "\"input\"" ":" "{" InputFields? "}" ;
InputFields     = (Args | Env | Content | BytesB64) ("," (Args | Env | Content | BytesB64))* ;
Args            = "\"args\"" ":" "[" (JSON ("," JSON)*)? "]" ;
Env             = "\"env\"" ":" "{" (KeyValue ("," KeyValue)*)? "}" ;
Content         = "\"content\"" ":" String ;
BytesB64        = "\"bytes_b64\"" ":" String ;

Payload         = "\"payload\"" ":" JSON ;

Output          = "\"output\"" ":" "{" OutputFields? "}" ;
OutputFields    = (Stdout | Stderr | ExitCode) ("," (Stdout | Stderr | ExitCode))* ;
Stdout          = "\"stdout\"" ":" String ;
Stderr          = "\"stderr\"" ":" String ;
ExitCode        = "\"exit_code\"" ":" Integer ;

Timestamp       = "\"ts\"" ":" "\"" ISO8601 "\"" ;
ISO8601         = (* RFC 3339 date-time *) ;

Signature       = "\"signature\"" ":" "{"
                    "\"kid\"" ":" String ","
                    "\"alg\"" ":" "\"ed25519-blake3\"" ","
                    "\"sig\"" ":" String
                  "}" ;

KeyValue        = String ":" JSON ;
String          = "\"" Character* "\"" ;
Integer         = ["-"] Digit+ ;
JSON            = (* Valid JSON value *) ;
Letter          = "a".."z" | "A".."Z" ;
Digit           = "0".."9" ;
Character       = (* Any Unicode character except unescaped " or \ *) ;
```

### 3.2 Natural Language Input Grammar

```ebnf
(* NL Input Grammar *)

NLInput         = Sentence+ ;
Sentence        = Clause ("," Clause)* ("." | ";" | "!") ;
Clause          = Subject Predicate Object? ;
Subject         = Noun | Pronoun | ProperNoun ;
Predicate       = Verb ;
Object          = Noun | ProperNoun | Value ;

(* Intent extraction patterns *)
IntentKeyword   = "criar" | "create" | "cadastrar" | "register" |
                  "deploy" | "subir" | "apply" | "executar" | "run" |
                  "assinar" | "sign" | "verificar" | "verify" ;

EntityKeyword   = "pessoa" | "person" | "cliente" | "customer" |
                  "organização" | "organization" | "org" |
                  "contrato" | "contract" | "acordo" | "agreement" |
                  "container" | "minicontainer" | "app" ;

(* Field extraction patterns *)
FieldPattern    = FieldName ":" FieldValue |
                  FieldName "=" FieldValue |
                  FieldName FieldValue ;

FieldName       = "email" | "e-mail" | "mail" |
                  "país" | "country" | "pais" |
                  "nome" | "name" | "nome completo" | "full name" |
                  "consent" | "consentimento" | "aceite" ;

FieldValue      = Email | CountryCode | Name | Boolean | Date | URI ;

Email           = LocalPart "@" Domain ;
CountryCode     = Letter Letter ;  (* ISO-3166 alpha-2 *)
Name            = Word (" " Word)* ;
Boolean         = "true" | "false" | "yes" | "no" | "sim" | "não" | "ok" ;
Date            = (* ISO-8601 date or natural language date *) ;
URI             = (* RFC 3986 URI *) ;
```

---

## 4. Parsing Algorithm

### 4.1 Parse Pipeline

```
Input (NL | Partial Span | Complete Span)
  │
  ├─► Tokenize & Normalize
  │     ├─ NL: Extract keywords, entities, fields
  │     ├─ Partial: Parse JSON, identify missing fields
  │     └─ Complete: Parse JSON, validate structure
  │
  ├─► Intent Detection
  │     ├─ Match keywords to intent catalog
  │     ├─ Use LLM Council (optional) for ambiguous cases
  │     └─ Assign canonical intent path
  │
  ├─► Field Extraction
  │     ├─ Map extracted values to intent schema
  │     ├─ Normalize (country codes, emails, dates, URNs)
  │     └─ Mark missing required fields
  │
  ├─► Schema Validation
  │     ├─ Load intent schema
  │     ├─ Type check all fields
  │     ├─ Validate constraints (enums, patterns, ranges)
  │     └─ Check required fields
  │
  └─► Decision
        ├─ All valid → ok + canonical span
        ├─ Missing required → need_more + missing[] + why
        └─ Invalid values → invalid + errors[] + suggested_reply
```

### 4.2 Pseudocode

```python
def parse(input: Union[str, dict]) -> ParseResult:
    """
    Parse input into validated span.

    Returns:
        ParseResult with status: ok | need_more | invalid
    """
    # 1. Determine input type
    if isinstance(input, str):
        tokens = tokenize_nl(input)
        intent = detect_intent(tokens)
        fields = extract_fields(tokens, intent)
    elif is_partial_span(input):
        intent = input.get('intent')
        fields = extract_span_fields(input)
    else:  # complete span
        intent = input['intent']
        fields = input
        return validate_complete_span(fields)

    # 2. Load intent schema
    schema = get_intent_schema(intent)
    if not schema:
        return ParseResult(
            status='invalid',
            errors=[f'Unknown intent: {intent}'],
            suggested_reply='Please use a valid intent'
        )

    # 3. Normalize fields
    normalized = normalize_fields(fields, schema)

    # 4. Validate
    validation = validate_against_schema(normalized, schema)

    # 5. Decide
    if validation.is_complete and validation.is_valid:
        canonical_span = canonicalize(normalized)
        return ParseResult(
            status='ok',
            span=canonical_span,
            hash=blake3(canonical_span)
        )
    elif validation.missing_required:
        return ParseResult(
            status='need_more',
            missing_fields=validation.missing_required,
            why=explain_missing(validation.missing_required, schema),
            suggested_reply=generate_prompt(validation.missing_required)
        )
    else:
        return ParseResult(
            status='invalid',
            errors=validation.errors,
            suggested_reply=suggest_fix(validation.errors)
        )
```

---

## 5. Normalization Rules

### 5.1 Field Normalizations

| Field Type | Normalization Rule | Example |
|------------|-------------------|---------|
| Email | Lowercase, trim | `Joana@EXAMPLE.com` → `joana@example.com` |
| Country | ISO-3166 alpha-2 uppercase | `pt`, `portugal`, `PT` → `PT` |
| Name | Title case, trim, collapse whitespace | `  joana  maria  ` → `Joana Maria` |
| Date | ISO-8601 UTC | `17/11/2025` → `2025-11-17T00:00:00Z` |
| URN Subject | Lowercase, slugify | `Joana Maria Silva` → `urn:registry:person/joana_m_silva` |
| Boolean | Canonical true/false | `yes`, `sim`, `ok`, `1` → `true` |
| Phone | E.164 format | `+351 91 234 5678` → `+351912345678` |
| URL | Encode, normalize | `example.com/path` → `https://example.com/path` |

### 5.2 URN Subject Generation

```python
def generate_subject_urn(intent: str, fields: dict) -> str:
    """
    Generate canonical subject URN from intent and fields.

    Examples:
        registry.person.create + {name: "Joana Maria Silva"}
          → urn:registry:person/joana_m_silva

        deploy.apply + {ref: "mc:vv/web@blake3:abc"}
          → urn:minicontainer:vv/web
    """
    if intent.startswith('registry.person'):
        name = fields.get('name', '')
        slug = slugify(name)  # joana_m_silva
        return f'urn:registry:person/{slug}'

    elif intent.startswith('deploy'):
        ref = fields.get('ref', '')
        # Parse mc:tenant/name@version
        match = re.match(r'mc:([^/]+)/([^@]+)', ref)
        if match:
            tenant, name = match.groups()
            return f'urn:minicontainer:{tenant}/{name}'

    elif intent.startswith('minicontainer'):
        manifest = fields.get('manifest', {})
        owner = manifest.get('owner', {})
        tenant = owner.get('tenant', 'unknown')
        name = fields.get('name', 'unnamed')
        return f'urn:minicontainer:{tenant}/{name}'

    # Fallback
    return f'urn:unknown:{uuid4()}'

def slugify(text: str) -> str:
    """Convert text to URL-safe slug."""
    # Joana Maria Silva → joana_m_silva
    text = text.lower().strip()
    words = text.split()
    if len(words) <= 2:
        return '_'.join(words)
    # First + middle initials + last
    return f"{words[0]}_{'_'.join(w[0] for w in words[1:-1])}_{words[-1]}"
```

### 5.3 Canonical JSON Ordering

```python
def canonicalize(span: dict) -> bytes:
    """
    Produce canonical JSON representation.

    Rules:
    - Keys sorted lexicographically
    - No extra whitespace
    - UTF-8 encoding
    - Consistent float representation
    """
    import json

    def sort_keys(obj):
        if isinstance(obj, dict):
            return {k: sort_keys(v) for k, v in sorted(obj.items())}
        elif isinstance(obj, list):
            return [sort_keys(item) for item in obj]
        return obj

    canonical = sort_keys(span)
    return json.dumps(
        canonical,
        ensure_ascii=False,
        separators=(',', ':'),
        sort_keys=True
    ).encode('utf-8')
```

---

## 6. Validation Logic

### 6.1 Type System

```typescript
type FieldType =
  | { type: 'string'; pattern?: RegExp; enum?: string[] }
  | { type: 'integer'; min?: number; max?: number }
  | { type: 'number'; min?: number; max?: number }
  | { type: 'boolean' }
  | { type: 'date'; format: 'iso8601' | 'unix' }
  | { type: 'email' }
  | { type: 'country'; format: 'iso3166-alpha2' }
  | { type: 'urn'; pattern?: string }
  | { type: 'object'; schema: Schema }
  | { type: 'array'; items: FieldType; minItems?: number; maxItems?: number };

type Schema = {
  [field: string]: {
    type: FieldType;
    required: boolean;
    description?: string;
    normalize?: (value: any) => any;
    validate?: (value: any) => boolean | string;  // true or error message
  }
};
```

### 6.2 Validation Rules

```python
def validate_field(value: any, field_type: FieldType) -> ValidationResult:
    """Validate single field against type definition."""

    if field_type['type'] == 'string':
        if not isinstance(value, str):
            return ValidationResult(valid=False, error=f'Expected string, got {type(value)}')
        if 'pattern' in field_type and not re.match(field_type['pattern'], value):
            return ValidationResult(valid=False, error=f'Does not match pattern: {field_type["pattern"]}')
        if 'enum' in field_type and value not in field_type['enum']:
            return ValidationResult(valid=False, error=f'Must be one of: {field_type["enum"]}')
        return ValidationResult(valid=True)

    elif field_type['type'] == 'email':
        if not is_valid_email(value):
            return ValidationResult(valid=False, error='Invalid email format')
        return ValidationResult(valid=True)

    elif field_type['type'] == 'country':
        if value not in ISO_3166_ALPHA2_CODES:
            return ValidationResult(valid=False, error=f'Invalid country code: {value}. Use ISO-3166 alpha-2')
        return ValidationResult(valid=True)

    elif field_type['type'] == 'boolean':
        if not isinstance(value, bool):
            return ValidationResult(valid=False, error=f'Expected boolean, got {type(value)}')
        return ValidationResult(valid=True)

    elif field_type['type'] == 'integer':
        if not isinstance(value, int):
            return ValidationResult(valid=False, error=f'Expected integer, got {type(value)}')
        if 'min' in field_type and value < field_type['min']:
            return ValidationResult(valid=False, error=f'Must be >= {field_type["min"]}')
        if 'max' in field_type and value > field_type['max']:
            return ValidationResult(valid=False, error=f'Must be <= {field_type["max"]}')
        return ValidationResult(valid=True)

    # ... other types
```

### 6.3 Cross-Field Validation

```python
def validate_cross_field(span: dict, intent_schema: Schema) -> list[str]:
    """
    Validate constraints that span multiple fields.

    Examples:
    - If consent=false, cannot proceed with person.create
    - If deploy.target.provider=railway, must have railway-specific fields
    """
    errors = []
    intent = span.get('intent', '')

    if intent == 'registry.person.create':
        payload = span.get('payload', {})
        if not payload.get('consent'):
            errors.append('Consent must be true for person registration')

        # Email required if country in EU
        if payload.get('country') in EU_COUNTRIES and not payload.get('email'):
            errors.append('Email required for EU registrations (GDPR)')

    elif intent == 'deploy.apply':
        payload = span.get('payload', {})
        target = payload.get('target', {})
        provider = target.get('provider')

        if provider == 'railway':
            if not target.get('project_id') or not target.get('service'):
                errors.append('Railway deploys require project_id and service')

        elif provider == 'podman':
            if not target.get('socket'):
                errors.append('Podman deploys require socket path')

    return errors
```

---

## 7. Intent Schema Catalog

### 7.1 Registry Intents

#### `registry.person.create`

```yaml
intent: registry.person.create
entity_type: contract
description: Register a new person in the registry

fields:
  name:
    type: string
    required: true
    normalize: title_case_trim
    description: Full name of the person

  email:
    type: email
    required: true
    normalize: lowercase_trim
    description: Email address

  country:
    type: country
    format: iso3166-alpha2
    required: true
    normalize: uppercase
    description: Country code (ISO-3166 alpha-2)

  consent:
    type: boolean
    required: true
    description: Privacy consent flag
    validate: must_be_true

  phone:
    type: string
    required: false
    pattern: '^\+[1-9]\d{1,14}$'
    normalize: e164
    description: Phone number in E.164 format

  tax_id:
    type: string
    required: false
    description: Tax identification number

payload_kind: registry/person.v1

example:
  intent: registry.person.create
  subject: urn:registry:person/joana_m_silva
  payload:
    kind: registry/person.v1
    name: Joana Maria Silva
    email: joana@example.com
    country: PT
    consent: true
```

#### `registry.org.create`

```yaml
intent: registry.org.create
entity_type: contract

fields:
  name:
    type: string
    required: true

  legal_name:
    type: string
    required: false

  country:
    type: country
    format: iso3166-alpha2
    required: true

  tax_id:
    type: string
    required: true

  industry:
    type: string
    enum: [technology, finance, healthcare, education, retail, other]
    required: false

payload_kind: registry/org.v1
```

### 7.2 Deploy Intents

#### `deploy.apply`

```yaml
intent: deploy.apply
entity_type: function
description: Deploy a minicontainer to a target provider

fields:
  ref:
    type: string
    pattern: '^mc:[a-z0-9_-]+/[a-z0-9_-]+@blake3:[a-f0-9]{64}$'
    required: true
    description: MiniContainer reference with content hash

  target:
    type: object
    required: true
    schema:
      provider:
        type: string
        enum: [railway, podman, docker, fly]
        required: true

      project_id:
        type: string
        required: true  # for railway, fly

      service:
        type: string
        required: true  # for railway

      socket:
        type: string
        required: false  # for podman

  options:
    type: object
    required: false
    schema:
      auto_scale:
        type: boolean

      replicas:
        type: integer
        min: 1
        max: 100

payload_kind: deploy/apply.v1

example:
  intent: deploy.apply
  subject: urn:minicontainer:vv/web
  payload:
    kind: deploy/apply.v1
    ref: mc:vv/web@blake3:abcd1234...
    target:
      provider: railway
      project_id: rw_proj_123
      service: web
    options:
      auto_scale: true
      replicas: 3
```

### 7.3 Wallet Intents

#### `wallet.proof.submit`

```yaml
intent: wallet.proof.submit
entity_type: decision
description: Submit proof of authorization

fields:
  proof:
    type: object
    required: true
    schema:
      type:
        type: string
        enum: [ll.proof.v1]
        required: true

      intent:
        type: string
        required: true
        description: Intent being authorized (e.g., deploy.allow)

      result:
        type: string
        enum: [allow, deny]
        required: true

      kid:
        type: string
        required: true

      sig:
        type: string
        format: base64
        required: true

      ts:
        type: date
        format: iso8601
        required: true

      nonce:
        type: string
        format: uuid
        required: true

payload_kind: wallet/proof.v1
```

### 7.4 MiniContainer Intents

#### `minicontainer.apply`

```yaml
intent: minicontainer.apply
entity_type: contract
description: Apply/update minicontainer manifest

fields:
  manifest:
    type: object
    required: true
    description: Full minicontainer manifest (see minicontainer.md)
    # Detailed schema validation done separately

payload_kind: minicontainer/manifest.v1
```

### 7.5 TDLN Meta Intents

#### `tdln.compile`

```yaml
intent: tdln.compile
entity_type: function
description: Compile/validate span without append

fields:
  span:
    type: object
    required: true
    description: Span to validate

# Returns compiled span or validation errors
```

---

## 8. Decision Flow

### 8.1 Decision Tree

```
Input
  │
  ├─ Is Complete Span?
  │   ├─ Yes → Validate Structure
  │   │         ├─ Valid → Load Intent Schema
  │   │         └─ Invalid → Return invalid (structural)
  │   │
  │   └─ No → Detect Intent
  │           ├─ Clear → Extract Fields
  │           └─ Ambiguous → Call LLM Council (optional)
  │                         ├─ Resolved → Extract Fields
  │                         └─ Still ambiguous → Return invalid (ambiguous)
  │
  ├─ Load Intent Schema
  │   ├─ Found → Proceed
  │   └─ Not Found → Return invalid (unknown intent)
  │
  ├─ Normalize Fields
  │   └─ Apply normalization rules
  │
  ├─ Validate Fields
  │   ├─ Type Check
  │   ├─ Constraint Check
  │   ├─ Cross-Field Check
  │   └─ Collect errors/missing
  │
  └─ Decide
      ├─ Complete & Valid → ok
      ├─ Missing Required → need_more
      └─ Has Errors → invalid
```

### 8.2 Decision Output Format

```typescript
type DecisionResult =
  | {
      status: 'ok';
      span: Span;
      this: string;
      hash: string;
      receipt?: Receipt;
    }
  | {
      status: 'need_more';
      missing_fields: string[];
      why: Record<string, string>;
      suggested_reply: string;
      partial_span?: Span;
    }
  | {
      status: 'invalid';
      errors: string[];
      suggested_reply: string;
      attempted_span?: Span;
    };
```

---

## 9. LLM Council Integration

### 9.1 Council Architecture

```
NL Input → LLM Counselor (local)
              ↓
          Draft Span + Confidence
              ↓
         Confidence < τ?
              ↓ Yes
         LLM Premium (optional)
              ↓
          Draft Span* + Rationale
              ↓
         TDLN Decides
              ↓
          Final Result
```

### 9.2 Counselor Prompt Template

```markdown
# System Prompt (LLM Counselor)

You are a TDLN Counselor. Your task is to propose a valid span for the TDLN parser.

RULES:
1. You ADVISE, you do NOT decide
2. List missing fields with canonical names
3. Respect schema constraints
4. Do NOT invent sensitive data (emails, phones, IDs)
5. Normalize values according to TDLN rules

## Intent: {intent}

## Schema:
{schema_yaml}

## Input:
{input_text}

## Output Format:
```json
{
  "draft_span": { ... },
  "missing": ["field1", "field2"],
  "confidence": 0.85,
  "rationale": "Brief explanation"
}
```

## Few-Shot Examples:
{few_shot_examples}
```

### 9.3 Confidence Thresholds

```python
CONFIDENCE_THRESHOLDS = {
    'high': 0.9,      # Accept immediately
    'medium': 0.7,    # TDLN validates, no premium
    'low': 0.5,       # Consider premium LLM
    'reject': 0.3,    # Too ambiguous, return need_more
}

def should_call_premium(confidence: float, intent: str) -> bool:
    """Determine if premium LLM is warranted."""
    if confidence >= CONFIDENCE_THRESHOLDS['medium']:
        return False  # Local counselor is sufficient

    if confidence < CONFIDENCE_THRESHOLDS['reject']:
        return False  # Too ambiguous, ask user directly

    # Medium-low confidence: check if high-stakes intent
    high_stakes = ['deploy.apply', 'wallet.proof.submit', 'law.create']
    return intent in high_stakes
```

### 9.4 Lineage Tracking

Every TDLN decision includes lineage:

```json
{
  "lineage": {
    "advisor": {
      "local": {
        "model": "qwen2.5-7b-instruct",
        "provider": "vllm",
        "quant": "awq-int4",
        "prompt_hash": "blake3:abc...",
        "params": {"temp": 0.2, "top_p": 0.9},
        "confidence": 0.85
      },
      "premium": null
    },
    "tdln": {
      "mode": "max",
      "grammar_version": "1.0.0",
      "policy_set": "default@2025-11",
      "proof": {
        "hash": "blake3:def...",
        "sig": "ed25519:ghi..."
      }
    }
  }
}
```

---

## 10. Error Handling

### 10.1 Error Categories

| Category | Code | Description | User Action |
|----------|------|-------------|-------------|
| Structural | `INVALID_STRUCTURE` | JSON parsing failed | Fix JSON syntax |
| Intent | `UNKNOWN_INTENT` | Intent not in catalog | Use valid intent |
| Missing | `MISSING_REQUIRED` | Required fields absent | Provide missing fields |
| Type | `TYPE_MISMATCH` | Field type incorrect | Fix field type |
| Constraint | `CONSTRAINT_VIOLATION` | Value out of range/pattern | Fix value |
| Cross-Field | `CROSS_FIELD_ERROR` | Multi-field constraint failed | Fix related fields |
| Ambiguous | `AMBIGUOUS_INPUT` | Cannot determine intent | Clarify input |
| Policy | `POLICY_VIOLATION` | Tenant/rate/quota policy | Check policy limits |

### 10.2 Error Messages

```python
ERROR_TEMPLATES = {
    'INVALID_EMAIL': 'Invalid email format: {value}. Expected: user@domain.com',
    'INVALID_COUNTRY': 'Invalid country code: {value}. Use ISO-3166 alpha-2 (e.g., PT, US, BR)',
    'MISSING_CONSENT': 'Consent is required for person registration',
    'UNKNOWN_INTENT': 'Unknown intent: {intent}. Valid intents: {valid_intents}',
    'TYPE_MISMATCH': 'Field {field} expects {expected_type}, got {actual_type}',
}

def format_error(code: str, **kwargs) -> str:
    """Format error message with context."""
    return ERROR_TEMPLATES.get(code, 'Unknown error').format(**kwargs)
```

### 10.3 Suggested Replies

```python
def generate_suggested_reply(result: ParseResult) -> str:
    """Generate helpful suggestion for user."""

    if result.status == 'need_more':
        missing = result.missing_fields
        why = result.why

        suggestions = []
        for field in missing:
            desc = why.get(field, f'Required field: {field}')
            suggestions.append(f'- {field}: {desc}')

        return f"To complete this request, please provide:\n" + '\n'.join(suggestions)

    elif result.status == 'invalid':
        errors = result.errors

        fixes = []
        for error in errors:
            if 'country' in error.lower():
                fixes.append('- Use ISO-3166 alpha-2 country code (e.g., PT, US, BR)')
            elif 'email' in error.lower():
                fixes.append('- Use valid email format: user@domain.com')
            else:
                fixes.append(f'- {error}')

        return f"Please fix the following:\n" + '\n'.join(fixes)

    return ''
```

---

## 11. Examples & Test Cases

### 11.1 Golden Tests

Golden tests ensure deterministic parsing: same input → same hash.

#### Test 1: Person Registration (NL → Span)

**Input (NL):**
```
Cadastrar pessoa Joana Maria Silva, email joana@example.com, país Portugal, consent ok
```

**Expected Output:**
```json
{
  "status": "ok",
  "span": {
    "schema_version": "1.1.0",
    "entity_type": "contract",
    "this": "urn:span:GENERATED",
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
    "ts": "TIMESTAMP"
  },
  "hash": "blake3:DETERMINISTIC_HASH"
}
```

**Normalization Applied:**
- Name: Title case
- Email: Lowercase
- Country: `Portugal` → `PT`
- Consent: `ok` → `true`
- Subject: `Joana Maria Silva` → `joana_m_silva`

#### Test 2: Missing Required Fields

**Input:**
```json
{
  "intent": "registry.person.create",
  "subject": "urn:registry:person/joana",
  "payload": {
    "name": "Joana"
  }
}
```

**Expected Output:**
```json
{
  "status": "need_more",
  "missing_fields": ["email", "country", "consent"],
  "why": {
    "email": "Email address is required for person registration",
    "country": "Country code (ISO-3166 alpha-2) is required",
    "consent": "Privacy consent must be explicitly provided"
  },
  "suggested_reply": "To complete this request, please provide:\n- email: Email address is required for person registration\n- country: Country code (ISO-3166 alpha-2) is required\n- consent: Privacy consent must be explicitly provided"
}
```

#### Test 3: Invalid Fields

**Input:**
```json
{
  "intent": "registry.person.create",
  "subject": "urn:registry:person/joana",
  "payload": {
    "name": "Joana",
    "email": "invalid-email",
    "country": "XX",
    "consent": true
  }
}
```

**Expected Output:**
```json
{
  "status": "invalid",
  "errors": [
    "Invalid email format: invalid-email. Expected: user@domain.com",
    "Invalid country code: XX. Use ISO-3166 alpha-2 (e.g., PT, US, BR)"
  ],
  "suggested_reply": "Please fix the following:\n- Use valid email format: user@domain.com\n- Use ISO-3166 alpha-2 country code (e.g., PT, US, BR)"
}
```

#### Test 4: Deploy (Complete Span)

**Input:**
```json
{
  "schema_version": "1.1.0",
  "entity_type": "function",
  "this": "urn:span:550e8400-e29b-41d4-a716-446655440000",
  "did": {"actor": "usr_abc", "action": "append"},
  "intent": "deploy.apply",
  "subject": "urn:minicontainer:vv/web",
  "payload": {
    "kind": "deploy/apply.v1",
    "ref": "mc:vv/web@blake3:abcd1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
    "target": {
      "provider": "railway",
      "project_id": "rw_proj_123",
      "service": "web"
    }
  },
  "ts": "2025-11-17T12:34:56Z"
}
```

**Expected Output:**
```json
{
  "status": "ok",
  "this": "urn:span:550e8400-e29b-41d4-a716-446655440000",
  "hash": "blake3:DETERMINISTIC_HASH",
  "receipt": {
    "ts": "2025-11-17T12:34:56.123Z",
    "hash": "blake3:DETERMINISTIC_HASH",
    "signature": "BASE64_SIG"
  }
}
```

### 11.2 Fuzzing Tests

Generate randomized inputs to test error handling:

```python
def fuzz_test_registry_person():
    """Generate random inputs for person registration."""
    test_cases = []

    # Valid variations
    test_cases.extend([
        {'name': random_name(), 'email': random_email(), 'country': random_country(), 'consent': True},
        {'name': random_name().upper(), 'email': random_email().upper(), 'country': random_country().lower(), 'consent': 'yes'},
    ])

    # Invalid variations
    test_cases.extend([
        {'name': '', 'email': random_email(), 'country': random_country(), 'consent': True},
        {'name': random_name(), 'email': 'not-an-email', 'country': random_country(), 'consent': True},
        {'name': random_name(), 'email': random_email(), 'country': 'INVALID', 'consent': True},
        {'name': random_name(), 'email': random_email(), 'country': random_country(), 'consent': False},
    ])

    return test_cases
```

### 11.3 Regression Tests

Track historical bugs to prevent regressions:

```yaml
# tests/regression/issue-001-country-normalization.yaml
name: Country code normalization regression
issue: "#001"
description: Ensure country names are normalized to ISO-3166 alpha-2

test_cases:
  - input:
      paragraph: "person João, email joao@example.com, country brazil, consent yes"
    expected:
      status: ok
      payload.country: BR

  - input:
      paragraph: "person Maria, email maria@example.com, country usa, consent ok"
    expected:
      status: ok
      payload.country: US
```

---

## 12. Implementation Notes

### 12.1 Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Parse latency (p50) | < 50ms | Pure TDLN, no LLM |
| Parse latency (p95) | < 200ms | Pure TDLN |
| Parse latency w/ LLM (p50) | < 800ms | Local counselor |
| Parse latency w/ Premium (p95) | < 3s | Premium LLM fallback |
| Throughput | > 1000 req/s | Single instance |
| Memory per request | < 1MB | Stateless parsing |

### 12.2 Implementation Checklist

- [ ] EBNF parser implementation (Rust or TS)
- [ ] Intent schema catalog (YAML files)
- [ ] Normalization functions library
- [ ] Validation engine with error messages
- [ ] LLM counselor integration (vLLM)
- [ ] Premium LLM fallback (OpenAI/Anthropic)
- [ ] Golden test suite (100+ cases)
- [ ] Fuzzing test harness
- [ ] Regression test suite
- [ ] Performance benchmarks
- [ ] Lineage tracking and logging
- [ ] Canonical JSON library (BLAKE3 hashing)
- [ ] Country/email/phone normalization utilities

### 12.3 TDLN Modes

#### TDLN Mini (Browser/Edge)

- **Target:** WebAssembly/JS
- **Features:**
  - Parse, normalize, validate
  - Lightweight schema validation
  - No LLM (or local ONNX model)
- **Use Cases:** CLI dry-run, IDE plugins, browser apps

#### TDLN Max (Server)

- **Target:** Rust binary or daemon
- **Features:**
  - Full validation with policies
  - LLM Council integration
  - Receipt generation with Ed25519
  - Merkle proof construction
  - High throughput
- **Use Cases:** API Gateway, production validation

### 12.4 Versioning Strategy

```
TDLN Version: {grammar}.{schemas}.{patch}
Example: 1.2.3

grammar (major): Breaking changes to grammar syntax
schemas (minor): New intents, backward-compatible schema changes
patch: Bug fixes, normalization improvements
```

Intent schemas are versioned independently:
```
registry.person.create@v1
registry.person.create@v2  (adds optional fields)
```

---

## 13. Appendix

### 13.1 ISO-3166 Alpha-2 Codes (Sample)

```python
ISO_3166_CODES = {
    'PT': 'Portugal',
    'US': 'United States',
    'BR': 'Brazil',
    'GB': 'United Kingdom',
    'DE': 'Germany',
    'FR': 'France',
    'ES': 'Spain',
    'IT': 'Italy',
    # ... complete list
}

# Reverse lookup for normalization
COUNTRY_NAME_TO_CODE = {
    'portugal': 'PT',
    'usa': 'US',
    'united states': 'US',
    'america': 'US',
    'brazil': 'BR',
    'brasil': 'BR',
    # ... complete list
}
```

### 13.2 Natural Language Patterns (Sample)

```python
NL_PATTERNS = {
    'registry.person.create': [
        r'cadastrar pessoa (.+)',
        r'create person (.+)',
        r'register person (.+)',
        r'criar cliente (.+)',
        r'new customer (.+)',
    ],
    'deploy.apply': [
        r'deploy (.+) to (.+)',
        r'subir (.+) para (.+)',
        r'aplicar deploy (.+)',
    ],
}

FIELD_PATTERNS = {
    'email': [
        r'email[:\s]+([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})',
        r'e-mail[:\s]+([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})',
    ],
    'country': [
        r'pa[ií]s[:\s]+(\w+)',
        r'country[:\s]+(\w+)',
    ],
    'consent': [
        r'consent[:\s]+(ok|yes|true|sim)',
        r'consentimento[:\s]+(ok|yes|true|sim)',
    ],
}
```

### 13.3 References

- [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) - Date and Time on the Internet
- [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986) - Uniform Resource Identifier (URI)
- [RFC 5322](https://www.rfc-editor.org/rfc/rfc5322) - Internet Message Format (Email)
- [ISO 3166-1](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) - Country Codes
- [E.164](https://en.wikipedia.org/wiki/E.164) - International Public Telecommunication Numbering Plan
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) - Cryptographic Hash Function
- [Ed25519](https://ed25519.cr.yp.to/) - Public-Key Signature System

---

**End of TDLN Grammar & Decision Specification v1.0**
