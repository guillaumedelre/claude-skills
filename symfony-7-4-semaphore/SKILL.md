---
name: "symfony-7-4-semaphore"
description: "Symfony 7.4 Semaphore component for concurrency limiting and resource access control. Triggers on: Semaphore, concurrency limiting, weighted semaphore, Redis semaphore, SemaphoreFactory, SemaphoreInterface, RedisStore, acquire, release, rate limiting, concurrent processes."
---

# Symfony 7.4 Semaphore Component

## Overview

The Symfony Semaphore component manages semaphores - mechanisms to control concurrent access to shared resources. Unlike locks (which allow only one process), semaphores allow **multiple processes** to access a resource simultaneously, up to a configurable limit.

## Quick Reference

### Installation

```bash
composer require symfony/semaphore
```

### Basic Usage

```php
use Symfony\Component\Semaphore\SemaphoreFactory;
use Symfony\Component\Semaphore\Store\RedisStore;

// Setup Redis store
$redis = new Redis();
$redis->connect('127.0.0.1');
$store = new RedisStore($redis);
$factory = new SemaphoreFactory($store);

// Create semaphore: resource name + max concurrent processes
$semaphore = $factory->createSemaphore('pdf-generation', 2);

// Acquire and use
if ($semaphore->acquire()) {
    try {
        // Only up to 2 processes can be here simultaneously
        $this->generatePdf();
    } finally {
        $semaphore->release();
    }
}
```

### Key Interfaces

| Class/Interface | Purpose |
|-----------------|---------|
| `SemaphoreFactory` | Creates semaphore instances |
| `SemaphoreInterface` | Contract for semaphore operations |
| `RedisStore` | Redis-backed persistence |
| `PersistingStoreInterface` | Contract for storage backends |

### acquire() vs release()

```php
$semaphore->acquire();  // Returns true if acquired, false if limit reached
$semaphore->release();  // Releases the semaphore (also auto-releases on destruction)
```

### Disable Auto-Release (for cross-request locking)

```php
$semaphore = $factory->createSemaphore(
    'resource-name',
    2,
    300.0,  // TTL in seconds
    false   // Disable auto-release
);
```

## Common Use Cases

- **Rate limiting**: Control concurrent access to expensive operations
- **Resource management**: Limit concurrent database connections or API calls
- **Batch processing**: Control parallelism in background jobs
- **Invoice generation**: Limit concurrent PDF generations

## Full Documentation
- GitHub: https://github.com/symfony/semaphore
- Docs: https://symfony.com/doc/7.4/components/semaphore.html
- **Full Reference**: See `references/semaphore.md` for complete documentation
