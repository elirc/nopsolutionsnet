# Data Model and Persistence

## Key entities and relationships

```
Store ──┐                    Customer ──< CustomerRole >── PermissionRecord
        │ (StoreMapping: generic per-entity)     │
Category ──< ProductCategory >── Product ──< ShoppingCartItem (cart & wishlist)
                                    │  \── ProductAttribute machinery (XML in items)
                                    │
                              Order ──< OrderItem  (price SNAPSHOTS, not references)
                                    └── QueuedEmail (outbox)
```

Conventions: every table row is `BaseEntity { int Id }`; FKs are plain `int` properties (`ProductId`, not navigation properties — this is Linq2Db, not EF: **no lazy loading exists**, all joins are explicit, which is why N+1 here is *visible* awaited calls, not hidden proxy magic). Schema is defined in code by builders ([ProductBuilder.MapEntity#L17](../../src/Libraries/Nop.Data/Mapping/Builders/Catalog/ProductBuilder.cs#L17)).

Three modeling decisions worth internalizing (each is an interview answer):

1. **Orders snapshot, catalogs reference.** `OrderItem` copies prices at purchase ([SaveOrderDetailsAsync#L723](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L723)); order history is immutable regardless of catalog edits. Rule: *reference data that may change → copy what the past must remember.*
2. **Soft delete for referenced entities.** `ISoftDeletedEntity` ([Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13)); repository reads expose `includeDeleted` ([GetByIdAsync#L122](../../src/Libraries/Nop.Data/EntityRepository.cs#L122)). Invariant: an order's product must always resolve. Cost: every query must remember the filter — note [ProductController.cs#L132](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L132) re-checks `product.Deleted` manually.
3. **Polymorphic mapping tables** for ACL/store-mapping/slugs (entity-name + id). Generic and extensible; costs referential integrity — the DB can't FK-check `("Product", 42)`.

## Transactions and consistency

The uncomfortable, instructive truth: business flows here are **not** wrapped in explicit multi-write transactions. `PlaceOrderAsync` performs a sequence of separate writes (order → items → histories → notes, [L1616-L1644](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1616-L1644)); a mid-sequence crash leaves partial state. Mitigations in the design: writes are ordered so the *anchor row* (Order) lands first, later steps are additive, statuses gate visibility, and support tooling can repair. This is "eventual consistency by ordering + repair" — common in mature systems, rarely admitted in tutorials. Consistency expectation to state in interviews: **per-write atomicity from the DB, cross-write consistency by design discipline**, and know which flow (checkout) deserves the upgrade to a real transaction/outbox.

Concurrency: no optimistic-concurrency tokens on entities — last write wins (see the lockout counter race, [CustomerRegistrationService.cs#L160](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L160)). Read-then-check-then-write sequences ([max cart items, ShoppingCartService.cs#L1637](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1637)) are race windows the DB never hears about; the durable fix is always *a constraint the DB enforces*.

## Migrations: how to change the schema safely here

Mechanics: FluentMigrator, timestamp-versioned attributes, `UpgradeTo{version}` folders, forward-only ([PriceListMigration](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L7-L20)), idempotent helpers (`CreateTableIfNotExists`). Installation schema lives separately in [Migrations/Installation](../../src/Libraries/Nop.Data/Migrations/Installation).

The safe-change recipe (expand/contract — memorize; it's a top-5 interview topic):

1. **Expand:** add the new nullable column/table in a migration; deploy code that writes both old+new but reads old.
2. **Backfill:** data migration or lazy backfill; verify counts.
3. **Flip reads** to the new column behind the same release or a setting.
4. **Contract:** only releases later, drop the old column — a separate migration users can decline until ready.

Forward-only makes step 4's discipline mandatory: **rollback = restore from backup**, so every migration must be survivable in production the moment it runs. Never mix a long-running data backfill into a schema migration that holds locks.

Drill: design the migration set to add `Product.CountryOfOriginId` (FK to Country, admin-editable, storefront-displayed). List: builder change, migration(s), which release drops nothing (answer: all of them), and which layers you touch (builder → entity → service? → factory → admin model/validator/view). Basic: schema right. Solid: expand/contract phases explicit. Strong: also names the localization and export/import surfaces you'd check ([Nop.Services/ExportImport](../../src/Libraries/Nop.Services/ExportImport) exists — did you remember it?).
