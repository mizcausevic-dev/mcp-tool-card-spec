# MCP Tool Cards

A draft specification for **per-tool disclosure documents** describing what an MCP tool does, what it can break, what it has been tested with, and how to report misbehavior.

The Model Context Protocol gives clients a way to *discover* an MCP server's tool surface. It does not give the server a way to *disclose* each tool's side effects, safety posture, latency profile, or tested-LLM compatibility. MCP Tool Cards fill that gap.

One JSON document per tool. Discoverable. Machine-readable. Designed for the audience that has to *approve* an MCP server's adoption — security reviewers, platform engineers, and CISOs.

## The three pillars

| Pillar | What it does |
|---|---|
| **Schema** | The input/output contract — either inline JSON Schema or a pointer to the MCP-native schema served by the tool |
| **Safety** | Side-effect classification, refusal modes, PII exposure profile, secret-handling profile, human-approval requirements |
| **Trial** | Tested-LLM matrix with pass rates, performance characteristics (p50/p99 latency, cost per call), and an audit-log surface |

## Why doesn't MCP already cover this?

The MCP specification standardizes **discovery and invocation** — how a client lists available tools and calls them. It does not standardize:

- **Side effects classification** (a `read_database` tool and a `delete_user` tool both look identical in MCP)
- **Tested-LLM matrix** — which models have been validated, with what pass rates
- **Latency and cost expectations** — operators need this to plan capacity and budget
- **PII / secrets exposure** — security reviewers need a structured answer, not prose
- **Refusal behavior** — when should the tool refuse to execute, and what does refusal look like

Tool Cards layer on top of MCP without modifying it. An MCP server that publishes Tool Cards remains MCP-compliant; clients that don't read Tool Cards continue to work.

## Quickstart

1. For each tool exposed by your MCP server, author a Tool Card conforming to [`tool-card.schema.json`](tool-card.schema.json).
2. Validate with any JSON Schema 2020-12 validator.
3. Serve at `https://<your-mcp-server>/.well-known/mcp-tools/<tool-name>.json` and reference the Tool Card URI from your MCP server's tool list.
4. Pair with an [Agent Card](https://github.com/mizcausevic-dev/agent-cards-spec) on consuming agents — agents reference Tool Cards via `capabilities.tools[].mcp_tool_card_uri`.

## Files in this repo

- [`SPEC.md`](SPEC.md) — full v0.1 specification
- [`tool-card.schema.json`](tool-card.schema.json) — JSON Schema (draft 2020-12)
- [`examples/`](examples/) — reference Tool Cards for a read-only billing lookup, a mutating ticket-create, and a destructive admin reset

## Status

**v0.1 draft.** Issues and pull requests welcome.

## License

AGPL-3.0. Specification text is freely implementable; schema and examples are AGPL-3.0.

## Kinetic Gain Protocol Suite

A family of open specifications for the answer-engine era. Each spec is a self-contained JSON document format with its own JSON Schema and reference examples; together they compose into an end-to-end account of entity, agent, prompt, tool, and citation.

| Spec | What it does |
|---|---|
| [AEO Protocol](https://github.com/mizcausevic-dev/aeo-protocol-spec) | Entity declaration at `/.well-known/aeo.json` — authoritative claims, citation preferences, audit hooks |
| [Prompt Provenance](https://github.com/mizcausevic-dev/prompt-provenance-spec) | Versioned, lineaged, reviewable LLM prompt records |
| [Agent Cards](https://github.com/mizcausevic-dev/agent-cards-spec) | Declarative agent capability and refusal disclosure |
| [AI Evidence Format](https://github.com/mizcausevic-dev/ai-evidence-format-spec) | Structured citations that travel with LLM-generated claims |
| **[MCP Tool Cards](https://github.com/mizcausevic-dev/mcp-tool-card-spec)** | Per-tool disclosure layered on Model Context Protocol servers |

---

**Connect:** [LinkedIn](https://www.linkedin.com/in/mirzacausevic/) · [Kinetic Gain](https://kineticgain.com) · [Medium](https://medium.com/@mizcausevic/) · [Skills](https://mizcausevic.com/skills/)
