# Symfony 7.4 Contracts Component - Full Documentation

Source: https://symfony.com/doc/7.4/components/contracts.html
GitHub: https://github.com/symfony/contracts

## Overview

The Contracts component provides a set of abstractions extracted out of the Symfony components. They can be used to build on semantics that the Symfony components proved useful - and that already have battle-tested implementations.

## Installation

Contracts are provided as separate packages, so you can install only the ones your projects really need:

```bash
composer require symfony/cache-contracts
composer require symfony/event-dispatcher-contracts
composer require symfony/deprecation-contracts
composer require symfony/http-client-contracts
composer require symfony/service-contracts
composer require symfony/translation-contracts
```

If you install this component outside of a Symfony application, you must require the `vendor/autoload.php` file in your code to enable the class autoloading mechanism provided by Composer.

## Usage

The abstractions in this package are useful to achieve loose coupling and interoperability. By using the provided interfaces as type hints, you are able to reuse any implementations that match their contracts. It could be a Symfony component, or another package provided by the PHP community at large.

### Autowiring Compatibility

Depending on their semantics, some interfaces can be combined with autowiring to seamlessly inject a service in your classes.

### Labeling Interfaces

Others might be useful as labeling interfaces, to hint about a specific behavior that can be enabled when using autoconfiguration or manual service tagging (or any other means provided by your framework).

## Contract Packages

### Cache Contracts (`symfony/cache-contracts`)

Provides interfaces for cache interoperability.

**Key interfaces:**

- `Symfony\Contracts\Cache\CacheInterface` - Main cache interface with callback-based get method
- `Symfony\Contracts\Cache\TagAwareCacheInterface` - Extends CacheInterface with tag-based invalidation
- `Symfony\Contracts\Cache\CallbackInterface` - Interface for cache callback items
- `Symfony\Contracts\Cache\ItemInterface` - Extends PSR-6 CacheItemInterface

```php
use Symfony\Contracts\Cache\CacheInterface;
use Symfony\Contracts\Cache\ItemInterface;

class ProductService
{
    public function __construct(private CacheInterface $cache) {}

    public function getProduct(int $id): array
    {
        return $this->cache->get('product_'.$id, function (ItemInterface $item) use ($id) {
            $item->expiresAfter(3600);
            return $this->fetchProduct($id);
        });
    }
}
```

Tag-aware caching:

```php
use Symfony\Contracts\Cache\TagAwareCacheInterface;
use Symfony\Contracts\Cache\ItemInterface;

class CatalogService
{
    public function __construct(private TagAwareCacheInterface $cache) {}

    public function getCategory(int $id): array
    {
        return $this->cache->get('category_'.$id, function (ItemInterface $item) use ($id) {
            $item->tag(['catalog', 'category_'.$id]);
            return $this->fetchCategory($id);
        });
    }

    public function invalidateCategory(int $id): void
    {
        $this->cache->invalidateTags(['category_'.$id]);
    }
}
```

### Event Dispatcher Contracts (`symfony/event-dispatcher-contracts`)

Provides the event dispatching abstraction.

**Key interfaces:**

- `Symfony\Contracts\EventDispatcher\EventDispatcherInterface` - Dispatches events to listeners

```php
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;
use Symfony\Contracts\EventDispatcher\Event;

class OrderService
{
    public function __construct(private EventDispatcherInterface $dispatcher) {}

    public function placeOrder(Order $order): void
    {
        // ... process order
        $this->dispatcher->dispatch(new OrderPlacedEvent($order));
    }
}
```

### HTTP Client Contracts (`symfony/http-client-contracts`)

Provides HTTP client abstractions.

**Key interfaces:**

- `Symfony\Contracts\HttpClient\HttpClientInterface` - HTTP client for making requests
- `Symfony\Contracts\HttpClient\ResponseInterface` - Represents an HTTP response
- `Symfony\Contracts\HttpClient\ResponseStreamInterface` - Streaming response interface
- `Symfony\Contracts\HttpClient\ChunkInterface` - Represents a chunk of a streamed response

```php
use Symfony\Contracts\HttpClient\HttpClientInterface;

class ApiClient
{
    public function __construct(private HttpClientInterface $httpClient) {}

    public function fetchData(string $url): array
    {
        $response = $this->httpClient->request('GET', $url);
        return $response->toArray();
    }
}
```

### Service Contracts (`symfony/service-contracts`)

Provides service container abstractions.

**Key interfaces:**

- `Symfony\Contracts\Service\ServiceSubscriberInterface` - Declares service dependencies for a service locator
- `Symfony\Contracts\Service\ServiceProviderInterface` - Extends ContainerInterface with getProvidedServices()
- `Symfony\Contracts\Service\ResetInterface` - Allows resetting a service to its initial state
- `Symfony\Contracts\Service\ServiceCollectionInterface` - Extends ServiceProviderInterface and is countable/iterable
- `Symfony\Contracts\Service\ServiceMethodsSubscriberTrait` - Trait for automatic service subscription via return types

```php
use Symfony\Contracts\Service\ServiceSubscriberInterface;
use Psr\Container\ContainerInterface;

class MyService implements ServiceSubscriberInterface
{
    public function __construct(private ContainerInterface $locator) {}

    public static function getSubscribedServices(): array
    {
        return [
            'logger' => '?'.LoggerInterface::class,
            'mailer' => MailerInterface::class,
        ];
    }

    public function doSomething(): void
    {
        if ($this->locator->has('logger')) {
            $this->locator->get('logger')->info('Doing something');
        }
    }
}
```

Using ResetInterface:

```php
use Symfony\Contracts\Service\ResetInterface;

class ConnectionPool implements ResetInterface
{
    private array $connections = [];

    public function reset(): void
    {
        foreach ($this->connections as $connection) {
            $connection->close();
        }
        $this->connections = [];
    }
}
```

### Translation Contracts (`symfony/translation-contracts`)

Provides translation abstractions.

**Key interfaces:**

- `Symfony\Contracts\Translation\TranslatorInterface` - Translates messages
- `Symfony\Contracts\Translation\LocaleAwareInterface` - Interface for locale-aware services
- `Symfony\Contracts\Translation\TranslatableInterface` - Interface for translatable objects

```php
use Symfony\Contracts\Translation\TranslatorInterface;

class NotificationService
{
    public function __construct(private TranslatorInterface $translator) {}

    public function getGreeting(string $name): string
    {
        return $this->translator->trans('greeting', ['%name%' => $name], 'messages');
    }
}
```

### Deprecation Contracts (`symfony/deprecation-contracts`)

Provides a unified way to trigger deprecation notices.

```php
use function Symfony\Component\Deprecation\trigger_deprecation;

trigger_deprecation('my/package', '2.3', 'The "%s" method is deprecated, use "%s" instead.', 'oldMethod', 'newMethod');
```

## Design Principles

1. **Domain-based Organization**: Contracts are split by domain, each into their own sub-namespaces.

2. **Consistency**: Contracts are small and consistent sets of PHP interfaces, traits, normative docblocks, and reference test suites (when applicable).

3. **Proven Implementation**: Contracts must have a proven implementation to enter the repository.

4. **Backward Compatibility**: Contracts must be backward compatible with existing Symfony components.

## Composer Provider Convention

Packages that implement specific contracts should list them in the `provide` section of their `composer.json` file, using the `symfony/*-implementation` convention:

```json
{
    "...": "...",
    "provide": {
        "symfony/cache-implementation": "3.0"
    }
}
```

## How Is This Different From PHP-FIG's PSRs?

When applicable, the provided contracts are built on top of PHP-FIG's PSRs. However, PHP-FIG has different goals and different processes. Symfony Contracts focuses on providing abstractions that are useful on their own while still compatible with implementations provided by Symfony. This allows more granular and specialized abstractions tailored to the Symfony ecosystem while maintaining compatibility with PHP standards where applicable.
