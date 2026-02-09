# Symfony 7.4 Semaphore Component - Complete Reference

## Overview

The Semaphore Component manages **semaphores**, a mechanism to provide controlled concurrent access to a shared resource. Unlike locks, which only allow one process to access a resource, semaphores allow **multiple processes** to access a resource simultaneously, up to a specified limit.

## Installation

```bash
composer require symfony/semaphore
```

If installing outside a Symfony application, require the Composer autoloader:

```php
require 'vendor/autoload.php';
```

## Core Concepts

### Semaphore vs Lock

| Feature | Semaphore | Lock |
|---------|-----------|------|
| Concurrent access | Multiple (configurable) | Single only |
| Use case | Rate limiting, resource pooling | Exclusive access |
| Example | "Allow 5 concurrent PDF generations" | "Only one process can update this file" |

## API Reference

### SemaphoreFactory

The factory class for creating semaphore instances.

```php
use Symfony\Component\Semaphore\SemaphoreFactory;
use Symfony\Component\Semaphore\Store\RedisStore;

$store = new RedisStore($redis);
$factory = new SemaphoreFactory($store);
```

#### createSemaphore()

```php
public function createSemaphore(
    string $resource,
    int $limit,
    ?float $ttl = 300.0,
    bool $autoRelease = true
): SemaphoreInterface
```

**Parameters:**
- `$resource` - Arbitrary string identifying the locked resource
- `$limit` - Maximum number of processes allowed concurrent access
- `$ttl` - Time-to-live in seconds (default: 300.0)
- `$autoRelease` - Whether to auto-release on destruction (default: true)

**Example:**

```php
// Allow up to 3 concurrent processes for 5 minutes
$semaphore = $factory->createSemaphore('api-calls', 3, 300.0, true);
```

### SemaphoreInterface

The contract that all semaphore implementations must follow.

```php
interface SemaphoreInterface
{
    public function acquire(): bool;
    public function refresh(?float $ttl = null): void;
    public function isAcquired(): bool;
    public function release(): void;
    public function isExpired(): bool;
    public function getRemainingLifetime(): ?float;
}
```

#### acquire()

Attempts to acquire the semaphore.

```php
public function acquire(): bool
```

**Returns:** `true` if semaphore was acquired, `false` if limit reached.

**Behavior:**
- Safe to call repeatedly, even if already acquired
- Returns `true` immediately if already holding the semaphore
- Returns `false` if the maximum number of processes are already using the resource

**Example:**

```php
if ($semaphore->acquire()) {
    // Successfully acquired - proceed with work
    $this->processJob();
    $semaphore->release();
} else {
    // Could not acquire - too many concurrent processes
    throw new \RuntimeException('Resource busy, try again later');
}
```

#### release()

Releases the semaphore.

```php
public function release(): void
```

**Behavior:**
- Frees the semaphore slot for other processes
- Safe to call even if not acquired (no-op)
- Called automatically on object destruction if `autoRelease` is true

#### refresh()

Extends the TTL of the semaphore.

```php
public function refresh(?float $ttl = null): void
```

**Parameters:**
- `$ttl` - New TTL in seconds (null uses original TTL)

**Example:**

```php
$semaphore = $factory->createSemaphore('long-job', 2, 60.0);
$semaphore->acquire();

// Job is taking longer than expected, extend the TTL
$semaphore->refresh(120.0);  // Extend to 2 minutes
```

#### isAcquired()

Checks if this instance currently holds the semaphore.

```php
public function isAcquired(): bool
```

#### isExpired()

Checks if the semaphore has expired.

```php
public function isExpired(): bool
```

#### getRemainingLifetime()

Gets the remaining TTL.

```php
public function getRemainingLifetime(): ?float
```

**Returns:** Remaining seconds, or `null` if not acquired.

## Storage Backends

### RedisStore

Redis-backed semaphore storage. Recommended for production use.

```php
use Symfony\Component\Semaphore\Store\RedisStore;

// Using ext-redis
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);
$store = new RedisStore($redis);

// Using Predis
$redis = new Predis\Client('tcp://127.0.0.1:6379');
$store = new RedisStore($redis);
```

### PersistingStoreInterface

The contract for storage backends.

```php
interface PersistingStoreInterface
{
    public function save(Key $key, float $ttlInSecond): void;
    public function delete(Key $key): void;
    public function exists(Key $key): bool;
}
```

## Complete Usage Examples

### Basic Rate Limiting

```php
use Symfony\Component\Semaphore\SemaphoreFactory;
use Symfony\Component\Semaphore\Store\RedisStore;

$redis = new Redis();
$redis->connect('127.0.0.1');
$store = new RedisStore($redis);
$factory = new SemaphoreFactory($store);

// Allow max 5 concurrent API calls
$semaphore = $factory->createSemaphore('external-api', 5);

if ($semaphore->acquire()) {
    try {
        $response = $this->apiClient->makeRequest();
        return $response;
    } finally {
        $semaphore->release();
    }
}

throw new TooManyRequestsException('API rate limit exceeded');
```

### PDF Generation with Concurrency Control

```php
class PdfGeneratorService
{
    private SemaphoreFactory $semaphoreFactory;

    public function __construct(SemaphoreFactory $semaphoreFactory)
    {
        $this->semaphoreFactory = $semaphoreFactory;
    }

    public function generateInvoice(Invoice $invoice): string
    {
        // Only allow 2 concurrent PDF generations (memory-intensive)
        $semaphore = $this->semaphoreFactory->createSemaphore(
            'pdf-invoice-generation',
            2,
            120.0  // 2 minute TTL
        );

        if (!$semaphore->acquire()) {
            throw new \RuntimeException('PDF generation busy, please retry');
        }

        try {
            return $this->doGeneratePdf($invoice);
        } finally {
            $semaphore->release();
        }
    }
}
```

### Long-Running Jobs with TTL Refresh

```php
$semaphore = $factory->createSemaphore('data-import', 1, 60.0);

if ($semaphore->acquire()) {
    try {
        foreach ($batches as $batch) {
            $this->processBatch($batch);

            // Refresh TTL after each batch to prevent expiration
            $semaphore->refresh();
        }
    } finally {
        $semaphore->release();
    }
}
```

### Cross-Request Locking (Disable Auto-Release)

```php
// First request: acquire and store reference
$semaphore = $factory->createSemaphore(
    'deployment-lock',
    1,
    3600.0,  // 1 hour TTL
    false    // Disable auto-release
);

if ($semaphore->acquire()) {
    // Store semaphore key in session/database for later release
    $this->storeKey($semaphore->getKey());
    $this->startDeployment();
}

// Later request: release manually
$key = $this->retrieveStoredKey();
$semaphore = $factory->createSemaphoreFromKey($key);
$semaphore->release();
```

### Symfony Service Configuration

```yaml
# config/services.yaml
services:
    Redis:
        class: Redis
        calls:
            - connect: ['%env(REDIS_HOST)%', '%env(int:REDIS_PORT)%']

    Symfony\Component\Semaphore\Store\RedisStore:
        arguments: ['@Redis']

    Symfony\Component\Semaphore\SemaphoreFactory:
        arguments: ['@Symfony\Component\Semaphore\Store\RedisStore']
```

## Best Practices

### 1. Always Use try/finally

Ensure semaphores are released even if exceptions occur:

```php
if ($semaphore->acquire()) {
    try {
        $this->doWork();
    } finally {
        $semaphore->release();
    }
}
```

### 2. Choose Appropriate TTL

- Set TTL slightly longer than expected operation time
- Use `refresh()` for variable-length operations
- Consider failure scenarios when setting TTL

### 3. Share Semaphore Instances

The component distinguishes semaphore instances. For multiple services using the same resource:

```php
// GOOD: Share the same instance
$semaphore = $factory->createSemaphore('resource', 2);
$serviceA->setSemaphore($semaphore);
$serviceB->setSemaphore($semaphore);

// BAD: Creates separate instances that don't coordinate
$serviceA->setSemaphore($factory->createSemaphore('resource', 2));
$serviceB->setSemaphore($factory->createSemaphore('resource', 2));
```

### 4. Handle Acquisition Failure Gracefully

```php
if (!$semaphore->acquire()) {
    // Options:
    // 1. Retry with backoff
    // 2. Queue for later processing
    // 3. Return 503 Service Unavailable
    // 4. Use fallback mechanism
}
```

### 5. Monitor Semaphore Usage

Track acquisition success/failure rates to:
- Identify bottlenecks
- Tune concurrency limits
- Detect resource contention

## Error Handling

### Common Exceptions

```php
use Symfony\Component\Semaphore\Exception\SemaphoreAcquiringException;
use Symfony\Component\Semaphore\Exception\SemaphoreReleasingException;

try {
    $semaphore->acquire();
} catch (SemaphoreAcquiringException $e) {
    // Storage backend error during acquire
}

try {
    $semaphore->release();
} catch (SemaphoreReleasingException $e) {
    // Storage backend error during release
}
```

## Resources

- **GitHub Repository**: https://github.com/symfony/semaphore
- **Official Documentation**: https://symfony.com/doc/7.4/components/semaphore.html
- **License**: MIT
