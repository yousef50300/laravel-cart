# laravel-cart

A driver-based shopping cart package for Laravel, built as a **portfolio/experimental project** to explore clean OOP design patterns — not intended for production use as-is.

> The goal was not to build a complete cart package. The goal was to think through *how* a cart package should be designed, document the decisions, and be honest about what's missing.

---

## What this demonstrates

| Pattern | Where |
|---|---|
| **Value Object** | `CartItem` is immutable — mutations return new instances via `withQuantity()` / `withPrice()` |
| **Driver / Manager pattern** | `CartManager` mirrors Laravel's `CacheManager` — swap drivers without touching business logic |
| **Interface segregation** | Drivers only handle persistence. Validation and sync live in `CartItemSyncService` |
| **Single Responsibility** | Each class has one reason to change |
| **Facade over complexity** | `Cart::driver('database')->add(...)` hides the resolution logic from the caller |

---

## Architecture

```
src/
├── Cart.php                        # Facade — thin public API
├── CartItem.php                    # Immutable Value Object
├── CartManager.php                 # Resolves drivers (session / database / custom)
├── CartServiceProvider.php
├── Contracts/
│   └── CartDriverInterface.php     # Contract every driver must satisfy
├── Drivers/
│   ├── SessionCartDriver.php       # Guest carts
│   └── DatabaseCartDriver.php      # Authenticated / persistent carts
├── Exceptions/
│   └── CartItemUnavailableException.php
└── Support/
    └── CartItemSyncService.php     # Re-fetches price + stock before checkout
```

---

## Usage

### Basic

```php
use Yousef\Cart\CartItem;
use Yousef\Cart\Cart;

$item = new CartItem(
    id:       'prod-42',
    name:     'Wireless Keyboard',
    price:    149.99,
    quantity: 2,
);

Cart::driver()->add($item);
Cart::driver()->total(); // 299.98
```

### Switching drivers

```php
// Guest — session
Cart::driver('session')->add($item);

// Authenticated — database
Cart::driver('database')->add($item);
```

### Price & stock sync before checkout

```php
use Yousef\Cart\Support\CartItemSyncService;
use Yousef\Cart\Exceptions\CartItemUnavailableException;

try {
    app(CartItemSyncService::class)->sync(Cart::driver());
} catch (CartItemUnavailableException $e) {
    // $e->items = ['Wireless Keyboard']
    return back()->withErrors($e->getMessage());
}

// Prices are now fresh — safe to proceed to payment
```

### Register a custom driver

```php
// AppServiceProvider::boot()
Cart::extend('redis', fn ($app) => new RedisCartDriver($app->make('redis')));
```

---

## Known limitations & what I'd add in production

This is an intentional list — these are not oversights, they're the next layer of complexity I chose not to build in order to keep the design clean and focused.

### 1. No coupon / discount system
The current `total()` is a flat sum of subtotals. A real implementation would introduce a `DiscountPipeline` that applies a chain of `DiscountInterface` objects (percentage off, fixed amount, BOGO) before returning the total. This maps cleanly onto Laravel's Pipeline pattern.

### 2. No cart merge on login
When a guest logs in, their session cart should merge into their database cart. This requires a `CartMergeService` that compares item IDs across both drivers and resolves conflicts (e.g. take the higher quantity). Currently the two drivers are fully independent.

### 3. `CartItemSyncService` hits the DB once per item
For a cart with 20 items, that's 20 queries. The correct approach is a single `whereIn` query followed by a keyed collection lookup — a straightforward optimisation I intentionally left as-is to keep the sync logic readable in isolation.

### 4. No event system
Production carts emit events: `ItemAdded`, `ItemRemoved`, `CartCleared`, `CartSynced`. Listeners can then update recommendation engines, analytics, or loyalty points. Wiring this into the current design would mean firing events inside each driver method — clean to add, just not in scope here.

### 5. No multi-tenancy / multi-instance support
`CartManager` resolves one cart per user/session. A marketplace with multiple sellers per cart, or a system with named wishlists alongside the main cart, would need a different resolution strategy.

### 6. `DatabaseCartDriver` has no soft-delete / expiry
Abandoned carts stay in the table forever. A real implementation would add an `expires_at` column and an artisan command (or scheduled job) to purge stale carts.

### 7. No test suite
I wrote the contracts and classes with testability in mind (constructor-injected dependencies, no static calls inside classes), but did not include tests in this repo. A proper suite would cover: driver isolation, sync edge cases (out-of-stock, price change, deleted product), and the manager's driver resolution.

---

## Installation (local / for review)

```bash
git clone https://github.com/yousef/laravel-cart
cd laravel-cart
composer install
```

To use in a local Laravel app via path repository:

```json
"repositories": [
    { "type": "path", "url": "../laravel-cart" }
],
"require": {
    "yousef/laravel-cart": "*"
}
```

```bash
php artisan vendor:publish --tag=cart-config
php artisan vendor:publish --tag=cart-migrations
php artisan migrate
```

---

## What I'd explore next

- Implementing the `DiscountPipeline` using Laravel's `Pipeline` facade
- A `CartMergeService` with conflict resolution strategies
- Replacing the per-item sync queries with a batched `whereIn` approach
- Adding a Redis driver as a third option (TTL-based expiry built-in)
