# fMCP — Philosophy and Design Rationale

**Version:** 0.1-draft
**Date:** 2026-04-17

> How this was written: this document was drafted by an AI assistant (Claude) from extended design conversations with Bryson Tang. The reasoning, reframes, and load-bearing insights came out of those conversations; the prose is a consolidation of that thinking into one place. Specific design claims were tested by pushback before being committed; an empirical benchmark to test the grammar against real LLMs is in progress and will be published separately. Where this document and SPEC.md disagree, SPEC.md prevails.

---

This document explains why fMCP is shaped the way it is. The specification itself is normative and terse; this document is the thinking behind it. If you are implementing fMCP, you do not need to read this. If you are evaluating it, extending it, or trying to understand why a particular design choice was made instead of an obvious alternative, this is where that lives.

## The problem fMCP exists to solve

Current MCP agent loops are imperative. The agent calls a tool, reads the result, decides what to do next, calls another tool, reads the result, and so on — each step interleaved with a full model turn. This pattern is what every MCP tutorial teaches and what every current agent framework implements.

It has four costs that compound badly as workflows get longer:

Tokens add up. Every intermediate result flows through the model's context, billed by the provider per megatoken. A four-step workflow that returns a 10KB result at step two pays for that 10KB at every subsequent step.

Latency adds up. Each step is a full prefill-and-decode round-trip. A workflow that could execute in 300ms as straight code takes 10+ seconds when it has to pass through three model turns.

Prompt injection lands on the control plane. Adversarial content in tool outputs can hijack the model's decision about what to do next. The defenses that exist today — "hope the model notices the injection attempt" — are not reliable and are an open research problem.

Plans are invisible until they've already started. Because the workflow is constructed turn by turn, there's no moment where an operator or a governance layer can see the whole thing and approve it. Approvals happen per-call, by which point partial work may already have occurred.

fMCP addresses all four by separating plan from execution. The model produces one thing — a composition that describes the whole workflow — and submits it as a single tool call. The rest happens server-side.

## The reframe that justifies the design

The most consequential property fMCP provides is not what it adds. It's what it moves.

Prompt injection in current agent loops is a research problem. You cannot prove that a sufficiently clever adversarial string won't trick the model into doing something it shouldn't. The defenses are heuristic, the attacks evolve, and the threat model is "the model itself might be the confused deputy."

fMCP makes the composition's control flow static. The model decides what tools to call before any tool is called; tool outputs flow through bindings but cannot change which tool runs next. Adversarial content can still appear in a tool's output, and that output can still flow into a downstream tool's arguments — but that's now an *engineering problem*, not a research problem. It's SQL injection. It's shell injection. It's XSS. It's the thing the AppSec industry has mitigations for. Parameterized queries. Argv-based invocation. Output encoding. Input validation at trust boundaries.

fMCP does not eliminate injection. It moves it from a place where defenses are speculative to a place where they are well-understood. That's the central security claim, and it's why the bounded-control-flow design is worth the ergonomic cost of asking agents to emit compositions instead of tool calls.

The same shape gives a separate property that matters for compliance. In imperative tool-calling, every intermediate result passes through the LLM provider's infrastructure. The provider sees it, may log it, handles it according to their policies. For HIPAA, PCI, GDPR-covered data, or any third-party NDA — this is often the blocking constraint. fMCP keeps intermediate results server-side. The provider sees the prompt, the tool schemas, the composition's shape, and whatever the `return` expression explicitly yields. Not the intermediate data. That's enumerable, documentable, defensible.

These two properties — research-to-engineering on injection, and enumerable LLM-provider exposure — are the reasons fMCP justifies its existence as more than a convenience layer.

## Why the grammar is this specific shape

Several decisions in the grammar trade elegance for LLM tractability. These are deliberate.

### Why `{name, arguments}` instead of `{tool, args}`

MCP's tool-call format is `{"name": ..., "arguments": ...}`. Every frontier model has been fine-tuned intensely on this exact shape since 2024-2025. When a composition's call nodes are byte-identical to MCP tool calls, the model's strongest generation prior is reusable without friction — writing a composition means writing MCP tool calls inside a small envelope, not learning a new dialect.

This hypothesis is being tested empirically in a forthcoming benchmark: compositions are generated across model sizes in both an MCP-aligned grammar and a sensible-but-different alternative, to measure whether structural alignment to the tool-call prior reliably improves one-shot correctness. "Align to what models already know" stands as a design principle regardless of the measurement's magnitude; the benchmark is about quantifying how much the principle is worth in practice.

### Why `let`/`return` at all, instead of pure nested function application

A pure nested form — `{"name": "f", "arguments": [{"name": "g", ...}, ...]}` — is shorter and more Lispy. It has a real aesthetic case. The problem is sharing: as soon as an intermediate value is referenced in more than one place, the nested form either recomputes it (wrong for cost and side effects) or hands it off to server-side memoization (which isn't statically analyzable). Haskell has `let` alongside pure expression syntax for exactly this reason. So does Racket. So does every language that's tried pure nesting at scale.

fMCP lets you use inline nesting when it's clean — a simple composition with no reuse can be written as a single `return` expression with nested calls. When you need to share an intermediate value, `let` names it. The grammar accommodates both styles; the user picks per-expression. This mirrors how real functional code is written, not how intro functional code is taught.

### Why no string interpolation

It's tempting to allow `"subject": "Past due — ${client.account_id}"` as a composition feature. We chose not to. Once string interpolation exists, it grows into a templating language within a year — escaping rules, conditionals inside templates, then loops inside templates, and you're inventing ERB or Jinja2 inside what was supposed to be a composition grammar.

The alternative is already clean: invoke a template-rendering tool with structured arguments. This is more verbose, and it's also better — the template is auditable (it's a tool call with typed context), operators reviewing the plan can see exactly what template is being used and what values it receives, and the templating engine is reusable across compositions.

### Why path access (`$name.field`) is in the grammar but arithmetic isn't

Both are plausible additions. Path access is in because it's how JSON data is navigated, models have strong priors for dot-access, and compositions that can't reach into structured results end up routing every field extraction through a helper tool, which is unnecessary friction. Arithmetic is out because it starts at `+` and ends at a full expression language; any serious computation belongs in a tool anyway, and the boundary between "this deserves to be in the grammar" and "this deserves to be a tool" has to go somewhere. Path access is trivial and ubiquitous; arithmetic is not.

### Why tools aren't first-class values

fMCP could allow `{"as": "processor", "value": "$template.render"}` — a tool as a value, passed to `map` as a function reference. Higher-order functional languages work this way.

The problem is static analysis. As soon as a tool reference can be dynamic — the tool to call is computed rather than named — the composition's scope (the set of tools it will invoke) can no longer be determined from the static form. Plan preview breaks. Authorization-ahead-of-execution breaks. Audit-trail shape becomes "we can tell you what happened, but not what was going to happen." The governance properties that make fMCP worth building depend on static scope extraction.

So fMCP is first-order: values are first-class, tools are not. Patterns that would use higher-order tool manipulation get expressed either by writing a tool that encapsulates the pattern or by unrolling the composition statically. This is the same trade total functional languages make.

## Why fMCP isn't Turing-complete

fMCP v0.1 has `let`, `return`, `map`, `on_error`, references, literals. It has no `while`, no recursion, no user-defined functions. Every well-typed composition terminates in finitely many steps.

This is not a temporary state pending future features. It's a commitment — to totality. Future versions may add constructs that preserve totality (conditional branching is the obvious v0.2 candidate), but will not add constructs that break it.

The totality commitment holds because every property that makes fMCP worth building depends on it:

**Plan preview.** To render a plan for operator review, the set of possible tool invocations must be determinable from the composition's static form. Bounded iteration (map over a finite array) lets us say "this composition will perform up to N tool calls of these types." Unbounded iteration breaks this — a `while` whose termination depends on tool output could invoke any number of tools. You can't preview a plan you can't enumerate.

**Authorization.** The scope of a composition is the set of tool names that might be called. With a bounded grammar, this is the union of `name` fields in the static form. With general recursion or unbounded iteration, scope extraction degrades to "anything reachable from here, which might be anything." Governance layers that authorize tool usage before execution can't work under that.

**Safety under LLM generation.** Small models generate fMCP compositions reliably because the grammar's possibility space is small. Each construct has one meaning. Every grammar addition compounds the probability of generating an incorrect composition — conditionals add branch selection, loops add termination reasoning, user-defined functions add scope reasoning. Bounded grammars stay cheap for models; Turing-complete grammars do not. This is why grammar additions (even totality-preserving ones like `if`) need to justify their cost against empirical generation quality, not just against theoretical expressivity.

**Termination analysis.** A bounded grammar guarantees that compositions don't hang. This is useful for operators (who want predictable latency), for governance (who want resource-bounded executions), and for audit (where a hanging composition is a more confusing failure mode than a crashed one).

The escape hatch for anything more expressive than composition is a tool. Tools can be Turing-complete. Tools can do recursion. Tools can run unbounded algorithms in whatever language they're written in, under whatever runtime isolation the host provides. The composition language coordinates tools safely; the tools themselves do the general computation.

This is the same discipline Unix applies — small composable programs, each doing one thing, connected by pipes. fMCP applies it to agent tool orchestration.

## The Turing-tarpit we avoid by holding this line

Every configuration language that lived long enough grew into a programming language, and most became bad programming languages. YAML grew Jinja2 templates, then Helm, then Kustomize, then CUE. Ansible grew conditionals, loops, filters, and a weird embedded language nobody can read. CSS is currently gesturing at Turing-completeness. SQL grew stored procedures. Every time, the growth happened because some specific feature seemed natural and bounded at the time, and then another feature seemed natural and bounded, and by feature number six the thing was Turing-complete and no one had noticed.

The way to avoid this is to state the commitment explicitly, put it in the spec, and hold the line. Every feature request gets evaluated against "does this belong in the grammar, or does this belong in a tool?" If it's unclear, the answer is tool. The grammar only grows when no tool-based alternative is ergonomic enough, and the addition preserves totality and static analyzability.

This isn't a hypothetical discipline; it's how Dhall, Nix, Agda, and the total functional programming tradition have successfully maintained bounded languages for years. fMCP joins that tradition deliberately.

## Why sanitization isn't a composition primitive

A natural design question: should fMCP provide a `sanitize` field at the composition level that runs a user-provided sanitization tool on every intermediate value between calls? We considered this and chose not to.

Sanitization is context-dependent. SQL injection mitigation requires parameterized queries at the SQL boundary. Shell injection requires argv-based invocation. HTML output encoding depends on whether the data lands in a text node, an attribute value, a JavaScript string literal, or a URL parameter — four different escape functions for the same input. A middleware sanitizer running between tool calls cannot know where the data is going next, so it can only do generic scrubbing. Generic scrubbing is either under-protective (scary characters were fine in the real context, legitimate characters were dangerous) or actively harmful (creates a false sense of security that masks the real vulnerability).

Sanitization belongs at the tool boundary, where the interpreter context is known. Compositions that need sanitization invoke a context-appropriate sanitization tool explicitly in their dataflow:

```json
{
  "let": [
    { "as": "raw", "name": "crm.get_customer_note", "arguments": { "id": "..." } },
    { "as": "safe", "name": "security.sanitize_for_email_body", "arguments": { "content": "$raw.text" } }
  ],
  "return": {
    "name": "email.send",
    "arguments": { "body": "$safe.cleaned" }
  }
}
```

This is explicit (visible in `fmcp.check`), composable (different sanitizers for different sinks), and auditable (each sanitization is its own tool call). Governance layers can enforce that compositions from a given profile include a sanitization step before any write to certain sinks — but that's a governance concern at the submission boundary, not a grammar concern. The grammar deliberately does not know which sinks are sensitive, because that's tenant- and deployment-specific knowledge that doesn't belong in a language specification.

This pattern — "compute that can't know enough to do the job right doesn't belong at this layer" — is a recurring principle. It's also why tools aren't first-class, why there's no string interpolation, why error handling is fail-fast by default. The composition language knows composition; tools know domains; governance knows policy. Each layer does its job.

## Why this is functional despite having sequential bindings

A composition's `let` bindings are evaluated in declaration order. On first read this looks like imperative sequencing — do this, then do that, and the ordering matters.

It isn't. The ordering in fMCP is **dependency ordering**, not **state mutation**. A binding named `order` is computed before a binding named `contact` because `contact`'s definition references `$order.customer_email` — the dataflow requires it. Two bindings that don't depend on each other can be evaluated in any order (or in parallel) without changing the result. This property is called confluence, and it's one of the standard properties of functional languages.

Single-assignment form (every name bound once, never reassigned) is what makes fMCP functional. There is no mutation, no shared state, no "later in the composition, `$order` means something different." References are to values, not to memory locations. This is exactly what Haskell's `let`, OCaml's `let`, Scheme's `let`, and the simply-typed lambda calculus's `let` provide.

The academic name for the specific variety fMCP is: first-order, total, typed, functional composition language. First-order because values are first-class but tools are not. Total because every well-typed composition terminates. Typed because expressions are checked against schemas before execution. Functional because composition is expression evaluation over immutable bindings. The tradition includes Dhall, the Nix expression language, Agda's terminating fragment, Lean, and the simply-typed lambda calculus. This is a serious family; fMCP joins it with specific decisions that make the composition language practical for LLM generation and agent governance.

## Alternatives considered and rejected

**Lisp / S-expression grammar.** More elegant. Weaker LLM priors. Rejected for ergonomic reasons; the elegance loss is real but the generation quality matters more.

**Nested-only (no `let`).** Racket-aesthetic. Breaks down when intermediate values are referenced more than once; forces either recomputation (wrong) or server-side memoization (not statically analyzable). fMCP retains nested inline calls for simple cases but provides `let` for reuse.

**Higher-order tools.** Lets you pass tools as values, compose them dynamically. Breaks static scope extraction, breaks plan preview, breaks authorization-ahead-of-execution. Rejected because the governance properties are load-bearing.

**String interpolation.** Ergonomically tempting. Grows into a templating language within a year. Rejected in favor of template-rendering tools, which are better in every way except for the extra verbosity in the simplest case.

**Composition-level sanitization middleware.** Context-dependent problems require context-aware mitigations; a middleware layer can't provide them correctly. Rejected; sanitization happens at tools.

**MCP protocol extension (new JSON-RPC method for composition).** Would be a cleaner protocol design but would require every MCP client to update, would require a new spec committee, and would fragment the ecosystem during adoption. fMCP is a tool, invoked via standard `tools/call`, and requires nothing from clients or from the MCP protocol itself.

**Full Turing-completeness.** Would make fMCP a general programming language. Would break plan preview, scope extraction, termination guarantees. Rejected permanently; any computation that requires Turing-completeness belongs in a tool.

## What comes next

v0.1 is deliberately minimal. The candidates for v0.2 are `if` (conditional branching), parallel map mode, named reusable compositions (procedures), cross-server composition, and an explicit approval-gate primitive. Each one needs empirical evidence that it earns its weight — that the pattern is common enough, underserved enough by the tool ecosystem, and safe enough to add without breaking the bounded-grammar commitment.

A benchmark harness is being built as the mechanism for that evidence. Future additions get tested before landing in the spec: does adding the feature improve or degrade small-model generation? Does it preserve static analyzability? Does it pull its weight against "just write a tool that does this"?

Things that will not be added: general recursion, `while`, user-defined functions, mutable state, string interpolation beyond minimal cases, arbitrary control flow. These aren't unfortunate omissions pending future consideration. They're structural commitments that make the spec worth what it is.

## Closing

fMCP is an attempt to take the part of agent tooling that's currently a research problem and convert it into an engineering problem. The design discipline — bounded grammar, static analyzability, MCP-aligned shape, tools as the escape hatch — is not decorative. Every constraint is load-bearing for a property that matters.

If you're thinking about implementing something fMCP-shaped, or extending fMCP, the question that should guide the work is: does this decision preserve the properties that make fMCP worth having? If yes, it's aligned with the design. If no, it's probably a v1.0-style breaking change and needs to be evaluated as such. If unclear — if the answer depends on specific use cases, specific deployments, specific tradeoffs — it probably belongs in a tool, in a governance layer, or in a host-specific extension rather than in the spec itself.

Bounded, total, first-order, typed, functional, composable, MCP-aligned, LLM-ergonomic, governance-compatible. That's the shape. The rest is details.

---

*End of philosophy v0.1-draft.*