# Symfony 7.4 Cache Component - Complete Reference

GitHub: https://github.com/symfony/cache
Docs: https://symfony.com/doc/7.4/components/cache.html
License: MIT

## Installation

```bash
composer require symfony/cache
```

## Two Approaches: Cache Contracts vs PSR-6

### Cache Contracts (Recommended)

Simpler API with built-in stampede protection via locking. Uses `get()` and `delete()` methods.

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Contracts\Cache\ItemInterface;

$cache = new FilesystemAdapter();

$value = $cache->get('my_cache_key', function (ItemInterface $item): string {
    $item->expiresAfter(3600);

    // Heavy computation or HTTP request
    return 'foobar';
});

echo $value; // 'foobar'

$cache->delete('my_cache_key');
```

### PSR-6 Generic Caching

More formal/structured approach implementing the PSR-6 standard interface.

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter();

$productsCount = $cache->getItem('stats.products_count');

if (!$productsCount->isHit()) {
    $productsCount->set(4711);
    $cache->save($productsCount);
}

$total = $productsCount->get();

// Delete
$cache->deleteItem('stats.products_count');
```

---

## Cache Items

Cache items are key/value pairs represented by the `CacheItem` class.

### Keys

Must contain only: letters (A-Z, a-z), numbers (0-9), underscore (`_`), and dot (`.`). Symbols `{ } ( ) / \ @ :` are reserved.

### Values

Any PHP-serializable data: strings, integers, floats, booleans, null, arrays, objects.

### Expiration

```php
// Relative (seconds or DateInterval)
$item->expiresAfter(60);
$item->expiresAfter(DateInterval::createFromDateString('1 hour'));

// Absolute
$item->expiresAt(new \DateTime('tomorrow'));
```

Default: stored permanently (0 = indefinite, adapter-dependent).

### Hit Detection

```php
$item = $cache->getItem('latest_news');

if (!$item->isHit()) {
    // Cache miss: compute value
    $news = computeNews();
    $item->set($news);
    $cache->save($item);
} else {
    $news = $item->get();
}
```

PSR-6 always returns a `CacheItemInterface` (never null), so `false` and `null` can be safely stored.

---

## Cache Pools (All Adapters)

All adapters accept `namespace`, `defaultLifetime`, and adapter-specific options.

### FilesystemAdapter

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter(
    namespace: 'my_app',
    defaultLifetime: 0,
    directory: null // system temp dir
);
```

### ArrayAdapter (In-memory, no persistence)

```php
use Symfony\Component\Cache\Adapter\ArrayAdapter;

$cache = new ArrayAdapter();
```

### RedisAdapter

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;

$redis = RedisAdapter::createConnection('redis://localhost:6379');
$cache = new RedisAdapter($redis, 'namespace', 3600);
```

### MemcachedAdapter

```php
use Symfony\Component\Cache\Adapter\MemcachedAdapter;

$client = MemcachedAdapter::createConnection('memcached://localhost');
$cache = new MemcachedAdapter($client, 'namespace', 3600);
```

### ApcuAdapter

```php
use Symfony\Component\Cache\Adapter\ApcuAdapter;

$cache = new ApcuAdapter('namespace', 0);
```

### PdoAdapter

```php
use Symfony\Component\Cache\Adapter\PdoAdapter;

$pdo = new \PDO('sqlite://path/to/db.sqlite');
$cache = new PdoAdapter($pdo, 'namespace', 0);
```

### DoctrineDbalAdapter

```php
use Symfony\Component\Cache\Adapter\DoctrineDbalAdapter;

$connection = DriverManager::getConnection([...]);
$cache = new DoctrineDbalAdapter($connection, 'namespace', 0);
```

### ChainAdapter (Multi-tier)

```php
use Symfony\Component\Cache\Adapter\ChainAdapter;
use Symfony\Component\Cache\Adapter\ApcuAdapter;
use Symfony\Component\Cache\Adapter\RedisAdapter;

$cache = new ChainAdapter([
    new ApcuAdapter(),          // L1 (fastest)
    new RedisAdapter($redis),   // L2 (fallback)
]);
```

### ProxyAdapter

```php
use Symfony\Component\Cache\Adapter\ProxyAdapter;

$proxy = new ProxyAdapter($redisCache, 'namespace', 0);
```

### PhpFilesAdapter / PhpArrayAdapter

For opcache-optimized storage of static data.

---

## Redis Configuration

### DSN Format

```
redis[s]://[pass@][ip|host|socket[:port]][/db-index]
redis[s]://[[user]:pass@]?[ip|host|socket[:port]][&params]
```

### DSN Examples

```php
RedisAdapter::createConnection('redis://localhost');
RedisAdapter::createConnection('redis://my.server.com:6379');
RedisAdapter::createConnection('redis://my.server.com:6379/20');
RedisAdapter::createConnection('redis://abcdef@localhost?timeout=5');
RedisAdapter::createConnection('redis://bad-pass@/var/run/redis.sock');
```

### Redis Sentinel (High Availability)

```php
RedisAdapter::createConnection(
    'redis:?host[redis1:26379]&host[redis2:26379]&host[redis3:26379]&redis_sentinel=mymaster'
);

// With credentials and DB index
RedisAdapter::createConnection(
    'redis:default:verysecurepassword@?host[redis1:26379]&host[redis2:26379]&redis_sentinel=mymaster&dbindex=3'
);
```

### Redis Cluster

```php
RedisAdapter::createConnection(
    'redis:?host[localhost]&host[localhost:6379]&host[/var/run/redis.sock:]&auth=my-password&redis_cluster=1'
);
```

### Valkey Support (Symfony 7.3+)

```php
RedisAdapter::createConnection('valkey://localhost');
```

### Connection Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `class` | string | null | Connection library: `\Redis`, `\Relay\Relay`, `\Predis\Client` |
| `persistent` | int | 0 | Enable persistent connections |
| `persistent_id` | string | null | Persistent connection ID |
| `timeout` | int | 30 | Connection timeout (seconds) |
| `read_timeout` | int | 0 | Read timeout (seconds) |
| `retry_interval` | int | 0 | Reconnection delay (ms) |
| `tcp_keepalive` | int | 0 | TCP keepalive timeout (seconds) |
| `lazy` | bool | null | Lazy connections (false standalone, true in Symfony) |
| `redis_cluster` | bool | false | Enable cluster mode |
| `redis_sentinel` | string | null | Sentinel master name |
| `dbindex` | int | 0 | Database index |
| `failover` | string | none | Failover: none, error, distribute, slaves |
| `ssl` | array | null | SSL context options |

### Recommended Redis Server Config

```
maxmemory 100mb
maxmemory-policy allkeys-lru
```

For `RedisTagAwareAdapter`, use `noeviction` or `volatile-*` eviction policy instead.

---

## Cache Tags and Invalidation

### Tag-Aware Adapters

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;

$cache = new TagAwareAdapter(
    new FilesystemAdapter(),        // Items storage
    new RedisAdapter($redis)        // Tags storage (fast invalidation)
);
```

### Optimized Tag-Aware Adapters

```php
use Symfony\Component\Cache\Adapter\RedisTagAwareAdapter;
use Symfony\Component\Cache\Adapter\FilesystemTagAwareAdapter;

$cache = new RedisTagAwareAdapter($redis);
$cache = new FilesystemTagAwareAdapter();
```

### Tagging Items

```php
$value = $cache->get('cache_key', function (ItemInterface $item): string {
    $item->tag('tag_1');
    $item->tag(['tag_2', 'tag_3']);
    $item->expiresAfter(3600);

    return $cachedValue;
});
```

### Invalidating by Tags

```php
// Invalidate all items with tag_1 or tag_3
$cache->invalidateTags(['tag_1', 'tag_3']);

// Or delete a specific key
$cache->delete('cache_key');
```

---

## Stampede Prevention

### Locking (Built-in by default)

Only one PHP process computes a specific key at a time; others wait.

### Probabilistic Early Expiration

Recompute before expiration to avoid simultaneous recomputation:

```php
$beta = 1.0; // Higher = earlier recompute, 0 = disabled, INF = immediate
$value = $cache->get('my_cache_key', function (ItemInterface $item): string {
    $item->expiresAfter(3600);
    return 'value';
}, $beta);
```

---

## Sub-Namespaces (Symfony 7.3+)

```php
// User-specific cache
$userCache = $cache->withSubNamespace(sprintf('user-%d', $user->getId()));

$userCache->get('dashboard_data', function (ItemInterface $item): string {
    $item->expiresAfter(3600);
    return '...';
});

// Locale-specific
$localeCache = $cache->withSubNamespace($request->getLocale());

// Dynamic versioning for invalidation
$userCache = $cache->withSubNamespace(
    sprintf('%s-user-%d', date('Ym'), $user->getId())
);
```

---

## Data Marshalling (Serialization)

### DefaultMarshaller

```php
use Symfony\Component\Cache\Marshaller\DefaultMarshaller;

$marshaller = new DefaultMarshaller();

// With igbinary (requires extension, Symfony 7.2+)
$marshaller = new DefaultMarshaller(useIgbinarySerialize: true);
```

### DeflateMarshaller (Compression)

```php
use Symfony\Component\Cache\Marshaller\DeflateMarshaller;
use Symfony\Component\Cache\Marshaller\DefaultMarshaller;

$marshaller = new DeflateMarshaller(new DefaultMarshaller());
$cache = new RedisAdapter($redis, 'namespace', 0, $marshaller);
```

### SodiumMarshaller (Encryption)

```php
use Symfony\Component\Cache\Marshaller\SodiumMarshaller;

$encryptionKeys = [sodium_crypto_box_keypair()];
$marshaller = new SodiumMarshaller($encryptionKeys);
$cache = new RedisAdapter($redis, 'namespace', 3600, $marshaller);
```

Key rotation with multiple keys:

```php
$keys = [sodium_crypto_box_keypair(), $oldKeypair];
$marshaller = new SodiumMarshaller($keys);
```

---

## PSR-6 Batch Operations

### Multiple Items

```php
$items = $cache->getItems(['AAPL', 'GOOGL', 'MSFT']);
$cache->deleteItems(['category1', 'category2']);
$cache->hasItem('user_badges');
$cache->clear(); // Clear all items
```

### Deferred Saving

```php
$cache->saveDeferred($item1);
$cache->saveDeferred($item2);
$cache->saveDeferred($item3);
$isSaved = $cache->commit();
```

---

## Conditional Saving

```php
$value = $cache->get('key', function (ItemInterface $item, bool &$save) {
    if ($someCondition) {
        $save = false; // Don't cache this result
    }
    return $value;
});
```

---

## Pruning Expired Items

Adapters supporting pruning: ChainAdapter, DoctrineDbalAdapter, FilesystemAdapter, PdoAdapter, PhpFilesAdapter.

```php
$cache->prune();
```

---

## PSR-6 / PSR-16 Interoperability

Convert between PSR-6 (CacheItemPoolInterface) and PSR-16 (SimpleCacheInterface):

```php
use Symfony\Component\Cache\Psr16Cache;

$psr16Cache = new Psr16Cache($psr6Pool);
```

---

## Console Commands

```bash
php bin/console cache:pool:clear cache.app
php bin/console cache:pool:delete cache.app cache_key
php bin/console cache:pool:clear cache.validation cache.app
php bin/console cache:pool:prune
```

---

## Key Interfaces

| Interface | Purpose |
|-----------|---------|
| `Symfony\Contracts\Cache\CacheInterface` | Cache Contracts (get/delete) |
| `Psr\Cache\CacheItemPoolInterface` | PSR-6 pool |
| `Psr\SimpleCache\CacheInterface` | PSR-16 simple cache |
| `Symfony\Contracts\Cache\TagAwareCacheInterface` | Tag-based invalidation |
| `Symfony\Component\Cache\PruneableInterface` | Pruning expired entries |
| `Symfony\Contracts\Cache\ItemInterface` | Cache item in contracts |
| `Psr\Cache\CacheItemInterface` | PSR-6 cache item |
