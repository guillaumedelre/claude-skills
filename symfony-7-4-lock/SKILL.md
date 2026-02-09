---
name: "symfony-7-4-lock"
description: "Symfony 7.4 Lock component reference for creating and managing locks to provide exclusive access to shared resources. Use when working with mutual exclusion, distributed locks, locking mechanisms, or concurrency control. Triggers on: Lock, locking, mutual exclusion, distributed locks, Flock, Redis lock, LockFactory, LockInterface, SharedLockInterface, FlockStore, RedisStore, SemaphoreStore, PostgreSqlStore, CombinedStore, BlockingStoreInterface, Key, lock TTL, expiring locks."
---

# Symfony 7.4 Lock Component

GitHub: https://github.com/symfony/lock
Docs: https://symfony.com/doc/7.4/components/lock.html

## Quick Reference

### Installation

```bash
composer require symfony/lock
```

### Basic Usage

```php
use Symfony\Component\Lock\LockFactory;
use Symfony\Component\Lock\Store\SemaphoreStore;

$store = new SemaphoreStore();
$factory = new LockFactory($store);

$lock = $factory->createLock('pdf-creation');

if ($lock->acquire()) {
    // Exclusive access to "pdf-creation" resource
    $lock->release();
}
```

### Blocking Lock

```php
$lock = $factory->createLock('pdf-creation');
$lock->acquire(true); // Blocks until acquired
```

### Expiring Lock with TTL

```php
$lock = $factory->createLock('pdf-creation', ttl: 30);

if ($lock->acquire()) {
    try {
        while (!$finished) {
            performPartialWork();
            $lock->refresh(); // Reset TTL
        }
    } finally {
        $lock->release();
    }
}
```

### Shared (Readers-Writer) Lock

```php
$lock = $factory->createLock('resource');
$lock->acquireRead();       // Read lock (concurrent readers allowed)
$lock->acquire();           // Promote to write lock
$lock->acquireRead();       // Demote back to read lock
```

### Disable Auto-Release

```php
$lock = $factory->createLock('resource', 3600, autoRelease: false);
```

### Serializing Locks Across Processes

```php
use Symfony\Component\Lock\Key;

$key = new Key('article.'.$article->getId());
$lock = $factory->createLockFromKey($key, 300, autoRelease: false);
$lock->acquire(true);
// Pass $key to another process via message bus
```

## Available Stores

| Store | Scope | Blocking | Expiring | Sharing |
|-------|-------|----------|----------|---------|
| FlockStore | local | yes | no | yes |
| SemaphoreStore | local | yes | no | no |
| RedisStore | remote | retry | yes | yes |
| MemcachedStore | remote | retry | yes | no |
| PdoStore | remote | retry | yes | no |
| DoctrineDbalStore | remote | retry | yes | no |
| PostgreSqlStore | remote | yes | no | yes |
| DoctrineDbalPostgreSqlStore | remote | yes | no | yes |
| MongoDbStore | remote | retry | yes | no |
| ZookeeperStore | remote | retry | no | no |
| CombinedStore | remote | varies | varies | varies |
| DynamoDbStore | remote | retry | yes | no |
| InMemoryStore | local | - | - | - |

### Common Store Setup

```php
// FlockStore (filesystem)
$store = new FlockStore('/var/stores');

// RedisStore
$redis = new \Redis();
$redis->connect('localhost');
$store = new RedisStore($redis);

// PostgreSqlStore (advisory locks, no table needed)
$store = new PostgreSqlStore('pgsql:host=localhost;dbname=app');

// CombinedStore (high availability)
$store = new CombinedStore($stores, new ConsensusStrategy());
```

## LockInterface Methods

- `acquire(bool $blocking = false): bool` - Acquire exclusive lock
- `acquireRead(bool $blocking = false): bool` - Acquire shared read lock
- `release(): void` - Release the lock
- `isAcquired(): bool` - Check if lock is held
- `refresh(float|null $ttl = null): void` - Reset TTL
- `getRemainingLifetime(): float|null` - Seconds until expiry
- `isExpired(): bool` - Check if lock expired

## Full Documentation
See [references/lock.md](references/lock.md) for complete store configuration, strategies, serialization details, and reliability considerations.
