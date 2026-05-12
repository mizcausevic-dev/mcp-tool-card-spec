# MCP Tool Cards v0.1 — Specification

**Status:** Draft
**Version:** 0.1.0
**Editor:** Miz Causevic
**License:** AGPL-3.0 (this document, schema, and examples). Implementations are unrestricted.

RFC 2119 keywords apply throughout.

---

## 1. Scope

This specification defines a JSON document format for disclosing per-tool properties of an MCP (Model Context Protocol) server's tools. Audience: security reviewers, platform engineers, procurement teams, and consuming agents that need machine-readable answers to "what does this tool do, what can it break, and what has been tested?"

The specification does **not** modify the MCP protocol. It is a layered disclosure format; servers that publish Tool Cards remain fully MCP-compliant.

## 2. Terminology

- **MCP server** — a server implementing the Model Context Protocol.
- **Tool** — a single named operation exposed by an MCP server.
- **Tool Card** — a JSON document describing one tool from one MCP server.
- **Side-effect class** — declarative classification of what a tool changes when invoked.
- **Tested-with matrix** — list of LLM model versions a tool has been validated against, with measured pass rates.

## 3. The three pillars

### 3.1 Schema

Every Tool Card **MUST** include either:
- `schema.input_schema_inline` — the JSON Schema for tool input, embedded; OR
- `schema.input_schema_uri` — a URI dereferencing to a JSON Schema document.

Output schema declaration is **RECOMMENDED** but not mandatory.

### 3.2 Safety

Every Tool Card **MUST** include a `safety` block declaring:
- Side-effect class (`read` / `mutating` / `external` / `destructive`)
- Reversibility
- Rate-limit posture
- PII exposure profile
- Secrets handling profile
- Human-approval requirement

These are publisher disclosures; the spec does not enforce runtime behavior. Consumers **MAY** treat undeclared side-effect classes as `destructive` for safety-first defaults.

### 3.3 Trial

Every Tool Card **SHOULD** include a `tested_with` matrix — at least one LLM/model pair with a measurable pass rate from a published eval suite.

Production-deployed tools **SHOULD** additionally include `performance` with measured p50 and p99 latency from real traffic, and `cost` per call in a declared currency.

## 4. Document structure

### 4.1 `tool_card_version` (required)

A semver string. **MUST** be `"0.1"`.

### 4.2 `tool` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `server_id` | string | yes | Stable identifier for the MCP server. |
| `name` | string | yes | Tool name as exposed by MCP. |
| `version` | string | yes | Semver. |
| `mcp_server_uri` | URI | yes | Canonical MCP server endpoint. |
| `description` | string | yes | One-paragraph description of what the tool does. |

### 4.3 `schema` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `input_schema_uri` | URI | one of inline/uri | JSON Schema document URI for tool input. |
| `input_schema_inline` | object | one of inline/uri | JSON Schema embedded inline. |
| `output_schema_uri` | URI | no | JSON Schema document URI for tool output. |
| `output_schema_inline` | object | no | JSON Schema embedded inline. |

### 4.4 `safety` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `side_effect_class` | enum | yes | `read` / `mutating` / `external` / `destructive`. |
| `external_systems` | array of string | no | External systems contacted (databases, third-party APIs, etc.). |
| `reversible` | boolean | yes | Whether the operation can be undone after success. |
| `rate_limited` | boolean | yes | Whether the tool enforces a rate limit. |
| `pii_exposure` | enum | yes | `none` / `low` / `medium` / `high`. |
| `secrets_exposure` | enum | yes | `none` / `reads_secret_material` / `writes_secret_material` / `handles_keys`. |
| `human_approval_required` | boolean | yes | Whether the tool requires human approval before executing. |
| `refusal_modes` | array of string | no | Conditions under which the tool refuses to execute (e.g. `cross_tenant_access`, `rate_limit_exceeded`). |

### 4.5 `tested_with` (optional but recommended)

An array of test results. Each entry:

| Field | Type | Required | Description |
|---|---|---|---|
| `llm` | string | yes | Model identifier (e.g. `claude-opus-4-7`, `gpt-4o-2024-08-06`). |
| `provider` | string | no | Model provider. |
| `test_suite_uri` | URI | yes | Location of the test suite. |
| `pass_rate` | number | yes | 0.0–1.0. |
| `tested_at` | datetime | yes | ISO 8601 UTC. |
| `sample_size` | integer | no | Number of test cases. |

### 4.6 `performance` (optional but recommended for production)

| Field | Type | Description |
|---|---|---|
| `p50_latency_ms` | integer | Median latency in milliseconds. |
| `p99_latency_ms` | integer | 99th-percentile latency. |
| `measurement_window` | string | E.g. `last_7d`, `last_30d`. |

### 4.7 `cost` (optional)

| Field | Type | Description |
|---|---|---|
| `per_call_amount` | number | Cost per single invocation. |
| `per_call_currency` | string | ISO 4217 code (e.g. `USD`). |
| `notes` | string | Free-form (e.g. "amortized across burst capacity"). |

### 4.8 `audit` (required)

| Field | Type | Required | Description |
|---|---|---|---|
| `log_uri` | URI | no | Where call logs are retained. |
| `retention_days` | integer | no | How long call logs are retained. |
| `signed_by` | string | no | Identifier of the publisher attesting to this Tool Card. |
| `incident_response_uri` | URI | no | Surface for reporting misbehavior. |

## 5. Discovery convention

A Tool Card **MUST** be served at:

```
https://<mcp-server-origin>/.well-known/mcp-tools/<tool-name>.json
```

The MCP server **MAY** additionally expose the Tool Card URI inline in its `tools/list` response. The proposed extension key is `x-tool-card-uri`. Servers using this extension **SHOULD** also serve the canonical well-known URI.

## 6. Conformance levels

| Level | Requirements |
|---|---|
| **Level 1 — Schema** | Schema-valid Tool Card with all required fields, including a complete `schema` block. |
| **Level 2 — Safety** | Level 1, plus `safety.refusal_modes` declared (may be empty array — empty signals "no documented refusals"). |
| **Level 3 — Trial** | Level 2, plus at least one `tested_with` entry whose `tested_at` is within the last 90 days. |

`destructive` tools **MUST** declare `safety.human_approval_required: true`.

## 7. Security and privacy considerations

- **Tool surface exposure.** Publishing detailed Tool Cards may aid attackers planning agent prompt injections. Servers **MAY** withhold Tool Cards from public crawling and provide them only to authenticated consumers.
- **Test result honesty.** `pass_rate` is publisher-reported. Independent re-runs against the published `test_suite_uri` are the verification surface.
- **Side-effect class drift.** A tool may be classified `mutating` today and `destructive` after a code change. Servers **MUST** bump `tool.version` when the side-effect class changes.
- **Cost disclosure.** Per-call cost may reveal infrastructure information. Disclose at a granularity that informs procurement without enabling cost-side-channel attacks.

## 8. Relationship to existing work

| Standard | Relationship |
|---|---|
| **Model Context Protocol (MCP)** | Tool Cards are a layered disclosure format. They do not modify MCP; they add metadata MCP does not standardize. |
| **Agent Cards** ([agent-cards-spec](https://github.com/mizcausevic-dev/agent-cards-spec)) | Agent Cards reference Tool Card URIs in `capabilities.tools[].mcp_tool_card_uri`. |
| **OpenAPI** | Output schemas can reference OpenAPI components. The Tool Card itself is not an OpenAPI document. |
| **AEO Protocol** ([aeo-protocol-spec](https://github.com/mizcausevic-dev/aeo-protocol-spec)) | An organization's AEO declaration MAY enumerate MCP servers and link to their Tool Card directories. |

## 9. Open questions

- **Cross-server tool aliasing.** If two MCP servers expose the same logical tool (`billing-lookup`), should there be a way to declare equivalence at the Tool Card level?
- **Streaming tools.** Tools that return token streams have different latency semantics. Should `performance` add `time_to_first_byte_ms`?
- **Multi-tenancy.** Should `tested_with` include tenancy posture (single-tenant pass rates may differ from multi-tenant)?
- **Versioning policy.** Should the spec mandate that bumping `tool.version` invalidates prior `tested_with` entries, or are entries cumulative across versions?
