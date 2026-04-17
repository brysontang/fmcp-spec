# fMCP — Functional Model Context Protocol

**Version:** 0.1-draft
**Status:** Draft. Grammar stable; empirical validation in progress.
**Date:** 2026-04-17
**License:** Apache-2.0

> How this was written: fMCP was designed by Bryson Tang through a series of extended design conversations with an AI assistant (Claude). The assistant helped draft, structure, and test language across this document; the design decisions, corrections, taste, and commitments are Bryson's. Every non-trivial design choice in this spec survived a pushback round. See PHILOSOPHY.md for the thinking behind the shape.

---

A composition layer for MCP that lets agents submit multi-step plans as a single typed, auditable expression — without routing intermediate data through the model.

## 1. Overview

fMCP defines a composable MCP tool, `fmcp.compose`, which accepts a **composition** — a typed, bounded dataflow expression over other MCP tool calls — and returns its result. The composition is evaluated server-side; intermediate tool results do not pass through the LLM's context.

fMCP does not modify the MCP protocol. It is additive: any MCP 2025-06-18+ compliant server MAY implement `fmcp.compose`, any MCP client that can invoke tools can invoke it, and no new transport, authentication, or client SDK is required. fMCP-unaware clients see `fmcp.compose` as one more tool and MAY ignore it.

For motivation and design rationale, see PHILOSOPHY.md.

## 2. Terminology

- **Composition** — A JSON value conforming to §4. The argument to `fmcp.compose`.
- **Binding** — A named value introduced by a `let` clause.
- **Reference** — A string of the form `$name` or `$name.path` resolving to a binding's value.
- **Call node** — A sub-expression invoking a tool. Structurally identical to an MCP `tools/call` inner object: `{"name": ..., "arguments": ...}`.
- **Scope** — The set of tool names referenced by any call node.
- **Host** — The MCP server implementing `fmcp.compose`.

Throughout, **MUST**, **SHOULD**, and **MAY** follow RFC 2119.

## 3. Design commitments

fMCP is:

- **First-order functional**: values are first-class and immutable; tools are not first-class values.
- **Total**: every well-typed composition terminates in finitely many steps, given terminating tools.
- **Typed**: compositions are type-checked against MCP tool schemas before execution.
- **Bounded**: no general recursion, no unbounded iteration, no user-defined functions.

These are load-bearing properties. They enable static plan preview, scope extraction, and safety analysis. Rationale in PHILOSOPHY.md.

## 4. Grammar

A composition is a JSON object. The normative JSON Schema is `composition-schema.json` in this repository; where prose and schema disagree, the schema prevails.

### 4.1 Composition

```
Composition ::= {
  "let"?:       Binding[],
  "return":     Expression,
  "on_error"?:  Expression
}
```

- `let` (optional): ordered array of bindings, each naming a value.
- `return` (required): the expression whose value is the composition's result.
- `on_error` (optional): evaluated if any binding or the return fails (§8).

### 4.2 Binding

A binding has an `as` field naming it, plus exactly one of a `value`, a call shape, or a `map` shape:

```
Binding ::=
  | { "as": Identifier, "value": JsonValue }
  | { "as": Identifier, "name": ToolName, "arguments": Object }
  | { "as": Identifier, "map": Expression, "each": Identifier, "do": Expression, "mode"?: MapMode }
```

The type is determined by which keys are present. A binding MUST have exactly one of these shapes.

**Identifiers** match `[a-zA-Z_][a-zA-Z0-9_]*`. Binding names MUST be unique within a composition. No shadowing.

### 4.3 Expression

An expression is one of:

- A **reference**: a string beginning with `$` (§4.4).
- A **literal**: any JSON value that is not a reference and not a call or map shape. Numbers, booleans, null, arrays, and objects — recursively, provided no nested string is a reference.
- An **inline call**: `{"name": ToolName, "arguments": Object}`. Semantically equivalent to binding it and referencing it once.
- An **inline map**: `{"map": ..., "each": ..., "do": ..., "mode"?: ...}`.

### 4.4 References

```
Reference  ::= "$" Identifier ("." PathSegment)*
PathSegment ::= Identifier | NonNegativeInteger
```

Examples:
- `"$order"` — the value of binding `order`
- `"$order.customer.email"` — field access
- `"$items.0.sku"` — array index + field

**Escape**: `"$$literal"` is the literal string `"$literal"`.

**Substitution in arguments**: within a call's `arguments`, any string value that is a reference is replaced by the referenced value. Substitution recurses into nested objects and arrays. Keys are not substituted.

```json
"arguments": {
  "to": "$customer.email",
  "subject": "Welcome",
  "meta": { "source": "$trace_id" }
}
```

**Resolution timing**: references resolve to a binding declared earlier in the composition. Forward references and references to undeclared bindings are type-check errors.

### 4.5 Map

Iterates a body expression over a list, returning a list of results.

```
{
  "map":    Expression,    // must evaluate to an array
  "each":   Identifier,    // bound per-iteration inside `do`
  "do":     Expression,    // evaluated per element
  "mode"?:  "serial" | "collect"    // default: "serial"
}
```

The `each` identifier MUST NOT collide with any outer binding. v0.1 evaluates serially; parallel evaluation is reserved for v0.2.

### 4.6 Reserved names

Identifiers beginning with `$__` are reserved for runtime-provided bindings:
- `$__errors` — populated in `on_error` handlers (§8)
- `$__execution_id` — the composition's execution identifier

## 5. Type checking

Before execution, hosts MUST verify:

1. **Grammar validity** against `composition-schema.json`.
2. **Identifier uniqueness**: all `as` names unique.
3. **Reference resolvability**: every reference names a prior binding.
4. **Path validity**: every `.path` segment consistent with the referenced binding's schema.
5. **Tool existence**: every `name` is known to the host via `tools/list`.
6. **Argument validity**: after substitution, each call's `arguments` validates against the tool's `inputSchema`.
7. **Scope authorization**: every tool in the composition's scope is authorized for the caller.

Failure at any step MUST reject the composition before any tool is invoked. Errors are structured per §11.

**Structured outputs**: when a tool declares an `outputSchema`, `.path` access on its result is validated statically. A tool without `outputSchema` is opaque; `.path` access on its result is a type-check error.

**Literal typing**: literal bindings (`value`) have the JSON Schema type implied by their JSON value.

## 6. Scope and authorization

A composition's **scope** is the set of tool names appearing in any call node:

```
scope(composition) = { name : name appears in a call shape anywhere }
```

The host MUST authorize every tool in the scope before execution. Authorization mechanism is host-defined. Hosts SHOULD re-validate authorization for each call at execution time, to defend against mid-execution authorization changes.

## 7. Execution

Once type-checking and authorization succeed:

1. **Bindings evaluate in declaration order.** Each binding completes before the next begins.
2. **Call evaluation**: resolve references in `arguments`, validate against `inputSchema`, invoke the tool, bind the result.
3. **Map evaluation** (serial): evaluate the list, then for each element bind `each` and evaluate `do`, in order. Collect results into an array.
4. **Return**: evaluate `return` after all bindings. Its value is the composition's result.

Bindings are immutable. Two bindings with no data dependency on each other may be evaluated in any order without observable difference (confluence). v0.1 specifies serial evaluation.

Hosts MUST emit a composition-level audit event including the composition (or a content hash), the scope, the caller identity, and the outcome. Per-tool-call audit events are emitted as for any direct MCP tool invocation.

## 8. Error handling

v0.1 uses **fail-fast** with two escape valves: the `on_error` handler and the `collect` map mode.

**Fail-fast (default)**: any failure — tool error, runtime path error, runtime type error, timeout — halts the composition. Prior tool calls are not rolled back. If `on_error` is absent, the composition fails and returns a structured error.

**`on_error`**: evaluated when the main body fails. Inside `on_error`:
- `$__errors` is bound to an array of error records.
- `$__execution_id` is bound to the execution identifier.
- Bindings that completed successfully remain in scope.

If `on_error` itself fails, the composition fails with the `on_error` error; hosts SHOULD include both.

**Map `collect` mode**: each iteration yields `{"ok": true, "value": ...}` or `{"ok": false, "error": ...}`. The map result is an array of these records. This is the only grammar path by which tool-error detail can reach the caller; compositions that return collect-mode results directly will expose those errors unless transformed by a subsequent tool or routed through `on_error`.

## 9. Introspection

Hosts SHOULD expose `fmcp.check`, which accepts the same input as `fmcp.compose`, performs type-checking and scope resolution, and returns a plan description without executing any tool. The plan SHOULD include the scope, the binding dependency graph, any `.path` accesses, and the return expression's inferred type. This enables host UIs to render plans for operator review before execution.

## 10. Integration with MCP

### 10.1 Tool shape

Hosts expose `fmcp.compose` as a standard MCP tool. Because `fmcp.compose`'s output type depends on the composition, hosts SHOULD either declare a permissive `outputSchema` (or omit it) or provide `fmcp.check` to return per-composition output schemas.

### 10.2 Naming

The canonical tool name is `fmcp.compose`. Implementations MAY expose vendor-namespaced variants (e.g., `fmcp.acme.compose`) when providing differing semantics. Bare `compose` without namespace is discouraged.

### 10.3 Coexistence

Hosts exposing `fmcp.compose` MUST continue to expose underlying tools individually. fMCP is additive; it does not gate or replace direct tool access.

### 10.4 Notifications

Tool-list changes use MCP's existing `notifications/tools/list_changed`. Ongoing compositions are unaffected; subsequent compositions type-check against the updated tool set.

### 10.5 MCP version

fMCP v0.1 requires MCP 2025-06-18 or later, because it depends on `outputSchema` for type checking.

## 11. Errors

Errors are structured JSON objects. Every error MUST include `kind`, `node_path` (or null), and `message`.

| `kind`                       | When                                                      | Additional fields                                |
| ---------------------------- | --------------------------------------------------------- | ------------------------------------------------ |
| `fmcp.grammar_error`         | Composition fails schema validation                       | —                                                |
| `fmcp.reference_error`       | `$name` does not resolve                                  | `referenced_binding`, `available_bindings`       |
| `fmcp.path_error`            | `.path` invalid against referenced binding's schema       | `referenced_binding`, `requested_path`           |
| `fmcp.type_error`            | Arguments don't validate against tool's `inputSchema`     | `expected_schema`, `actual_schema`               |
| `fmcp.unknown_tool`          | Referenced tool not known to host                         | `referenced_tool`, `suggestions`                 |
| `fmcp.scope_error`           | Referenced tool not authorized for caller                 | `referenced_tool`                                |
| `fmcp.runtime_error`         | Tool call fails during execution                          | `upstream_error`                                 |
| `fmcp.resource_error`        | Host limit exceeded                                       | `limit_exceeded`, `limit_value`                  |
| `fmcp.unsupported_feature`   | Host does not implement a used feature                    | `unsupported_feature`                            |

Example:

```json
{
  "kind": "fmcp.type_error",
  "node_path": "let[1].arguments.customer_id",
  "message": "Expected type 'string' but got type 'object' from '$customer'.",
  "expected_schema": { "type": "string" },
  "actual_schema": { "type": "object" }
}
```

## 12. Security requirements

### 12.1 Trust model

fMCP does not change MCP's trust model. Tool invocations within a composition carry the same authorization, confirmation, and auditing requirements as direct invocations.

### 12.2 Injection

Composition control flow is fixed at submission time. Tool outputs flow through bindings into subsequent tools' `arguments` but cannot change which tools are called. This bounds prompt injection at the control layer; injection risk at the data layer (untrusted output flowing into interpreters) is the tool implementer's responsibility, handled with standard mitigations (parameterized queries, argv-based shell invocation, output encoding, etc.).

Sanitization is context-dependent and belongs at the tool boundary, not in the grammar. Compositions needing sanitization invoke a context-appropriate sanitization tool explicitly. Governance layers MAY enforce such invocations at composition submission. Rationale in PHILOSOPHY.md.

### 12.3 LLM-provider exposure

Intermediate tool results do not transit through the LLM's context; they flow through server-side bindings. The LLM provider sees the prompt, the advertised tool schemas, the composition shape, and whatever the composition's `return` expression yields — nothing else.

This property is preserved or defeated by the `return` expression. Composition authors SHOULD design `return` to yield only data appropriate for the caller's trust level, and SHOULD NOT return raw sensitive records unless that is the caller's explicit need.

### 12.4 Resource limits

Hosts SHOULD enforce:

- Maximum nested `map` depth (recommended: 3)
- Maximum total tool invocations per composition (recommended: 1000)
- Maximum total execution time (host-defined)
- Per-caller rate limits

Exhaustion MUST produce a structured `fmcp.resource_error`.

### 12.5 Audit integrity

The composition (or a content hash) MUST be recorded in the audit event. Hosts SHOULD store compositions sufficiently to reproduce execution paths, subject to data-retention policy.

## 13. Examples

### Single call

```json
{
  "return": {
    "name": "weather.get_current",
    "arguments": { "location": "Raymond, NH" }
  }
}
```

### Two-step read-then-format

```json
{
  "let": [
    {
      "as": "order",
      "name": "crm.get_order",
      "arguments": { "order_id": "ORD-7742" }
    }
  ],
  "return": {
    "name": "template.render",
    "arguments": {
      "template_name": "order_confirmation",
      "context": {
        "customer_name": "$order.customer.name",
        "total": "$order.total"
      }
    }
  }
}
```

### Literal binding

```json
{
  "let": [
    { "as": "threshold", "value": 30 },
    {
      "as": "clients",
      "name": "billing.query_accounts",
      "arguments": { "status": "past_due", "days_overdue_gte": "$threshold" }
    }
  ],
  "return": "$clients"
}
```

### Map with fan-out

```json
{
  "let": [
    {
      "as": "clients",
      "name": "billing.query_accounts",
      "arguments": { "status": "past_due", "days_overdue_gte": 30 }
    }
  ],
  "return": {
    "map": "$clients",
    "each": "client",
    "do": {
      "name": "email.send_past_due_notice",
      "arguments": { "account_id": "$client.account_id" }
    }
  }
}
```

### With `on_error`

```json
{
  "let": [
    {
      "as": "results",
      "map": {
        "name": "billing.query_accounts",
        "arguments": { "status": "past_due" }
      },
      "each": "client",
      "mode": "collect",
      "do": {
        "name": "email.send_past_due_notice",
        "arguments": { "account_id": "$client.account_id" }
      }
    }
  ],
  "return": {
    "name": "billing.record_outreach_batch",
    "arguments": { "results": "$results" }
  },
  "on_error": {
    "name": "ops.log_composition_failure",
    "arguments": {
      "execution_id": "$__execution_id",
      "errors": "$__errors"
    }
  }
}
```

## 14. Versioning

This specification is versioned `v0.1-draft`. Within v0.x, breaking changes may occur between minor versions. From v1.0, new features are additive; breaking changes require a major version bump.

Features reserved for v0.2+ include `if`, `approve` primitives, parallel map mode, cross-server composition, and named reusable compositions. Features explicitly excluded from future scope: general recursion, `while`, user-defined functions, mutable state. Rationale in PHILOSOPHY.md.

## 15. Appendix — Recommended agent guidance

Hosts implementing `fmcp.compose` may include the following in their agent system prompts:

> When a task involves more than one tool call, prefer to submit a single composition to `fmcp.compose` rather than making separate tool calls across multiple turns. A composition describes the full dataflow: `let` names intermediate results, `$name.path` passes data between steps, `map` iterates over lists, `return` specifies the result. Using `fmcp.compose` is faster, cheaper, and safer than step-by-step orchestration because intermediate results do not pass through your context and the plan is approved as a whole. Use individual tool calls when the task is a single step or when you need to inspect an intermediate result before deciding what to do next. Declare `let` bindings for values referenced more than once; use inline nested calls when a value is used exactly once.

---

*End of specification v0.1-draft.*