# Fake-Code Contrasts

Eight bad-vs-better pairs. **All snippets are illustrative fake code, not from this repo** — each pair mirrors a real pattern the repo does (or guards against), with the anchor to the real thing. Read the bad one first and articulate what's wrong *before* reading the better one.

## 1. Coupling UI shape to DB shape

```csharp
// Illustrative fake code: not from this repo
public IActionResult Details(int id) {
    var product = _repo.GetById(id);
    return View(product);   // entity straight into the view
}
```

```csharp
// Illustrative fake code: not from this repo
public async Task<IActionResult> Details(int id) {
    var product = await _productService.GetProductByIdAsync(id);
    if (product == null) return NotFound();
    var model = await _productModelFactory.PrepareProductDetailsModelAsync(product);
    return View(model);
}
```

Why it matters: an entity in a view makes every column rename a UI change and leaks fields you never audited (cost prices! `Deleted` flags!). Real version: [ProductController.cs#L198](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L198) + factory. Interview line: "the view model is the contract; the entity is an implementation detail."

## 2. Missing the row-level permission (IDOR)

```csharp
// Illustrative fake code: not from this repo
[CheckPermission(StandardPermission.Catalog.PRODUCTS_CREATE_EDIT_DELETE)]
public async Task<IActionResult> Edit(ProductModel model) {
    var product = await _productService.GetProductByIdAsync(model.Id);
    model.ToEntity(product);            // any product id accepted!
    await _productService.UpdateProductAsync(product);
```

```csharp
// Illustrative fake code: not from this repo
    var currentVendor = await _workContext.GetCurrentVendorAsync();
    if (currentVendor != null && product.VendorId != currentVendor.Id)
        return RedirectToAction("List");   // scope check BEFORE the write
```

The attribute proves capability, never ownership. Real vendor scoping: [Admin ProductController#L969](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L969); concept: [key flow 6](../01-codebase-cartography/05-key-flows.md).

## 3. N+1 loop

```csharp
// Illustrative fake code: not from this repo
foreach (var item in cart)
    total += (await _productService.GetProductByIdAsync(item.ProductId)).Price * item.Quantity;
```

```csharp
// Illustrative fake code: not from this repo
var products = await _productService.GetProductsByIdsAsync(cart.Select(i => i.ProductId).ToArray());
var byId = products.ToDictionary(p => p.Id);
foreach (var item in cart) total += byId[item.ProductId].Price * item.Quantity;
```

The repo provides the batch API ([GetByIdsAsync#L157](../../src/Libraries/Nop.Data/EntityRepository.cs#L157)); a real near-miss lives at [ShoppingCartController.cs#L904-L918](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L904-L918) (mitigated by entity caching — *measure first*).

## 4. Cache write-through that forgets invalidation

```csharp
// Illustrative fake code: not from this repo
public async Task UpdatePriceAsync(int id, decimal price) {
    await _db.ExecuteAsync("UPDATE Product SET Price=@p WHERE Id=@id", ...); // raw SQL
    // nobody evicted ProductById_42; storefront serves old price until restart
}
```

Better: route writes through the repository so the event → [CacheEventConsumer](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53) spine fires. The invariant: **every write path must reach the eviction machinery** — the repo achieves it by having exactly one write path.

## 5. Side effects in the wrong place

```csharp
// Illustrative fake code: not from this repo
public async Task<Order> PlaceOrderAsync(...) {
    var order = await SaveOrderAsync(...);
    await _smtpClient.SendAsync(BuildConfirmationEmail(order)); // inline SMTP!
    return order;   // mail server down => customer can't buy anything
}
```

Better: enqueue durable intent, drain in background — the queued-email outbox ([QueuedMessagesSendTask](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L41-L90)). Rule: a checkout's availability must not depend on any system that isn't required to *record the order*.

## 6. Swallowed errors

```csharp
// Illustrative fake code: not from this repo
try { await consumer.HandleEventAsync(evt); }
catch { }   // silence
```

```csharp
// Illustrative fake code: not from this repo
try { await consumer.HandleEventAsync(evt); }
catch (Exception ex) { await _logger.ErrorAsync(ex.Message, ex); }  // visible, at least
```

The repo does the second ([EventPublisher#L35-L50](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L35-L50)) — and even that is only *logged* visibility. Grading question: for which consumers is log-and-continue wrong outright? (Anything whose failure must fail the operation — then it shouldn't be an event consumer at all.)

## 7. Client-supplied money

```csharp
// Illustrative fake code: not from this repo
public async Task<IActionResult> Checkout(CheckoutModel m) {
    await _orders.PlaceOrderAsync(customerId, m.Items, m.Total); // trusting m.Total
```

Better: server re-derives every amount from persisted state — the repo computes totals in `PreparePlaceOrderDetailsAsync` ([L1602](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1602)) and the only client-entered price is the explicitly-featured "customer entered price," parsed and validated per product ([ShoppingCartController.cs#L855](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L855)). Invariant to quote: *the client chooses what to buy, never what it costs.*

## 8. Casual public-contract change

```csharp
// Illustrative fake code: not from this repo — a "harmless cleanup" PR
- public virtual async Task<IList<string>> AddToCartAsync(Customer customer, Product product,
-     ShoppingCartType shoppingCartType, int storeId, string attributesXml = null, ...)
+ public virtual async Task<CartResult> AddToCartAsync(AddToCartRequest request)  // "nicer API!"
```

In a normal app this is a refactor; here it breaks every plugin and every merchant override built on the old signature ([pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)). Better: additive overload, delegate old→new, obsolete over a deprecation window. The transferable question before any signature change: **who can call this that I can't see?** — for public APIs, package consumers; for this repo, the entire plugin ecosystem.

---

Rubric — Basic: names each flaw. Solid: states the *invariant* each better version protects. Strong: for each pair, can point at where this repo enforces it (anchor from memory) and name one situation where the "bad" version is actually acceptable.
