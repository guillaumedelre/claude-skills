# Zenstruck Foundry 2.x - Complete Reference

## Table of Contents

- [1.x to 2.x Migration](#1x-to-2x-migration)
- [Factory Hierarchy](#factory-hierarchy)
- [Configuration](#configuration)
- [Lifecycle Hooks](#lifecycle-hooks)
- [Proxy API](#proxy-api)
- [Stories](#stories)
- [Fixtures Integration](#fixtures-integration)
- [Reset Database](#reset-database)
- [Anonymous Factories](#anonymous-factories)
- [Faker](#faker)
- [Test Traits](#test-traits)
- [PHPUnit / Pest Integration](#phpunit--pest-integration)
- [DAMA Doctrine Test Bundle](#dama-doctrine-test-bundle)
- [CI Tips](#ci-tips)

---

## 1.x to 2.x Migration

The core breaking changes:

| 1.x | 2.x |
|-----|-----|
| `extends ModelFactory` | `extends PersistentProxyObjectFactory` (proxy) or `PersistentObjectFactory` (raw) |
| `getDefaults(): array` | `defaults(): array \| callable` |
| `getClass(): string` | `static class(): string` |
| `Factory::new()->withoutPersisting()->create()` | Same; or use `ObjectFactory` for non-persisted |
| `Proxy::object()` | `Proxy::_real()` |
| `Proxy::refresh()` | `Proxy::_refresh()` |
| `Proxy::save()` | `Proxy::_save()` |
| `Proxy::remove()` | `Proxy::_delete()` |
| `Foundry::configuration()` | `Configuration::default()` |

Run `vendor/bin/foundry-upgrade` (provided by the package) for assisted migration.

## Factory Hierarchy

```
Factory (abstract)
├── ObjectFactory<T>             — non-persisted (POPO, value objects)
└── PersistentObjectFactory<T>   — Doctrine entity, raw return
    └── PersistentProxyObjectFactory<T>   — Doctrine entity, returns Proxy<T>
```

Choose:
- **PersistentProxyObjectFactory**: tests where state changes after creation; auto-refresh helps assertions.
- **PersistentObjectFactory**: bulk seeding; raw entity is faster (no proxy overhead) and easier to type-hint.
- **ObjectFactory**: non-Doctrine objects (DTOs, aggregates, immutables).

## Configuration

```yaml
# config/packages/zenstruck_foundry.yaml
when@dev:
    zenstruck_foundry:
        auto_refresh_proxies: true
when@test:
    zenstruck_foundry:
        auto_refresh_proxies: true
        instantiator:
            allow_extra_attributes: false
            always_force_properties: true
        orm:
            reset:
                connections: ['default']
                entity_managers: ['default']
                mode: 'schema'             # 'schema' | 'migrate'
        odm:
            reset:
                document_managers: ['default']
        mongo:
            reset: ~
        faker:
            locale: 'en_US'
            seed: null
            provider: []
        global_state:
            - 'App\Story\AdminUserStory'
```

`always_force_properties: true` writes via reflection even when the property has no setter. Useful for read-only entities.

`global_state` runs the listed Stories on every test (in setUp), giving consistent baseline data.

## Lifecycle Hooks

In `initialize()`:

```php
protected function initialize(): static
{
    return $this
        ->beforeInstantiate(function (array $attrs): array {
            // Modify attribute array before object construction
            $attrs['slug'] ??= u($attrs['title'])->slug();
            return $attrs;
        })
        ->instantiateWith(function (array $attrs, string $class): object {
            // Custom constructor (skip default instantiator)
            return new $class($attrs['title'], $attrs['slug']);
        })
        ->afterInstantiate(function (Article $a, array $attrs): void {
            // Mutate the freshly-built object before persist
        })
        ->afterPersist(function (Article $a, array $attrs): void {
            // After flush; e.g., trigger downstream side effects in tests
        });
}
```

Multiple hooks of the same type run in registration order.

## Proxy API

`Proxy<T>` (when using `PersistentProxyObjectFactory`) wraps an entity and:

- Forwards method calls and property access to the entity.
- Auto-refreshes from DB before each read (configurable).
- Provides `_*` helpers prefixed with underscore to avoid collisions:

| Method | Purpose |
|--------|---------|
| `_real(): T` | Return underlying entity |
| `_refresh(): static` | Re-fetch from DB |
| `_save(): static` | Persist + flush |
| `_delete(): static` | Remove + flush |
| `_repository(): RepositoryDecorator<T>` | Get the repository |
| `_disableAutoRefresh(callable): mixed` | Run block without auto-refresh |
| `_withoutAutoRefresh(callable): mixed` | Same as above |
| `_assertPersisted(): static` | PHPUnit assertion |
| `_assertNotPersisted(): static` | PHPUnit assertion |

```php
$article = ArticleFactory::createOne();

$article->_disableAutoRefresh(function (Article $a): void {
    $a->setTitle('Mutated');
    $a->setBody('More body');
    // No auto-flush; do not call $a->_save() manually if you don't want to persist
});
```

## Stories

```php
use Zenstruck\Foundry\Story;

final class TenantStory extends Story
{
    public function build(): void
    {
        $this->addState('admin', UserFactory::createOne(['roles' => ['ROLE_ADMIN']]));
        $this->addState('member', UserFactory::createOne(['roles' => ['ROLE_USER']]));

        ArticleFactory::createMany(5, [
            'author' => $this->get('admin'),
        ]);
    }
}

// Usage
TenantStory::load();
$admin = TenantStory::get('admin');

// Pool helpers
$story = TenantStory::load();
$story->getRandom('article');
$story->getRandomSet('article', 3);
```

`Story::load()` is idempotent within a test class. To reload across tests, ensure `ResetDatabase` is in use.

## Fixtures Integration

Stories double as fixture sources via `#[AsFixture]`:

```php
#[AsFixture(group: 'app', priority: 100)]
final class TenantStory extends Story
{
    public function build(): void { /* ... */ }
}
```

Then run:

```bash
php bin/console doctrine:fixtures:load --group=app
```

You can still use plain `Fixture` classes too:

```php
final class AppFixtures extends Fixture
{
    public function load(ObjectManager $manager): void
    {
        UserFactory::createMany(20);
        ArticleFactory::createMany(50);
    }
}
```

## Reset Database

`ResetDatabase` trait wipes/rebuilds DB between test classes (and resets sequences). Modes:

- `schema`: drop & recreate via `doctrine:schema:create`. Fast, ignores migrations.
- `migrate`: drop & run all migrations. Slower; verifies migration consistency.

```yaml
zenstruck_foundry:
    orm:
        reset:
            mode: 'migrate'
```

For per-test isolation (instead of per-class), DAMA Doctrine Test Bundle wraps each test in a transaction that rolls back. Combine with Foundry:

```yaml
# config/packages/dama_doctrine_test_bundle.yaml
when@test:
    dama_doctrine_test:
        enable_static_connection: true
        enable_save_points: true
```

```php
class MyTest extends KernelTestCase
{
    use Factories;
    use ResetDatabase;     // Per class; DAMA gives per-test rollback
}
```

## Anonymous Factories

For one-off entities without a dedicated factory class:

```php
use function Zenstruck\Foundry\Persistence\persistent_factory;
use function Zenstruck\Foundry\Persistence\proxy_factory;
use function Zenstruck\Foundry\object_factory;

$user = persistent_factory(User::class)->create(['email' => 'x@y.z']);
$post = proxy_factory(Post::class)->create(['title' => 'X']);
$dto  = object_factory(SomeDto::class)->create(['field' => 'value']);
```

Variants accept defaults via callable:

```php
$factory = persistent_factory(User::class, ['email' => 'default@x.y']);
$factory->many(5)->create();
```

## Faker

```yaml
zenstruck_foundry:
    faker:
        locale: 'fr_FR'
        seed: 42                 # Deterministic
        provider:
            - App\Faker\BookProvider
        service: 'app.faker'     # Use a DI service instead
```

Custom provider:

```php
final class BookProvider extends \Faker\Provider\Base
{
    public function isbn(): string { return '978-' . $this->generator->numerify('##########'); }
}
```

```php
self::faker()->isbn();   // Available after registering
```

## Test Traits

| Trait | Effect |
|-------|--------|
| `Factories` | Boots Foundry; auto-discovers configuration; refreshes between tests |
| `ResetDatabase` | Drops and recreates schema (or runs migrations) before first test of class |

Add both to `KernelTestCase` subclasses:

```php
class MyTest extends KernelTestCase
{
    use Factories, ResetDatabase;
}
```

## PHPUnit / Pest Integration

PHPUnit 12+ works out of the box. For Pest:

```php
// tests/Pest.php
uses(Factories::class)->in('Feature');
uses(ResetDatabase::class)->in('Feature');
```

```php
test('creates an article', function (): void {
    $article = ArticleFactory::createOne();
    expect($article->getTitle())->not->toBeEmpty();
});
```

## DAMA Doctrine Test Bundle

```bash
composer require --dev dama/doctrine-test-bundle
```

Bundle automatically activates in `test`. Combined with Foundry:

- `ResetDatabase` recreates schema once per class.
- DAMA wraps each test method in a transaction, rolling back at end.

This gives both schema-correctness and per-test isolation without truncating tables every test.

## CI Tips

- Use `mode: schema` for fast feedback in CI; periodically run `mode: migrate` to catch broken migrations.
- Set deterministic faker seed via `KERNEL_FAKER_SEED` env var or fixture-time:

  ```php
  Factory::configuration()->faker()->seed(42);
  ```
- Cache vendor + var/cache between CI jobs to skip schema warmup.
- Parallelize PHPUnit with `--processes` (Paratest) but ensure each process uses an isolated DB (Postgres template DBs work well).
