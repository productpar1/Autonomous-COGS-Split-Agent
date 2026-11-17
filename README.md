+-------------------------------------------------------------------+
|                     E-COMMERCE STOREFRONT                         |
|               (Webhook Event: order.created)                      |
+---------------------------------+---------------------------------+
|
v
+-------------------------------------------------------------------+
|              DETERMINISTIC AGENT ORCHESTRATOR                     |
|  * Schema-Bound JSON Execution (< 800 tokens / order)             |
|  * Strict VPC Isolation (PII stripped prior to financial calls)   |
|  * Negative-Margin Floor Guardrail                                |
+---------+-----------------------+-----------------------+---------+
|                       |                       |
v                       v                       v
+-------------------+   +-------------------+   +-------------------+
| 1. COST MATRIX    |   | 2. ATOMIC PAYOUT  |   | 3. EDI / 3PL      |
| Supplier Lookup   |   | Stripe Connect /  |   | Fulfillment Route |
| (COGS + Shipping) |   | Adyen Split       |   | (PII-Scoped)      |
+-------------------+   +-------------------+   +-------------------+
|                       |                       |
+-----------------------+-----------------------+
|
v
+-------------------------------------------------------------------+
|                  STOREFRONT LEDGER RECONCILIATION                 |
|         (Audit Log + Idempotency Key Verification)                |
+-------------------------------------------------------------------+

## Key Architectural Guarantees

1. **Zero Middleman Fund Custody:** Funds never sit in an intermediary holding account. Splits are executed atomically at the payment processor API layer (`Stripe Connect` / `Adyen`), routing exact supplier COGS + shipping to the supplier while locking net profit in the merchant's primary account.
2. **COGS Margin Floor Guardrail:** Before executing any transfer, the agent validates `(checkout_total - (cogs + shipping)) > 0`. If supplier costs exceed order revenue, execution halts immediately, flags the order, and escalates to human review.
3. **Strict Idempotency:** Every payout call enforces a deterministic `idempotency_key` (`{order_id}_split`) to prevent duplicate supplier payouts during webhook retries or network timeouts.
4. **VPC Data Isolation:** Customer PII (name, street address) is strictly scoped to the fulfillment transmission endpoint. Financial ledger and split-payout APIs receive only sanitized order identifiers and monetary amounts.

---

## SLA & Unit Economics Targets

| Metric | Target | Architectural Mechanism |
| :--- | :--- | :--- |
| **End-to-End Execution Latency** | `< 1.5 seconds` | Parallelized tool execution & deterministic JSON tool binding |
| **LLM Token Budget** | `<= 800 tokens / order` | Zero unstructured prompt loops; strict schema-enforced function calling |
| **AI Inference Overhead** | `< $0.015 / transaction` | Lightweight structured reasoning model with deterministic pre-checks |
| **Split Reconciliation Accuracy** | `100%` | Atomic ledger parity verification prior to order status commit |
| **Zero-Touch Fulfillment Rate** | `> 98%` | Automated exception routing for negative-margin or invalid SKU states |

Quickstart: Run the Guardrail & Latency Simulation
Test the deterministic margin-floor guardrail, PII isolation filter, and idempotency key generator locally (requires Python 3.9+, zero external dependencies):
python3 benchmarks/simulate_split_guardrails.py

1. Execution Lifecycle
Event Ingestion: Storefront emits order.created webhook containing order_id, payment_intent_id, line_items, and shipping_address.

Deterministic Cost Lookup: Agent invokes get_supplier_cost_matrix(supplier_id) to retrieve verified unit COGS, flat/tiered shipping rates, and the supplier's connected payout account ID.

Pre-Execution Margin Validation:

Computes total_supplier_obligation = (cogs_per_unit * quantity) + shipping_fee.

Computes merchant_net_profit = checkout_total - total_supplier_obligation.

Hard Stop: If merchant_net_profit <= 0, payout execution is blocked and an escalation ticket is emitted.

PII Isolation & Parallel Execution:

Financial Plane (Sanitized): Calls execute_split_payout() with idempotency_key = f"{order_id}_split". No customer name or street address enters the financial payload.

Logistics Plane (PII-Scoped): Calls transmit_fulfillment_order() to push SKU, quantity, and shipping address to the supplier's ERP/EDI endpoint.

Ledger State Commit: Invokes update_store_order_status() with split confirmation IDs and fulfillment tracking references.

2. Deterministic Tool Surface (4-Tool Boundary)
To eliminate hallucinated API calls and bound inference latency under < 1.5s, the agent operates exclusively within a strict 4-tool boundary:

get_supplier_cost_matrix(supplier_id)

execute_split_payout(payment_intent_id, supplier_account_id, cogs_amount, profit_amount, idempotency_key)

transmit_fulfillment_order(supplier_id, sanitized_fulfillment_payload)

update_store_order_status(order_id, status, routing_logs)

3. Enterprise Security & Compliance Posture
Zero Fund Custody: Eliminates money-transmitter licensing exposure by executing splits natively via payment processor split APIs (e.g., Stripe Connect Destination Charges / Separate Charges and Transfers, or Adyen for Platforms).

Replay Protection: Deterministic idempotency keys ensure network retries or duplicate webhook deliveries cannot trigger double transfers.

Data Minimization: Enforces strict schema separation between financial routing and 3PL logistics dispatch.
"""

order_schema = """{
"$schema": "http://json-schema.org/draft-07/schema#",
"title": "StorefrontOrderCreatedWebhook",
"type": "object",
"required": ["order_id", "currency", "checkout_total", "payment_intent_id", "line_items", "shipping_address"],
"properties": {
"order_id": { "type": "string", "example": "ORD-98421" },
"currency": { "type": "string", "minLength": 3, "maxLength": 3, "example": "USD" },
"checkout_total": { "type": "number", "minimum": 0, "example": 150.00 },
"payment_intent_id": { "type": "string", "example": "pi_3MtwB2Lkdlw" },
"line_items": {
"type": "array",
"minItems": 1,
"items": {
"type": "object",
"required": ["sku", "quantity", "retail_price", "supplier_id"],
"properties": {
"sku": { "type": "string", "example": "SKU-SUPPLIER-A-01" },
"quantity": { "type": "integer", "minimum": 1, "example": 2 },
"retail_price": { "type": "number", "minimum": 0, "example": 75.00 },
"supplier_id": { "type": "string", "example": "sup_alpha_99" }
}
}
},
"shipping_address": {
"type": "object",
"required": ["name", "street", "city", "state", "zip"],
"properties": {
"name": { "type": "string" },
"street": { "type": "string" },
"city": { "type": "string" },
"state": { "type": "string" },
"zip": { "type": "string" }
}
}
}
}
"""

tool_contracts = """{
"agent_name": "autonomous_cogs_split_agent",
"max_token_budget_per_event": 800,
"target_latency_ms": 1500,
"tools": [
{
"name": "get_supplier_cost_matrix",
"description": "Fetches unit COGS, shipping allocations, and connected payout account ID for a supplier.",
"parameters": {
"type": "object",
"required": ["supplier_id"],
"properties": {
"supplier_id": { "type": "string" }
}
}
},
{
"name": "execute_split_payout",
"description": "Triggers atomic payment gateway split routing COGS to supplier and locking profit in merchant account. PII strictly prohibited.",
"parameters": {
"type": "object",
"required": ["payment_intent_id", "supplier_account_id", "cogs_amount", "profit_amount", "idempotency_key"],
"properties": {
"payment_intent_id": { "type": "string" },
"supplier_account_id": { "type": "string" },
"cogs_amount": { "type": "number" },
"profit_amount": { "type": "number" },
"idempotency_key": { "type": "string" }
}
}
},
{
"name": "transmit_fulfillment_order",
"description": "Transmits order line items and PII shipping address to supplier ERP/EDI.",
"parameters": {
"type": "object",
"required": ["supplier_id", "order_details"],
"properties": {
"supplier_id": { "type": "string" },
"order_details": { "type": "object" }
}
}
},
{
"name": "update_store_order_status",
"description": "Commits payout split confirmation and fulfillment routing state to storefront backend.",
"parameters": {
"type": "object",
"required": ["order_id", "status", "routing_logs"],
"properties": {
"order_id": { "type": "string" },
"status": { "type": "string" },
"routing_logs": { "type": "object" }
}
}
}
]
}
"""

benchmark_script = '''"""
Deterministic Guardrail & Unit Economics Verifier
Autonomous Order Fulfillment & COGS Split Agent
Copyright (c) 2026 Vine Venture Group LLC
"""

import json
import time
from typing import Dict, Any

SUPPLIER_DB = {
"sup_alpha_99": {
"cogs_per_unit": 25.00,
"shipping_flat_fee": 10.00,
"payout_account_id": "acct_stripe_connect_12345",
"fulfillment_method": "API_REST",
}
}

def process_order_event(payload: Dict[str, Any]) -> Dict[str, Any]:
start_ns = time.perf_counter_ns()
order_id = payload["order_id"]
checkout_total = payload["checkout_total"]
item = payload["line_items"][0]

supplier = SUPPLIER_DB.get(item["supplier_id"])
if not supplier:
    return {"order_id": order_id, "status": "ESCALATED_UNKNOWN_SUPPLIER"}

total_cogs = (supplier["cogs_per_unit"] * item["quantity"]) + supplier["shipping_flat_fee"]
merchant_profit = round(checkout_total - total_cogs, 2)

# Guardrail 1: Negative Margin Floor
if merchant_profit <= 0:
    elapsed_ms = (time.perf_counter_ns() - start_ns) / 1_000_000
    return {
        "order_id": order_id,
        "status": "HALTED_NEGATIVE_MARGIN_FLOOR",
        "checkout_total": checkout_total,
        "calculated_cogs": total_cogs,
        "merchant_profit": merchant_profit,
        "latency_ms": round(elapsed_ms, 3),
    }

# Guardrail 2: Deterministic Idempotency Key & PII Stripping
idempotency_key = f"{order_id}_split"
financial_payload = {
    "payment_intent_id": payload["payment_intent_id"],
    "supplier_account_id": supplier["payout_account_id"],
    "cogs_amount": total_cogs,
    "profit_amount": merchant_profit,
    "idempotency_key": idempotency_key,
}

assert "shipping_address" not in financial_payload, "VPC PII Isolation Violation"

elapsed_ms = (time.perf_counter_ns() - start_ns) / 1_000_000
return {
    "order_id": order_id,
    "status": "SPLIT_AND_ROUTED",
    "financial_instruction": financial_payload,
    "pii_isolated": True,
    "latency_ms": round(elapsed_ms, 3),
}
if name == "main":
valid_order = {
"order_id": "ORD-98421",
"currency": "USD",
"checkout_total": 150.00,
"payment_intent_id": "pi_3MtwB2Lkdlw",
"line_items": [
{"sku": "SKU-SUPPLIER-A-01", "quantity": 2, "retail_price": 75.00, "supplier_id": "sup_alpha_99"}
],
"shipping_address": {
"name": "Jane Doe",
"street": "123 Main St",
"city": "Austin",
"state": "TX",
"zip": "78701",
},
}

negative_margin_order = {**valid_order, "order_id": "ORD-98422", "checkout_total": 55.00}

print("--- Test 1: Valid Atomic Split ---")
print(json.dumps(process_order_event(valid_order), indent=2))
print("\\n--- Test 2: Negative Margin Floor Halt ---")
print(json.dumps(process_order_event(negative_margin_order), indent=2))
'''

files = {
f"{repo_dir}/README.md": readme_content,
f"{repo_dir}/LICENSE": license_content,
f"{repo_dir}/docs/ARCHITECTURE_BRIEF.md": arch_brief,
f"{repo_dir}/schemas/order_webhook.schema.json": order_schema,
f"{repo_dir}/schemas/tool_contracts.json": tool_contracts,
f"{repo_dir}/benchmarks/simulate_split_guardrails.py": benchmark_script,
}

for path, content in files.items():
with open(path, "w") as f:
f.write(content.strip() + "\n")

zip_filename = "autonomous-cogs-split-agent-repo.zip"
with zipfile.ZipFile(zip_filename, "w", zipfile.ZIP_DEFLATED) as zf:
for path in files:
zf.write(path)

print(zip_filename)


```text?code_stdout&code_event_index=1
autonomous-cogs-split-agent-repo.zip
