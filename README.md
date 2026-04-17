# fMCP

A composition layer for MCP that lets agents submit multi-step plans as
a single typed, auditable expression — without routing intermediate data
through the model.

- SPEC.md — the specification (normative)
- PHILOSOPHY.md — why it's shaped this way
- composition-schema.json — normative JSON Schema for compositions
- benchmarks/ — empirical validation harness (coming soon)

## Status

v0.1-draft, 2026-04-17. Grammar stable. Empirical validation in progress.

## Quick look

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

One tool call result bound as `order`, then a second tool call that reads fields from it via `$order.path`. The composition is submitted as a single `fmcp.compose` call; the intermediate order data never returns to the model.

## License

Apache-2.0.