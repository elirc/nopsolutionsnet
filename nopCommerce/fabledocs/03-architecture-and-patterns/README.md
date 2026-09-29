# 03 — Architecture and Patterns

The judgment module: not *what* the code does but *what it commits the project to*.

1. [01-boundaries-and-layers.md](01-boundaries-and-layers.md) — the layer rules and where they leak
2. [02-data-model-and-persistence.md](02-data-model-and-persistence.md) — schema, transactions, safe change
3. [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) — every gate, mapped
4. [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) — events, queues, retries, idempotency
5. [05-pattern-catalog.md](05-pattern-catalog.md) — 16 pattern cards (the module centerpiece)
6. [06-architecture-critique.md](06-architecture-critique.md) — strengths, risks, and what I'd change owning this

Vocabulary being trained (all defined in context as they appear, all interview-grade): *boundary, invariant, contract, ownership, idempotency, isolation, consistency, blast radius, migration, rollback, observability*. When you can use six of these accurately about code you've read, you interview differently.
