LogLine — TDLN + LLM Council Pack (Mini & Max)

Resumo: O TDLN é o único decisor. Um LLM Conselheiro (local) e um LLM Premium (opcional, escolhido pelo cliente) atuam como assistentes de redação/garantia de qualidade. Há dois sabores do TDLN: Mini (browser) e Max (Rust). Este pack é obrigatório para a Universal API e para qualquer produto LogLine que aceite linguagem natural.

⸻

1) Objetivos
	•	Determinismo de entrada: toda requisição vira span válido; respostas com recibo e trilha.
	•	Consistência: NL → (LLM aconselha) → TDLN decide → span perfeito.
	•	Privacidade & Custo: default LLM Conselheiro local; Premium apenas quando necessário.
	•	Portabilidade: TDLN Mini roda no browser/edge; TDLN Max em Rust (server/CLI/daemon) com máxima robustez.

⸻

2) Arquitetura (alto nível)

Entrada (parágrafo, span incompleto ou span perfeito)
   │
   ├─► (Opcional) LLM Conselheiro Local  ← cache, regras, prompts LogLine
   │        └─ propõe: candidato de span + diagnóstico + campos faltantes
   │
   ├─► (Opcional) LLM Premium (fallback)  ← só se confiança baixa / tarefa difícil
   │        └─ propõe: candidato de span + justificativa
   │
   └─► TDLN (Mini/Max) — **decide/valida** gramática, tipos, políticas e
           emite o span final + recibo + motivos de correção/recusa

Regra de ouro: Quem decide é o TDLN. Os LLMs aconselham e geram rascunhos.

⸻

3) Componentes

3.1 TDLN Mini (browser)
	•	WebAssembly/JS, roda 100% cliente; ideal para UIs, IDEs e flows de baixa latência.
	•	Entradas: parágrafo ou span; Saídas: compile(span), need_more(fields), invalid(errors).
	•	Proofs leves (hash + schema OK). Pode assinar localmente via Web Crypto.

3.2 TDLN Max (Rust)
	•	Binário/daemon; API FFI/HTTP; throughput alto, logs estruturados, política rígida.
	•	Provas completas: BLAKE3 + Ed25519, trilha determinística, limites fortes.
	•	Integração com ledger NDJSON e policies (allowlist, quotas, intents).

3.3 LLM Conselheiro Local (LLM1)
	•	Objetivo: assistir, não decidir. Enriquecer o NL, sugerir campos, normalizar termos.
	•	Requisitos: rápido, barato, open-weight, bom em instrução e preenchimento de campos.
	•	Sugestões (exemplos): Llama 3.1 (8B/70B Instruct), Qwen2.5 (7B/14B Instruct), Mixtral 8x7B Instruct.
	•	Servir via vLLM (OpenAI-compatible), com suporte a quantização e LoRA.

3.4 LLM Premium (LLM2)
	•	Escolha do cliente (ex.: OpenAI/Anthropic/Google). Rota apenas quando confidence < τ ou tarefa específica.
	•	Logs guardam: modelo, versão, prompt-hash, rationale resumido (sem dados sensíveis).

⸻

4) Fluxo de Decisão (formal)
	1.	Ingress: payload { paragraph? , span? , context } + metadados (tenant, ambiente, traceId).
	2.	Fast-path: se span perfeito → vai direto ao TDLN para validação e recibo.
	3.	Counsel: se NL ou span parcial → chamar LLM1 com prompt canônico (ver §7) → draft + missing[] + confidence.
	4.	Fallback: se confidence < τ ou missing crítico → opcional LLM2 (premium) → draft* + justificativa.
	5.	Decisão: TDLN valida gramática, tipos, políticas e decide:
	•	ok(span_final), ou need_more(missing[]), ou invalid(errors).
	6.	Recibo & Auditoria: assina/hasheia; registra no ledger; emite eventos (SSE/Webhook) e ACK.

⸻

5) Contratos (API/SDK)

5.1 Endpoints
	•	POST /v1/tdln/decide
	•	In: { paragraph?: string, span?: object, ctx?: {...}, prefer_premium?: boolean }
	•	Out: { status: "ok"|"need_more"|"invalid", span?: {...}, missing?: string[], errors?: string[], receipt: {...}, lineage: {...} }
	•	POST /v1/tdln/assist (somente aconselhamento)
	•	Out: { draft?: {...}, missing?: string[], confidence: number, advisor: {...} }
	•	POST /v1/tdln/compile (TDLN puro)
	•	Out: { status, span?, errors?, receipt }

5.2 SDK (TS)

client.tdln.decide({ paragraph, span, ctx, prefer_premium })
client.tdln.assist({ paragraph, span, ctx })
client.tdln.compile({ span })

5.3 CLI

logline tdln:assist  --text "criar cliente Joana…"  --watch
logline tdln:decide  --text "refund do pedido 123 por fraude"  --prefer-premium
logline tdln:compile --file span.json


⸻

6) Modelo de Dados — lineage

{
  "advisor": {
    "local": {"model": "qwen2.5-7b-instruct", "provider": "vllm", "quant": "awq-int4", "prompt_hash": "…", "params": {"temp": 0.2, "top_p": 0.9}},
    "premium": {"model": "gpt-x?", "provider": "openai", "prompt_hash": "…"}
  },
  "tdln": {"mode": "max|mini", "grammar": "json✯atomic@1.0", "policy_set": "default@2025-11", "proof": {"hash": "blake3:…", "sig": "ed25519:…"}}
}


⸻

7) Prompts Canônicos (LLM Conselheiro)
	•	System: “Você é conselheiro do TDLN. Sua tarefa é propor um span que passe no TDLN. Nunca decide. Liste missing[] com nomes canônicos. Respeite schema e restrições. Não invente dados sensíveis.”
	•	Few-shot: pares NL → span correto → rationale curto (apenas para auditoria humana).
	•	Tool awareness: incluir schema simplificado (campos obrigatórios, enums, formatos) e tabelas de sinônimos de domínio.

⸻

8) Fine-tuning (LLM1)
	•	Objetivo: melhorar recall/precisão de campos e normalização.
	•	Dados: TDLN-Cookbook + históricos aprovados (consentidos) + hard negatives (campos ambíguos/faltantes).
	•	Técnica: LoRA/QLoRA em 7B/8B; learning rate baixo; eval em TDLN-EvalSet (pass rate e missing@k).
	•	Entrega: checkpoints versionados (ex.: logline-tdln-counselor@YYYYMMDD.lora).

⸻

9) Roteamento & Política de Uso
	•	Default: LLM1 local (Qwen2.5 7B Instruct ou Llama 3.1 8B Instruct) via vLLM.
	•	Fallback: premium quando confidence < τ, need_more repetido, ou categoria “difícil” (ex.: jurídico, fiscal).
	•	Circuit breaker: quotas por tenant/ambiente, latência máxima, custo estimado por decisão.

⸻

10) Segurança & Privacidade
	•	TDLN decide: sem hallucination authority.
	•	Minimização: o Conselheiro recebe apenas campos necessários; mascarar PII.
	•	Local-first: preferência por LLM1 local + quantização.
	•	Provas: hash/assinatura (BLAKE3 + Ed25519) quando em modo Max; recibos assináveis.

⸻

11) Operação & Observabilidade
	•	Métricas: ok/need_more/invalid, counsel_confidence, taxa de fallback, P50/P95, custo/1000 decisões.
	•	Logs estruturados: traceId, tenant, action, advisor.model, tdln.mode, policy_set.
	•	Eventos: SSE/Webhooks; DLQ para entregas premium.

⸻

12) Roadmap de Implantação (4 semanas)
	1.	Descoberta: schemas-alvo, intents, sinônimos; seleção LLM1 (benchmark curto).
	2.	MVP: TDLN Mini/Max integrados; vLLM on-prem; prompts canônicos; SDK/CLI.
	3.	Aprimorar: LoRA Counselor; thresholds & roteamento; painéis de métricas.
	4.	Endurecer: políticas, DLQ, auditoria formal, checklist de privacidade.

⸻

13) Sugestões de Modelo (LLM1 Local)
	•	Qwen2.5 Instruct (7B/14B) — forte em instrução e sizes variados; open-weight.
	•	Llama 3.1 Instruct (8B/70B) — ampla adoção, tool-use maduro.
	•	Mixtral 8x7B Instruct — MoE eficiente, bom custo/latência.

Servir via vLLM (API compatível OpenAI; quantizações e cache), com prefix caching e multi-LoRA.

⸻

14) Integração com Universal API
	•	ANTES da API: usar o TDLN Gate (Mini/Max). A CLI e SDK falam com /v1/tdln/decide quando houver NL.
	•	SPAN Perfeito: passa reto pelo Gate (só validação/recibo).
	•	LLM Council: transparente para o cliente; configurável por tier.

⸻

15) Critérios de Aceite
	•	≥ 95% de pass rate no TDLN-EvalSet (sem premium).
	•	Fallback ≤ 10% no piloto; P95 latência ≤ 1.2s (LLM1 local quantizado).
	•	Zero spans aprovados fora de schema/policy.

⸻

16) Riscos & Mitigações
	•	Ambiguidade de domínio → cookbook e hard negatives.
	•	Drift de prompts → versionamento + testes de regressão.
	•	Custos premium → thresholds e budget guard por ambiente.

⸻

17) Anexos
	•	Contracts OpenAPI (tdln.*), exemplos de prompts, playbook de fine-tuning, guia de deploy (vLLM + quantização), e cookbook.
