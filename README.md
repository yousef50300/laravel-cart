# laravel-cart

A driver-based cart package for Laravel demonstrating clean OOP design:
**Value Objects**, **Interface segregation**, **Manager pattern**, and **immutability**.

> Built as a portfolio project — not published on Packagist.

---

## Design decisions

| Concept | Implementation |
|---|---|
| Immutable item | `CartItem` is a `final` Value Object — mutations return new instances |
| Driver pattern | `CartManager` mirrors Laravel's built-in `CacheManager` / `QueueManager` |
| Interface segregation | Drivers only handle persistence — validation lives in `CartItemSyncService` |
| Price integrity | `CartItemSyncService::sync()` re-fetches price + stock before checkout |
| Extensibility | Register custom drivers via `CartManager::extend()` |

---

## Structure

```
src/
├── Cart.php                        # Facade
├── CartItem.php                    # Value Object
├── CartManager.php                 # Driver resolver
├── CartServiceProvider.php
├── Contracts/
│   └── CartDriverInterface.php
├── Drivers/
│   ├── SessionCartDriver.php
│   └── DatabaseCartDriver.php
├── Exceptions/
│   └── CartItemUnavailableException.php
└── Support/
    └── CartItemSyncService.php
```

---

## Usage

### Add an item

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
```

### Switch driver at runtime

```php
// Session (default — guests)
Cart::driver('session')->add($item);

// Database (authenticated users)
Cart::driver('database')->add($item);
```

### Sync items before checkout

```php
use Yousef\Cart\Support\CartItemSyncService;
use Yousef\Cart\Exceptions\CartItemUnavailableException;

$sync = app(CartItemSyncService::class);

try {
    $sync->sync(Cart::driver());
} catch (CartItemUnavailableException $e) {
    // $e->items = ['Wireless Keyboard', ...]
    return back()->withErrors($e->getMessage());
}

// Safe to proceed to payment
```

### Register a custom driver

```php
// In a ServiceProvider
Cart::extend('redis', function ($app) {
    return new RedisCartDriver($app->make('redis'));
});
```

---

## Configuration

```bash
php artisan vendor:publish --tag=cart-config
php artisan vendor:publish --tag=cart-migrations
php artisan migrate
```

```php
// config/cart.php
'driver'         => env('CART_DRIVER', 'session'),
'session_key'    => 'cart.items',
'table'          => 'cart_items',
'products_table' => 'products',
```

---

## Running tests

```bash
composer install
./vendor/bin/pest
```
