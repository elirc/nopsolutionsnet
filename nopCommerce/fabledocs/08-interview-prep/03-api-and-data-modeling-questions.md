# API and Data Modeling Question Cards

Twelve cards for the API/data round, all repo-anchored except where marked.

## Q1: How do you paginate a large collection endpoint?
Testing: whether you know offset pagination's limits.
Repo anchor: [IPagedList / GetAllPagedAsync](../../src/Libraries/Nop.Data/EntityRepository.cs#L302) — offset/limit baked into the data layer; every admin grid uses it.
Junior: "page and pageSize parameters." Mid: offset works until deep pages (DB still scans skipped rows) and mutates under writes (items shift between pages); keyset/cursor pagination fixes both at the cost of arbitrary-page jumps. Senior: matches to use case — admin grids (humans, shallow pages) = offset fine; feeds/APIs = cursor; and notes the *total count* query is often the hidden cost.
Follow-up: "Design the cursor for products sorted by price." (Non-unique sort key → composite cursor.)

## Q2: Where do you enforce authorization in a layered API?
Repo anchor: the three-mechanism stack — attribute ([CheckPermission](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L16)), entity ACL ([ProductController#L139](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L139)), row scope ([vendor filter](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L969)).
Junior: "middleware checks the token." Mid: capability at the edge, *ownership in the query* — the IDOR rule; anchor with the vendor example. Senior: pushes ownership predicates into shared query methods so new endpoints can't forget them; fail-open vs fail-closed defaults ([security checklist](../05-quality-engineering/05-security-checklist.md)).

## Q3: Soft delete vs hard delete — how do you decide?
Repo anchor: [ISoftDeletedEntity on Product](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13); `includeDeleted` parameter throughout [IRepository](../../src/Libraries/Nop.Data/IRepository.cs).
Junior: "soft delete so you can undo." Mid: the real driver is *referential integrity of history* — orders must resolve their products forever; costs: every query filters, uniqueness constraints complicate ("deleted" email reuse), GDPR needs true erasure paths anyway. Senior: split by entity class — transactional history: soft/never delete; PII: hard delete or anonymize (this repo has a GDPR service — [Nop.Services/Gdpr](../../src/Libraries/Nop.Services/Gdpr)); ephemera: hard delete.

## Q4: Model product options (size/color with price deltas). Go.
Repo anchor: the attributes machinery + `attributesXml` ([AddToCartAsync#L1550](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1550)) — and its critique ([type-system file](../02-stack-and-language-mastery/03-type-system-and-contracts.md)).
Junior: columns on product. Mid: attribute → values → combination entities; selections stored per cart/order line; price/weight adjustments per value ([the weight loop, ShoppingCartController#L904-L918](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L904-L918)). Senior: the variant-vs-attribute decision (pre-materialized SKUs vs combinatorial), inventory-per-combination, and honest storage critique (XML here; JSONB today).

## Q5: How do you evolve a production schema without downtime?
Repo anchor: forward-only, dated migrations ([PriceListMigration](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L7-L20)); full recipe in [02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md).
Junior: "run the migration at night." Mid: expand → backfill → flip → contract, each step shippable and survivable; forward-only means rollback = backup, so additive-first is mandatory. Senior: lock behavior of specific operations, batched backfills, and the org discipline (contract phase ships releases later).

## Q6: Idempotency for a payment/order API — design it.
Repo anchor: `OrderGuid` guard ([L1598](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1598)) + the mutex block (L1679-L1718) as the cautionary contrast.
Junior: "check if it exists first." (Race — that's exactly the read-check-write bug.) Mid: client-supplied idempotency key + unique constraint; on violation return the original result; TTL on stored results. Senior: scope of the key (per operation type), retry semantics with *same* key, and cites this repo's evolution as evidence you've thought about it against real code.

## Q7: What belongs in the request DTO vs computed server-side?
Repo anchor: totals recomputed in [PreparePlaceOrderDetailsAsync](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1602); the client sends product ids + quantities only.
Junior: "whatever the endpoint needs." Mid: the trust rule — identity of choices from the client, *consequences* (prices, totals, permissions) from the server; every client-supplied consequence is a vulnerability ([fake-code contrast 7](../04-code-reading-gym/03-fake-code-contrasts.md)). Senior: also versioning/compat — DTOs are contracts, entities are not ([contrast 8](../04-code-reading-gym/03-fake-code-contrasts.md)).

## Q8: Transactions — when do you need one, and what did this repo do instead?
Repo anchor: no cross-write transaction in [PlaceOrderAsync](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1616-L1644); ordering + repair instead ([02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
Junior: "wrap everything in a transaction." Mid: transaction for multi-write invariants *within one DB*; but external calls (payment!) can't join, so the boundary is drawn around DB writes only, with compensation for the external part. Senior: articulates the repo's implicit strategy (anchor row first, additive follow-ups, status gating) and when it stops being acceptable (financial reconciliation requirements).

## Q9: Cache a read-heavy endpoint — what invalidates it?
Repo anchor: cache-aside + event-driven prefix eviction ([pattern 4 + 11](../03-architecture-and-patterns/05-pattern-catalog.md)).
Junior: "TTL of five minutes." Mid: TTL is *staleness budget*, events are *correctness*; this repo does both, and the write spine makes event-driven eviction reliable. Senior: coarseness tradeoffs, multi-node story, stampede handling — the full answer is the [README golden-rule example](README.md).

## Q10: Multi-tenancy in one database — how, and what are the failure modes?
Repo anchor: store-scoping via mapping tables + `StoreId` columns ([StoreMappingService#L204-L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221)).
Junior: "tenant id column." Mid: shared-schema scoping with a *generic* authorize check; failure modes: unmapped-means-visible default, and every hand-written query must remember the scope. Senior: contrasts schema/db-per-tenant isolation levels, and proposes the systemic fix (scope enforced in a shared query layer, not per endpoint).

## Q11: Design the API error contract.
Repo anchor: warnings-as-data ([AddToCartAsync](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1559-L1582)) vs the inconsistent JSON shapes ([ticket 7](../06-contribution-practice/01-good-first-tickets.md)).
Junior: "return 400 with a message." Mid: distinguish validation (field-mapped, many at once — like the warnings list), domain rejections (stable machine-readable codes), and faults (500, correlation id, no internals leaked); one envelope for all. Senior: error codes are *versioned contract*; the repo's string warnings show exactly what unversioned, unmatchable errors cost consumers.

## Q12: Reporting queries are slowing the store down. Options?
Conceptual — no direct anchor (the repo runs reports on the primary).
Junior: "add indexes." Mid: indexes → then read replica for reporting → then a separate analytical store; each step trades freshness and ops cost. Senior: knows replica lag is a *product* decision (are morning-after numbers acceptable?), and that this repo's single-DB choice is right for its self-hosted scale — tying the generic ladder back to deployment reality.
