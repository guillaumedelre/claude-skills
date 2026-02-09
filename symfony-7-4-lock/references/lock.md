# Symfony 7.4 Lock Component -- Detailed Reference

Source: https://symfony.com/doc/7.4/components/lock.html
GitHub: https://github.com/symfony/lock

## Core Classes and Interfaces

### LockFactory

Factory for creating lock instances.

```php
use Symfony\Component\Lock\LockFactory;

$factory = new LockFactory($store);
$lock = $factory->createLock('resource-name', ttl: 300, autoRelease: true);
$lock = $factory->createLockFromKey($key, ttl: 300, autoRelease: false);
```

### LockInterface

```php
interface LockInterface
{
    public function acquire(bool $blocking = false): bool;
    public function release(): void;
    public function isAcquired(): bool;
    public function refresh(float|null $ttl = null): void;
    public function getRemainingLifetime(): float|null;
    public function isExpired(): bool;
}
```

### SharedLockInterface

```php
interface SharedLockInterface extends LockInterface
{
    public function acquireRead(bool $blocking = false): bool;
}
```

### Key

Unique identifier for a lock. Used for serialization across processes.

```php
use Symfony\Component\Lock\Key;

$key = new Key('article.'.$articleId);
$lock = $factory->createLockFromKey($key, 300, false);
```

### NoLock

A no-operation lock that always succeeds. Useful for disabling locking in certain environments.

### Store Interfaces

- **PersistingStoreInterface** -- Base contract for all stores.
- **BlockingStoreInterface** -- Stores supporting native blocking (waitAndSave).
- **BlockingSharedLockStoreInterface** -- Blocking + shared lock support.

## Store Configuration Details

### FlockStore

Uses filesystem locks via `flock()`.

```php
use Symfony\Component\Lock\Store\FlockStore;

$store = new FlockStore('/var/stores');
```

Constraints:
- All processes must share the same physical directory and filesystem.
- No TTL support; locks released when process ends.
- Not suitable for multi-server deployments.
- Avoid symlinks and volatile filesystems.

### SemaphoreStore

Uses PHP semaphore functions (`sem_get`, `sem_acquire`).

```php
use Symfony\Component\Lock\Store\SemaphoreStore;

$store = new SemaphoreStore();
```

Constraints:
- Local only; all processes on the same machine.
- Kernel auto-releases on process termination.
- With systemd, set `RemoveIPC=off` for system users.

### RedisStore

```php
use Symfony\Component\Lock\Store\RedisStore;

$redis = new \Redis();
$redis->connect('localhost');
$store = new RedisStore($redis);
```

Accepted client types: `\Redis`, `\RedisArray`, `\RedisCluster`, `\Relay\Relay`, `\Relay\Cluster`, `\Predis\ClientInterface`.

Constraints:
- Not persisted by default; restart loses all locks.
- Delay Redis restart by the longest lock TTL.
- Avoid round-robin DNS or load balancers.

### MemcachedStore

```php
use Symfony\Component\Lock\Store\MemcachedStore;

$memcached = new \Memcached();
$memcached->addServer('localhost', 11211);
$store = new MemcachedStore($memcached);
```

Constraints:
- Minimum TTL is 1 second.
- Locks lost on restart; disable `flush()`.

### PdoStore

```php
use Symfony\Component\Lock\Store\PdoStore;

$store = new PdoStore('mysql:host=127.0.0.1;dbname=app', [
    'db_username' => 'user',
    'db_password' => 'pass',
]);
$store->createTable(); // Auto-create lock table
```

### DoctrineDbalStore

```php
use Symfony\Component\Lock\Store\DoctrineDbalStore;

$store = new DoctrineDbalStore('mysql://user:pass@127.0.0.1/app');
$store->configureSchema($schema, fn() => true);
```

### PostgreSqlStore

Uses PostgreSQL advisory locks. No table needed, no expiration required.

```php
use Symfony\Component\Lock\Store\PostgreSqlStore;

$store = new PostgreSqlStore('pgsql:host=localhost;port=5432;dbname=app', [
    'db_username' => 'user',
    'db_password' => 'pass',
]);
```

Advantages:
- Native blocking support.
- Native shared lock support.
- Auto-released at session end.

### DoctrineDbalPostgreSqlStore

PostgreSQL advisory locks via Doctrine DBAL.

```php
use Symfony\Component\Lock\Store\DoctrineDbalPostgreSqlStore;

$store = new DoctrineDbalPostgreSqlStore('postgresql+advisory://user:pass@127.0.0.1:5432/lock');
```

### MongoDbStore

```php
use Symfony\Component\Lock\Store\MongoDbStore;

$store = new MongoDbStore('mongodb://localhost/database?collection=lock', [
    'gcProbability' => 0.001,
    'database' => 'myapp',
    'collection' => 'lock',
]);
```

Options: `gcProbability` (float 0.0-1.0, default 0.001), `database`, `collection`, `uriOptions`, `driverOptions`.

Constraints:
- Resource name field limited to 1024 bytes.
- Requires TTL index for expiration.

### ZookeeperStore

```php
use Symfony\Component\Lock\Store\ZookeeperStore;

$zookeeper = new \Zookeeper('localhost:2181');
$store = new ZookeeperStore($zookeeper);
```

Ephemeral nodes auto-released; no TTL required.

### DynamoDbStore

```bash
composer require symfony/amazon-dynamo-db-lock
```

```php
use Symfony\Component\Lock\Bridge\DynamoDb\Store\DynamoDbStore;

$store = new DynamoDbStore('dynamodb://default/lock');
$store->createTable();
```

### CombinedStore

High-availability store using multiple backends with a consensus strategy.

```php
use Symfony\Component\Lock\Store\CombinedStore;
use Symfony\Component\Lock\Strategy\ConsensusStrategy;
use Symfony\Component\Lock\Strategy\UnanimousStrategy;

$store = new CombinedStore($stores, new ConsensusStrategy()); // Majority wins
$store = new CombinedStore($stores, new UnanimousStrategy()); // All must agree
```

Use at least 3 servers with ConsensusStrategy.

### InMemoryStore

In-process only. Useful for testing.

### NullStore

No-op store. Always succeeds.

## Strategies (for CombinedStore)

- **ConsensusStrategy** -- Lock acquired if majority of stores agree (quorum).
- **UnanimousStrategy** -- Lock acquired only if all stores agree.

## Robust Lock Pattern

```php
$lock = $factory->createLock('pdf-creation', 30);

if (!$lock->acquire()) {
    return;
}

try {
    while (!$finished) {
        if ($lock->getRemainingLifetime() <= 5) {
            if ($lock->isExpired()) {
                throw new \RuntimeException('Lock lost');
            }
            $lock->refresh();
        }
        performTask();
    }
} finally {
    $lock->release();
}
```

## Reliability Considerations

### Remote Stores
- Use unique tokens internally to prevent false ownership.
- All concurrent processes must use the same server (avoid load balancers).

### Expiring Stores
- Set TTL with buffer for network and clock drift.
- Monitor `isExpired()` and `getRemainingLifetime()`.
- NTP adjustments can cause early expiration.

### Clock Synchronization
- All nodes must have synchronized clocks.
- Add extra time to TTL to account for drift.

### Store-Specific Notes
- **Memcached**: Disable `flush()` to avoid accidental lock removal.
- **MongoDB**: Resource name limited to 1024 bytes; TTL index required.
- **PostgreSQL**: Locks released on connection loss or session end.
- **Redis/Memcached**: Not persisted; restart loses locks.
- **Semaphores**: Set `RemoveIPC=off` in systemd for system users.
