# Writing Tests Here — Recipes

Six recipes using the repo's actual helpers. Skeletons are **illustrative fake code (not from this repo)** but follow the suite's real conventions: inherit the right base, resolve via `GetService<T>`, restore mutated state. Commands are __inferred__ NUnit filters.

Base classes: most service tests extend `ServiceTest` (which extends [BaseNopTest](../../src/Tests/Nop.Tests/BaseNopTest.cs#L98)); resolve anything with `GetService<T>()` ([L572](../../src/Tests/Nop.Tests/BaseNopTest.cs#L572)); an admin customer helper exists ([L668](../../src/Tests/Nop.Tests/BaseNopTest.cs#L668)).

## Recipe 1 — happy path through a service

```csharp
// Illustrative fake code: not from this repo
public class MyCartTests : ServiceTest {
    private IShoppingCartService _cartService;
    private IProductService _productService;

    [OneTimeSetUp]
    public void SetUp() {
        _cartService = GetService<IShoppingCartService>();
        _productService = GetService<IProductService>();
    }

    [Test]
    public async Task CanAddSimpleProductToCart() {
        var customer = await GetService<ICustomerService>().GetCustomerByEmailAsync(NopTestsDefaults.AdminEmail);
        var product = await _productService.GetProductBySkuAsync("COMP_CUST_PC"); // seed data sku
        var warnings = await _cartService.AddToCartAsync(customer, product,
            ShoppingCartType.ShoppingCart, storeId: 1);
        warnings.Should().BeEmpty();
        // cleanup: remove the item so later tests see pristine state
    }
}
```

Run: `dotnet test src --filter "FullyQualifiedName~MyCartTests"`

## Recipe 2 — validation failure (assert the warning, not an exception)

Target: quantity ≤ 0 → warning, per [AddToCartAsync#L1578-L1582](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1578-L1582).

```csharp
// Illustrative fake code: not from this repo
var warnings = await _cartService.AddToCartAsync(customer, product,
    ShoppingCartType.ShoppingCart, 1, quantity: 0);
warnings.Should().NotBeEmpty();          // style point: assert on the outcome,
                                          // not the exact string — strings are localized data
```

## Recipe 3 — permission failure

Target: cart disabled for a role → "Shopping cart is disabled" branch ([#L1560-L1564](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1560-L1564)). Arrange by removing the `ENABLE_SHOPPING_CART` permission mapping for the customer's role via [IPermissionService](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L374-L395), act, assert one warning, **restore the mapping in finally/teardown** — permission mappings are cached per role, so also beware [the cached authorize path](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L331): clearing cache may be required (note: no `PermissionService` tests exist yet — verified — so you're breaking new ground; that makes this recipe double as [a contribution](../06-contribution-practice/01-good-first-tickets.md)).

## Recipe 4 — cross-store isolation (the "tenant rejection" test)

Target: [StoreMappingService.AuthorizeAsync#L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L221). Arrange: pick a seed product, set `LimitedToStores = true`, insert a mapping to store 2 only. Assert `AuthorizeAsync(product, storeId: 1)` is false and `(product, 2)` is true. Restore both changes. This is ticket 5 in [good-first-tickets](../06-contribution-practice/01-good-first-tickets.md) — writing it teaches the multi-store model faster than reading ever will.

## Recipe 5 — async side effect (queued email exists after an action)

Target: business action → outbox row (not: email actually sent — that's the task's job, and SMTP is faked anyway).

```csharp
// Illustrative fake code: not from this repo
var before = (await _queuedEmailService.SearchEmailsAsync(null, null, null, null,
    loadNotSentItemsOnly: true, true, 3, false, 0, int.MaxValue)).Count;
await _workflowMessageService.SendCustomerPasswordRecoveryMessageAsync(customer, languageId: 1);
var after = (await _queuedEmailService.SearchEmailsAsync(...)).Count;
after.Should().Be(before + 1);
```

Lesson encoded: test the *durable intent*, not the delivery — delivery has its own component and its own tests.

## Recipe 6 — cache invalidation regression

Target: the write-spine guarantee ([CacheEventConsumer](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53)). Read an entity through a cached path (e.g. `GetProductByIdAsync` twice — second hit cached), update it via its service, read again and assert the new value is visible. If this test fails, someone bypassed the write spine — exactly the regression class from [debugging scenario 1](03-systematic-debugging.md).

---

General rules for this suite: match existing naming (`Can...`, `Should...` — see [ProductServiceTests](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L63)); assert outcomes not localized strings; restore every mutation; no `[Test]` ordering dependencies; prefer seed data over creating entities when read-only suffices. Interview angle: recipes 3–6 are the answers to "how would you test authorization / multi-tenancy / async side effects / caching" — with a real codebase to cite.
