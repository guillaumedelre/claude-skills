---
name: "zenstruck-foundry-2"
description: "Zenstruck Foundry 2.x reference for object factories in Symfony/Doctrine testing. Use when creating test data with factories, anonymous factories, sequences, stories, or fixtures. Triggers on: Zenstruck Foundry, Foundry 2, PersistentProxyObjectFactory, PersistentObjectFactory, ObjectFactory, Factory::createOne, createMany, createSequence, createOneAfter, with, withoutPersisting, _real, _refresh, _save, _delete, _disableAutoRefresh, ResetDatabase, Factories, Story, AsFixture, AsScenario, faker, sequence stories, Proxy, instantiateWith, afterInstantiate, afterPersist, beforeInstantiate, recipes, anonymous factory, make:factory, make:story, FoundryBundle, Configuration, MIN, MAX."
---

# Zenstruck Foundry 2.x

GitHub: https://github.com/zenstruck/foundry
Bundle docs: https://symfony.com/bundles/ZenstruckFoundryBundle/current/index.html
Docs (2.x): https://github.com/zenstruck/foundry/tree/2.x/docs

## Installation

```bash
composer require --dev zenstruck/foundry
```

The bundle is auto-registered in `dev` and `test` environments by Symfony Flex.

## Foundry 2.x: Key Changes from 1.x

| 1.x | 2.x |
|-----|-----|
| `extends ModelFactory` | `extends PersistentProxyObjectFactory` (proxy) or `PersistentObjectFactory` (no proxy) |
| `getDefaults()` | `defaults()` |
| `initialize()` | `initialize()` (same) |
| `getClass()` | `class()` (static) |
| `Factory::new()` deprecated for entities | use `Factory::class()` static |
| `proxy()` (always) | proxy via `PersistentProxyObjectFactory`; raw via `PersistentObjectFactory` |

## Quick Reference

### PersistentProxyObjectFactory (proxy, auto-refresh)

```php
namespace App\Factory;

use App\Entity\Article;
use Zenstruck\Foundry\Persistence\PersistentProxyObjectFactory;

/**
 * @extends PersistentProxyObjectFactory<Article>
 */
final class ArticleFactory extends PersistentProxyObjectFactory
{
    public static function class(): string
    {
        return Article::class;
    }

    protected function defaults(): array|callable
    {
        return [
            'title' => self::faker()->sentence(),
            'slug' => self::faker()->slug(),
            'body' => self::faker()->paragraphs(3, asText: true),
            'publishedAt' => self::faker()->dateTimeBetween('-1 year'),
            'author' => UserFactory::new(),                    // Lazy: created on persist
            'tags' => TagFactory::new()->many(3),              // 3 tags created
        ];
    }

    protected function initialize(): static
    {
        return $this
            ->afterInstantiate(function (Article $article): void {
                $article->setReadingTime(str_word_count($article->getBody()) / 200);
            });
    }
}
```

### PersistentObjectFactory (no proxy, raw object)

```php
use Zenstruck\Foundry\Persistence\PersistentObjectFactory;

/**
 * @extends PersistentObjectFactory<Tag>
 */
final class TagFactory extends PersistentObjectFactory
{
    public static function class(): string
    {
        return Tag::class;
    }

    protected function defaults(): array
    {
        return ['name' => self::faker()->word()];
    }
}
```

Use proxy when tests verify changes via reread; use non-proxy for performance / when you only need a fresh handle.

### ObjectFactory (non-Doctrine objects)

```php
use Zenstruck\Foundry\ObjectFactory;

/**
 * @extends ObjectFactory<Money>
 */
final class MoneyFactory extends ObjectFactory
{
    public static function class(): string
    {
        return Money::class;
    }

    protected function defaults(): array
    {
        return [
            'amount' => self::faker()->randomNumber(4),
            'currency' => 'EUR',
        ];
    }
}
```

### Creating Objects

```php
ArticleFactory::createOne();
ArticleFactory::createOne(['title' => 'Specific title']);
ArticleFactory::createMany(10);
ArticleFactory::createMany(10, ['author' => $user]);
ArticleFactory::createMany(10, fn (int $i) => ['title' => "Article #$i"]);

ArticleFactory::createSequence([
    ['title' => 'First', 'priority' => 1],
    ['title' => 'Second', 'priority' => 2],
]);
```

### Without Persisting

```php
$article = ArticleFactory::new()->withoutPersisting()->create();
// Or the static helper:
$article = ArticleFactory::createOne()->_disableAutoRefresh();
```

### Find / Random

```php
ArticleFactory::find(['slug' => 'hello']);
ArticleFactory::findOrCreate(['slug' => 'hello'], ['title' => 'Hello']);
ArticleFactory::random();
ArticleFactory::randomSet(3);
ArticleFactory::randomRange(min: 2, max: 5);
ArticleFactory::all();
ArticleFactory::count();
ArticleFactory::truncate();
```

### State Methods (with...)

```php
final class ArticleFactory extends PersistentProxyObjectFactory
{
    public function published(): static
    {
        return $this->with(['publishedAt' => new \DateTimeImmutable()]);
    }

    public function draft(): static
    {
        return $this->with(['publishedAt' => null]);
    }

    public function withTitle(string $title): static
    {
        return $this->with(['title' => $title]);
    }
}

ArticleFactory::new()->published()->createOne();
ArticleFactory::new()->draft()->withTitle('Hello')->createMany(3);
```

### Lifecycle Hooks

```php
protected function initialize(): static
{
    return $this
        ->beforeInstantiate(function (array $attrs): array {
            $attrs['slug'] ??= str_replace(' ', '-', strtolower($attrs['title']));
            return $attrs;
        })
        ->afterInstantiate(function (Article $article, array $attrs): void {
            // Modify before persist
        })
        ->afterPersist(function (Article $article, array $attrs): void {
            // Post-flush logic
        })
        ->instantiateWith(function (array $attrs, string $class): Article {
            return new Article($attrs['title'], $attrs['slug']);
        });
}
```

### Proxy API (PersistentProxyObjectFactory)

```php
$article = ArticleFactory::createOne();
// $article is a Proxy<Article>

$article->getTitle();              // Auto-refreshed from DB
$article->_real();                 // Get raw entity
$article->_refresh();              // Force re-fetch
$article->_save();                 // Persist + flush
$article->_delete();
$article->_disableAutoRefresh(function (Article $article): void {
    $article->setTitle('No refresh during this block');
});
```

### Test Setup

```php
use PHPUnit\Framework\TestCase;
use Zenstruck\Foundry\Test\Factories;
use Zenstruck\Foundry\Test\ResetDatabase;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class ArticleTest extends KernelTestCase
{
    use Factories;
    use ResetDatabase;

    public function testCreate(): void
    {
        $article = ArticleFactory::createOne(['title' => 'Hello']);
        $this->assertSame('Hello', $article->getTitle());
    }
}
```

`ResetDatabase` strategies (configured in bundle config):

```yaml
# config/packages/zenstruck_foundry.yaml
when@test:
    zenstruck_foundry:
        orm:
            reset:
                connections: ['default']
                entity_managers: ['default']
                mode: schema       # schema | migrate
        global_state:
            - 'App\Story\GlobalStory'
        instantiator:
            allow_extra_attributes: false
            always_force_properties: true
```

### Stories

```php
use Zenstruck\Foundry\Story;

final class CategoriesStory extends Story
{
    public function build(): void
    {
        CategoryFactory::createOne(['name' => 'Tech'])->_save();
        CategoryFactory::createOne(['name' => 'News'])->_save();

        $this->add('featured', CategoryFactory::createOne(['name' => 'Featured']));
    }
}

// Usage
CategoriesStory::load();
$featured = CategoriesStory::get('featured');
```

`Story::load()` is idempotent for the test class.

### #[AsFixture] (Doctrine Fixtures Bundle)

```php
use Doctrine\Bundle\FixturesBundle\Fixture;
use Zenstruck\Foundry\AsFixture;

#[AsFixture(group: 'demo')]
final class DemoFixtures extends Fixture
{
    public function load(ObjectManager $manager): void
    {
        ArticleFactory::createMany(20);
    }
}
```

### Anonymous Factory

```php
use function Zenstruck\Foundry\Persistence\persistent_factory;
use function Zenstruck\Foundry\Persistence\proxy_factory;

$user = persistent_factory(User::class)->create(['email' => 'a@b.c']);
$post = proxy_factory(Post::class)->create(['title' => 'Anon post']);
```

### Make Commands

```bash
php bin/console make:factory                  # Interactive
php bin/console make:factory Article --persisted
php bin/console make:factory Article --no-persistence
php bin/console make:story
```

## Faker

`self::faker()` returns a `\Faker\Generator`:

```php
self::faker()->name();
self::faker()->email();
self::faker()->dateTimeBetween('-1 year', 'now');
self::faker()->randomElement(['a', 'b', 'c']);
self::faker()->numberBetween(1, 100);
self::faker()->sentence();
self::faker()->paragraphs(3, asText: true);
```

Custom locale / seed:

```yaml
zenstruck_foundry:
    faker:
        locale: fr_FR
        seed: 42
        provider: ['App\Faker\Custom']
```

## Full Documentation

For migration from 1.x, advanced lifecycle, anonymous factories, custom services, and DAMA integration: see [references/foundry.md](references/foundry.md).
