# LogLine — Governance Addendum (v1)
**Data:** 2025-11-17

Este adendo consolida práticas de governança para o pacote **Universal API + CLI + SDK**,
baseado no “best of” da LogLine Foundation (bundle extraído).

## 1) Princípios
- **Determinismo** na entrada (Gate/TDLN) e na decisão (policies declarativas).
- **Idempotência & Assinatura** em todas as operações sensíveis.
- **Separação de poderes**: API mínima, CLI complexa, SDK padronizador.
- **Observabilidade verificável**: recibos, métricas e audit log.

## 2) Políticas (Policies)
- **Allowlist por ação/tenant** (quem pode executar o quê).
- **Quotas & Rate limits** por ação, tenant e ambiente.
- **Segurança por padrão**: dados sensíveis via grant/materialização na borda.
- **Versionamento de schema** (breaking: major; compatível: minor/patch).

## 3) Chaves, Wallet & Identidade
- **ApiKey** para transporte público (dev/CI/integrações).
- **Ed25519 + BLAKE3** para assinatura e recibos (onde aplicável).
- **Wallet** do usuário/tenant como custodiante de chaves e grants.
- **Rotação** e **Key IDs** (KIDs) com escopo e validade.

## 4) Recibos & Auditoria
- **Recibo**: `request_id`, `checksum` e (opcional) assinatura.
- **Linha do tempo** por `trace_id` com ordenação determinística.
- **DLQ/Retry** para webhooks/entregas (produção).
- **Provas** exportáveis (hash/assinatura) sob demanda.

## 5) Tenants & Ambientes
- **Tenant** obrigatório (segregação lógica) e **ambiente** (`dev/stg/prd`).
- Políticas específicas por ambiente (limites mais rígidos no `prd`).
- **SLOs/SLA** por tier (Starter/Pro/Enterprise).

## 6) Observabilidade
- Métricas: taxa `ok/need_more/invalid`, latências (P50/P95), throughput por ação.
- Logs estruturados com `trace_id`, `action`, `resource` e `tenant`.
- **SSE/Webhooks** como padrão de retorno (tempo real), **polling** como fallback.

---
### Apêndice A — Arquivos de referência (bundle)
Arquivos detectados (amostra):
- README.md
- _TREE.txt
- .gitignore
- CHANGES_AUTOFIX.md
- DEPLOY_VERCEL.md
- DNS_GUIDE.md
- vercel.json
- package.json
- next.config.js
- tsconfig.json
- VERSION
- CHANGELOG.md
- README_CI.md
- DEPLOY_VERCEL.md.gpt
- DNS_GUIDE.md.gpt
… (total: 138 arquivos)

> Dica: mover “Policies”, “Keys/Wallet/Identity” e “Receipts/Audit” para o **Manual de Operação** do cliente.
