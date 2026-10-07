# Data Model and Persistence

How entities become tables, how reads and writes actually reach the database, what is (and isn't) transactional, and how to change the schema safely. Every mechanism below is cited to the line that implements it.

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

Conventions: every table row is `BaseEntity { int Id }` ([BaseEntity.cs#L6-L12](../../src/Libraries/Nop.Core/BaseEntity.cs#L6-L12)); FKs are plain `int` properties (`ProductId`, not navigation properties — this is Linq2Db, not EF: **no lazy loading exists**, all joins are explicit, which is why N+1 here is *visible* awaited calls, not hidden proxy magic).

## How an entity becomes a table

The schema is defined in C#, and the ORM mapping is *derived from the schema*, not the other way round:

1. **Builders declare the exceptions.** `ProductBuilder.MapEntity` only lists columns that need non-default treatment — eight string columns with lengths and nullability ([ProductBuilder.cs#L17-L28](../../src/Libraries/Nop.Data/Mapping/Builders/Catalog/ProductBuilder.cs#L17-L28)). There are 127 files matching `*Builder.cs` under `Mapping/Builders` (counted with `find … | wc -l`).
2. **Everything else is auto-mapped by reflection.** `RetrieveTableExpressions` runs the builder, adds an identity `Id` primary key if none was declared, then auto-maps every public writable property whose type appears in a fixed table and that isn't marked `[NotMapped]`/`[NotColumn]` ([FluentMigratorExtensions.cs#L248-L288](../../src/Libraries/Nop.Data/Extensions/FluentMigratorExtensions.cs#L248-L288)).
3. **The type table sets the defaults** ([FluentMigratorExtensions.cs#L28-L38](../../src/Libraries/Nop.Data/Extensions/FluentMigratorExtensions.cs#L28-L38)): `string` → `AsString(int.MaxValue).Nullable()` (i.e. unbounded and nullable unless a builder says otherwise), `decimal` → `decimal(18,4)`, `DateTime` → a provider-specific `datetime2`, plus `int`, `long`, `bool`, `byte[]`, `Guid`. Enums aren't in the table, which is why entities expose `XxxId` ints with typed wrapper properties.
4. **Linq2Db reads the same definitions.** `FluentMigratorMetadataReader` produces `[Table]`/`[Column]` attributes at runtime from the entity descriptor, matching properties to columns by name and skipping read-only properties ([FluentMigratorMetadataReader.cs#L86-L118](../../src/Libraries/Nop.Data/Mapping/FluentMigratorMetadataReader.cs#L86-L118)).
5. **Foreign keys cascade by default.** The `ForeignKey<TPrimary>` helper's `onDelete` parameter defaults to `Rule.Cascade` ([FluentMigratorExtensions.cs#L106](../../src/Libraries/Nop.Data/Extensions/FluentMigratorExtensions.cs#L106)), as does `AddOrAlterForeignKeyColumnFor` (L192). A hard delete of a parent can therefore remove children you didn't name — one more reason soft delete exists.

## Three modeling decisions worth internalizing

1. **Orders snapshot, catalogs reference.** `OrderItem` copies prices at purchase (`UnitPriceInclTax`, `PriceExclTax`, `OriginalProductCost`, a rendered `AttributeDescription` and the raw `AttributesXml` — [OrderItem.cs#L31-L71](../../src/Libraries/Nop.Core/Domain/Orders/OrderItem.cs#L31-L71)); created in [SaveOrderDetailsAsync#L723](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L723). Order history is immutable regardless of catalog edits. Rule: *reference data that may change → copy what the past must remember.*
2. **Soft delete for referenced entities.** `ISoftDeletedEntity` ([Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13), [Customer.cs#L9](../../src/Libraries/Nop.Core/Domain/Customers/Customer.cs#L9)). Invariant: an order's product must always resolve. Cost: every read must remember the filter.
3. **Polymorphic mapping tables** for ACL/store-mapping/slugs (entity-name + id). Generic and extensible; costs referential integrity — the DB can't FK-check `("Product", 42)`, and renaming an entity class changes the stored name (`entity.GetType().Name`, [AclService.cs#L247](../../src/Libraries/Nop.Services/Security/AclService.cs#L247)).

## The repository, method by method

[EntityRepository](../../src/Libraries/Nop.Data/EntityRepository.cs#L14) is the generic gateway (registered as an open generic in [NopDbStartup.cs#L63](../../src/Libraries/Nop.Data/NopDbStartup.cs#L63)). Its exact semantics matter more than its interface:

| Method | Soft-delete aware? | Caching | Events | Transaction |
| --- | --- | --- | --- | --- |
| `GetByIdAsync(id, getCacheKey, includeDeleted = true)` [L122](../../src/Libraries/Nop.Data/EntityRepository.cs#L122) | yes, but **includes deleted by default** | only if `getCacheKey` is passed | — | — |
| `InsertAsync(entity)` [L340-L349](../../src/Libraries/Nop.Data/EntityRepository.cs#L340-L349) | n/a | — | `EntityInsertedAsync` after write | single statement |
| `InsertAsync(list)` [L357-L371](../../src/Libraries/Nop.Data/EntityRepository.cs#L357-L371) | n/a | — | one event per entity, after commit | **yes** (`CreateTransactionScope`) |
| `UpdateAsync(entity)` [L393-L402](../../src/Libraries/Nop.Data/EntityRepository.cs#L393-L402) | n/a | — | `EntityUpdatedAsync` | single statement |
| `UpdateAsync(list)` [L410-L425](../../src/Libraries/Nop.Data/EntityRepository.cs#L410-L425) | n/a | — | one per entity | **no** — see finding F1 |
| `DeleteAsync(entity)` [L433-L452](../../src/Libraries/Nop.Data/EntityRepository.cs#L433-L452) | yes: sets `Deleted = true` and updates | — | `EntityDeletedAsync` | single statement |
| `DeleteAsync(list)` [L460-L487](../../src/Libraries/Nop.Data/EntityRepository.cs#L460-L487) | yes | — | one per entity | **yes** |
| `DeleteAsync(predicate)` [L497-L506](../../src/Libraries/Nop.Data/EntityRepository.cs#L497-L506) | **no — always hard-deletes** | — | **none** | yes |
| `Table` [L525](../../src/Libraries/Nop.Data/EntityRepository.cs#L525) | **no filter** | — | — | — |

Two rows deserve emphasis because they surprise people:

- `includeDeleted` defaults to **true** on `GetByIdAsync`. That is why [ProductController.cs#L132](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L132) re-checks `product.Deleted` by hand: the repository will happily return a soft-deleted product, and the controller is the one who decides that's a 404. (Order history *needs* this default — an old order's product must load.)
- The predicate `DeleteAsync` bypasses both soft delete and events. It's a deliberate bulk escape hatch: guest-customer cleanup uses it to hard-delete `Customer` rows (an `ISoftDeletedEntity`!) by joining a temp table ([CustomerService.cs#L744-L757](../../src/Libraries/Nop.Services/Customers/CustomerService.cs#L744-L757)). Correct for guests; wrong for anything history references.

Underneath, every provider call opens and disposes its own `DataConnection` — e.g. `UpdateEntityAsync` is `using var dataContext = CreateDataConnection(); await dataContext.UpdateAsync(entity);` ([BaseDataProvider.cs#L260-L264](../../src/Libraries/Nop.Data/DataProviders/BaseDataProvider.cs#L260-L264)). There is **no unit of work**: two repository calls in a service are two independent statements unless an ambient `TransactionScope` wraps them.

## Transactions and consistency

The scope factory is worth reading in full because its comment is a small design document ([BaseDataProvider.cs#L503-L534](../../src/Libraries/Nop.Data/DataProviders/BaseDataProvider.cs#L503-L534)): it uses `IsolationLevel.Serializable` on purpose, acknowledges this caused SQL Server deadlocks (issues #6482 and #6681 are cited in the comment), and chooses safety across all supported databases over SQL-Server-specific tuning, inviting you to override it if you know better. `TransactionScopeAsyncFlowOption.Enabled` makes the ambient transaction flow across `await`.

Who uses it? Only the repository's bulk methods, plus two call sites in `Nop.Services` — both in [SyncCodeHelper.cs#L103](../../src/Libraries/Nop.Services/Helpers/SyncCodeHelper.cs#L103) and [#L119](../../src/Libraries/Nop.Services/Helpers/SyncCodeHelper.cs#L119) (counted with `grep -rn "TransactionScope" --include=*.cs src/Libraries/Nop.Services`). Business flows are **not** wrapped in explicit multi-write transactions. `PlaceOrderAsync` performs a sequence of separate writes (order → items → discount/gift-card histories → notes → checkout reset → activity log → event, [L1616-L1644](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1616-L1644)); a mid-sequence crash leaves partial state. Mitigations in the design: writes are ordered so the *anchor row* (Order) lands first, later steps are additive, statuses gate visibility, and support tooling can repair. This is "eventual consistency by ordering + repair" — common in mature systems, rarely admitted in tutorials. Consistency expectation to state in interviews: **per-statement atomicity from the DB, cross-write consistency by design discipline**, and know which flow (checkout) deserves the upgrade to a real transaction/outbox.

Concurrency: no optimistic-concurrency tokens on entities — last write wins (see the lockout counter race, [CustomerRegistrationService.cs#L160](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L160)). Read-then-check-then-write sequences ([max cart items, ShoppingCartService.cs#L1637](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1637)) are race windows the DB never hears about; the durable fix is always *a constraint the DB enforces*.

## Feature trace: an admin product save, down to the cache

| Step | Code | Effect |
| --- | --- | --- |
| 1 | [Admin ProductController.cs#L1253](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1253) | `UpdateProductAsync(product)` |
| 2 | [ProductService.cs#L598-L601](../../src/Libraries/Nop.Services/Catalog/ProductService.cs#L598-L601) | one-line delegate to `_productRepository.UpdateAsync` |
| 3 | [EntityRepository.cs#L393-L402](../../src/Libraries/Nop.Data/EntityRepository.cs#L393-L402) | provider `UpdateEntityAsync`, then `EntityUpdatedAsync` |
| 4 | [BaseDataProvider.cs#L260-L264](../../src/Libraries/Nop.Data/DataProviders/BaseDataProvider.cs#L260-L264) | new connection, `UPDATE`, dispose |
| 5 | [EventPublisher.cs#L20-L52](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L20-L52) | resolve every `IConsumer<EntityUpdatedEvent<Product>>` and call each, swallowing exceptions |
| 6 | [CacheEventConsumer.cs#L44-L53](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53) | evict `Product` by-ids/all prefixes and the by-id key |
| 7 | [ProductCacheEventConsumer.cs#L20-L35](../../src/Libraries/Nop.Services/Catalog/Caching/ProductCacheEventConsumer.cs#L20-L35) | evict product-specific derived caches (prices, homepage, featured, discounts…) |
| 8 | [Admin ProductController.cs#L1290](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1290) → [StoreMappingService.cs#L266-L270](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L266-L270) | flag write via `INopDataProvider` — **no product event** |

Steps 3–7 are the "persistence + invalidation" contract. Step 8 shows the contract is only as strong as the discipline of using the repository.

## Migrations: how to change the schema safely here

Mechanics: FluentMigrator, timestamp-versioned attributes, `UpgradeTo{version}` folders (`UpgradeTo440` … `UpgradeTo500` in [Migrations/](../../src/Libraries/Nop.Data/Migrations)), forward-only ([PriceListMigration](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L7-L20)), idempotent helpers (`CreateTableIfNotExists`, [FluentMigratorExtensions.cs#L127](../../src/Libraries/Nop.Data/Extensions/FluentMigratorExtensions.cs#L127)). Installation schema lives separately in [Migrations/Installation](../../src/Libraries/Nop.Data/Migrations/Installation). Schema migrations are applied **at application startup** on every node ([NopDbStartup.cs#L65-L71](../../src/Libraries/Nop.Data/NopDbStartup.cs#L65-L71)) — see [01-system-map.md](../01-codebase-cartography/01-system-map.md) finding F1.

The safe-change recipe (expand/contract — memorize; it's a top-5 interview topic):

1. **Expand:** add the new nullable column/table in a migration; deploy code that writes both old+new but reads old.
2. **Backfill:** data migration or lazy backfill; verify counts.
3. **Flip reads** to the new column behind the same release or a setting.
4. **Contract:** only releases later, drop the old column — a separate migration users can decline until ready.

Forward-only makes step 4's discipline mandatory: **rollback = restore from backup**, so every migration must be survivable in production the moment it runs. Never mix a long-running data backfill into a schema migration that holds locks — and here, remember the migration runs during boot, so a slow one delays the site coming up.

## Exercise ladder

1. **Goal:** predict a column's SQL type without running a migration.
   **Check:** for `Product.Sku` and `Product.Price`, find the builder line or the type-table entry that decides each (Sku: [ProductBuilder.cs#L23](../../src/Libraries/Nop.Data/Mapping/Builders/Catalog/ProductBuilder.cs#L23), 400 chars nullable; Price: `decimal(18,4)` from L34 of `FluentMigratorExtensions.cs`).
2. **Goal:** prove `GetByIdAsync` returns soft-deleted rows.
   **Check:** cite the default parameter at [EntityRepository.cs#L122](../../src/Libraries/Nop.Data/EntityRepository.cs#L122) and the filter helper at L96-L105; then find one caller that relies on this (order history) and one that must override it (a storefront page).
3. **Goal:** classify every repository write by atomicity.
   **Check:** fill the table above yourself from L340-L506 without looking; compare.
4. **Goal (mid):** find a business flow that would benefit from `CreateTransactionScope`.
   **Check:** pick `PlaceOrderAsync` L1616-L1644 and list which writes must be atomic together and which (e.g. the activity log) should *not* be inside the transaction because an external call or a slow consumer would extend lock time.
5. **Goal (mid) — design:** add `Product.CountryOfOriginId` (FK to Country, admin-editable, storefront-displayed).
   **Check:** your answer lists the builder change (or reliance on auto-mapping), the migration with an explicit `onDelete` (not the cascade default), the release that drops nothing, and the layers touched (builder → entity → service → factory → admin model/validator/view), plus the [ExportImport](../../src/Libraries/Nop.Services/ExportImport) surface.

## Senior findings (verified in source)

**F1 — `UpdateAsync(IList<T>)` is not atomic, while its insert/delete siblings are.** `InsertAsync(list)` and `DeleteAsync(list)` wrap their bulk work in `CreateTransactionScope()` ([L361-L363](../../src/Libraries/Nop.Data/EntityRepository.cs#L361-L363), [L467-L479](../../src/Libraries/Nop.Data/EntityRepository.cs#L467-L479)). `UpdateAsync(list)` does not ([L410-L425](../../src/Libraries/Nop.Data/EntityRepository.cs#L410-L425)), and the provider implements it as a loop of single updates, each on its own connection ([BaseDataProvider.cs#L285-L291](../../src/Libraries/Nop.Data/DataProviders/BaseDataProvider.cs#L285-L291)). A failure on item 7 of 10 leaves items 1–6 updated, and — because events are published only after the provider call returns — *no* update events are raised for those six, so their caches are not invalidated either. Confidence: high (read directly). Caller-side mitigation: wrap the call in an ambient `CreateTransactionScope()`; the scope uses `AsyncFlowOption.Enabled` so it flows into the provider.

**F2 — The predicate `DeleteAsync` is silent.** It hard-deletes regardless of `ISoftDeletedEntity` and publishes no events ([L497-L506](../../src/Libraries/Nop.Data/EntityRepository.cs#L497-L506)). Its callers today are bulk/maintenance paths (guest cleanup, log cleanup, plugin install/uninstall — e.g. [DefaultLogger.cs#L133](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs#L133)), where that is intended. The risk is the *next* caller: an overload of `DeleteAsync` that behaves fundamentally differently from its namesakes is a trap for anyone reading only the method name. A rename (`BulkHardDeleteAsync`) would make the contract visible.

## Debugging scenario: "deleted product still appears in an old order — and in a new cart"

*Report:* a product was deleted in admin. Old orders still show it (expected). But a customer who had it in their cart could still see it in the cart page.

1. **What does delete do?** `DeleteProductAsync` → repository `DeleteAsync(entity)` → `Deleted = true` + update ([ProductService.cs#L510-L513](../../src/Libraries/Nop.Services/Catalog/ProductService.cs#L510-L513), [EntityRepository.cs#L437-L442](../../src/Libraries/Nop.Data/EntityRepository.cs#L437-L442)). The row and its cart items remain.
2. **Why does the cart load it?** The cart model factory loads each line's product with `GetProductByIdAsync(sci.ProductId)` ([ShoppingCartModelFactory.cs#L387](../../src/Presentation/Nop.Web/Factories/ShoppingCartModelFactory.cs#L387)), and `GetByIdAsync` includes deleted rows by default. Nothing removes cart items on product delete in that path.
3. **Where is it caught?** Checkout and add-to-cart run `GetStandardWarningsAsync`, which returns "ShoppingCart.ProductDeleted" for `product.Deleted` ([ShoppingCartService.cs#L339-L344](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L339-L344)). So the system relies on *warnings at use time* rather than *cleanup at delete time* — consistent with soft delete, but visible to users as a stale cart line until they try to proceed.
4. **Decide, don't just patch:** either accept (and make the cart page render the warning prominently) or add a product-deleted consumer that removes cart lines. The second adds a side effect to every product delete — weigh it in [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) terms.

## Verification

2026-10-06: every anchor opened and checked; repository semantics table built by reading `EntityRepository.cs` L96–L525; transaction-scope usage counted with grep; builder count from `find src/Libraries/Nop.Data/Mapping/Builders -name "*Builder.cs" | wc -l` (127). The debugging scenario's cart-page behaviour (step 2) is derived from the factory call and the default `includeDeleted = true`; it was not reproduced in a running store.
