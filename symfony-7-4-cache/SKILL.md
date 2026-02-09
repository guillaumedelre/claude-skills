---
name: "symfony-7-4-cache"
description: "Symfony 7.4 Cache component reference for caching data with PSR-6 and Cache Contracts. Use when implementing, configuring, or debugging cache pools, adapters, tags, or invalidation strategies. Triggers on: Cache, PSR-6, PSR-16, cache pools, adapters, Redis, Memcached, tags, cache invalidation, CacheInterface, TagAwareAdapter, RedisAdapter, FilesystemAdapter, ChainAdapter, cache stampede, cache items, cache expiration."
---

# Symfony 7.4 Cache Component

GitHub: https://github.com/symfony/cache
Docs: https://symfony.com/doc/7.4/components/cache.html

## Quick Reference

### Cache Contracts (Recommended)

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Contracts\Cache\ItemInterface;

$cache = new FilesystemAdapter();

$value = $cache->get('my_cache_key', function (ItemInterface $item): string {
    $item->expiresAfter(3600);

    return 'computed_value';
});

$cache->delete('my_cache_key');
```

### PSR-6 Usage

```php
$item = $cache->getItem('stats.products_count');

if (!$item->isHit()) {
    $item->set(4711);
    $item->expiresAfter(3600);
    $cache->save($item);
}

$value = $item->get();
```

### Common Adapters

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\Cache\Adapter\ArrayAdapter;
use Symfony\Component\Cache\Adapter\ChainAdapter;

// Redis
$redis = RedisAdapter::createConnection('redis://localhost:6379');
$cache = new RedisAdapter($redis, 'namespace', 3600);

// Chain (multi-tier)
$cache = new ChainAdapter([
    new ArrayAdapter(),
    new RedisAdapter($redis),
]);
```

### Tag-Based Invalidation

```php
use Symfony\Component\Cache\Adapter\TagAwareAdapter;
use Symfony\Component\Cache\Adapter\RedisTagAwareAdapter;

// Tag-aware wrapper
$cache = new TagAwareAdapter(new FilesystemAdapter(), new RedisAdapter($redis));

// Inside callback
$value = $cache->get('key', function (ItemInterface $item): string {
    $item->tag(['tag_1', 'tag_2']);
    $item->expiresAfter(3600);
    return 'value';
});

// Invalidate by tags
$cache->invalidateTags(['tag_1']);
```

### Redis DSN Examples

```php
RedisAdapter::createConnection('redis://localhost');
RedisAdapter::createConnection('redis://secret@my.server.com:6379/2');
RedisAdapter::createConnection('redis:?host[redis1:26379]&host[redis2:26379]&redis_sentinel=mymaster');
```

### Console Commands

```bash
php bin/console cache:pool:clear cache.app
php bin/console cache:pool:delete cache.app cache_key
php bin/console cache:pool:prune
```

## Full Documentation

For complete details including all adapters (APCu, Memcached, PDO, Doctrine DBAL, PhpFiles, Proxy, Couchbase), marshalling/serialization, encryption with SodiumMarshaller, stampede prevention (locking and probabilistic early expiration), sub-namespaces, pruning, deferred saving, and Redis Sentinel/cluster configuration, see [references/cache.md](references/cache.md).
