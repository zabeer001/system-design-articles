## System Design (Low-Level) — Laravel Object Lifecycle, Service Container & Octane

When we create an object manually:

```php
function payment()
{
    $stripe = new StripePayment();
}

payment();
payment();
```

Every time `new StripePayment()` is called, a **new object** is created.

```text
1st call → StripePayment Object A
2nd call → StripePayment Object B

Object A !== Object B
```

The two objects have different references.

Now let's move to the **Laravel Service Container**.

### `bind()`

```php
$this->app->bind(
    PaymentGateway::class,
    StripePayment::class
);
```

Every time we resolve it, Laravel creates a new instance.

```php
$a = resolve(PaymentGateway::class);
$b = resolve(PaymentGateway::class);

$a === $b; // false
```

So `bind()` essentially provides **transient behavior**.

### `scoped()`

```php
$this->app->scoped(TenantContext::class);
```

Within the same request/lifecycle:

```text
resolve() → Object A
resolve() → Object A
resolve() → Object A
```

The same instance is reused.

But when a new request comes in:

```text
Request 1 → Object A
Request 2 → Object B
Request 3 → Object C
```

This is the important behavior of `scoped()`.

### `singleton()`

With a singleton, the container keeps returning the same instance for the lifetime of that container.

In a traditional PHP-FPM Laravel application, a request normally gets an isolated application lifecycle. Once the request finishes, that in-memory application state goes away.

But **Laravel Octane changes the situation**.

With Swoole or RoadRunner, Octane keeps the application in memory and workers can serve multiple requests.

```text
Octane Worker
│
├── Request A
├── Request B
├── Request C
└── ...
```

Now imagine we're building a **Multi-DB, Multi-Tenant SaaS**.

Tenant A sends a request, and we store tenant-specific database information inside a long-lived singleton object.

The next request belongs to Tenant B.

If the previous tenant-specific state is not properly reset, there is a risk that Tenant A's database or connection state could leak into Tenant B's request.

That becomes a serious **data-isolation problem**.

So with **Octane + Multi-Tenant architecture**, keeping tenant-specific mutable state inside a long-lived singleton can be dangerous.

A better approach is:

```text
Request
   ↓
Identify Tenant
   ↓
Scoped TenantContext
   ↓
Configure Tenant DB
   ↓
Purge / Reconnect
   ↓
Execute Query
```

One important correction:

We don't necessarily need to make Laravel's entire **Database Manager scoped**.

The important part is keeping **tenant-specific context/state request-scoped** and correctly configuring, purging, and reconnecting the tenant database connection for each tenant request.

### Simple Mental Model

```text
new Class()
→ New object every time

bind()
→ New object every resolution

scoped()
→ Same object within one request/lifecycle

singleton()
→ Same object for the container lifetime
```

This is one of the interesting parts of learning **Dependency Injection and object lifecycle management**: DI isn't only about avoiding `new`. The lifetime of the resolved object can become an architectural decision, especially when moving from traditional PHP-FPM to long-running workers such as Laravel Octane.
