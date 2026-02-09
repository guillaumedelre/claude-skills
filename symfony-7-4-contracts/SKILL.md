---
name: "symfony-7-4-contracts"
description: "Symfony 7.4 Contracts component reference for PHP interfaces, traits, and abstractions enabling interoperability. Use when working with Symfony Contracts, service interfaces, autowiring abstractions, or implementing interoperability patterns. Triggers on: Contracts, interfaces, abstractions, interoperability, ServiceSubscriberInterface, ResetInterface, CacheInterface, TagAwareCacheInterface, EventDispatcherInterface, TranslatorInterface, HttpClientInterface."
---

# Symfony 7.4 Contracts Component

GitHub: https://github.com/symfony/contracts
Docs: https://symfony.com/doc/7.4/components/contracts.html

## Quick Reference

### What Are Contracts?

Contracts are a set of PHP interfaces, traits, and normative docblocks extracted from Symfony components. They provide abstractions for loose coupling and interoperability.

### Installation

```bash
composer require symfony/cache-contracts
composer require symfony/event-dispatcher-contracts
composer require symfony/deprecation-contracts
composer require symfony/http-client-contracts
composer require symfony/service-contracts
composer require symfony/translation-contracts
```

### Using Contracts as Type Hints

```php
use Symfony\Contracts\Cache\CacheInterface;

class MyService
{
    public function __construct(
        private CacheInterface $cache,
    ) {}

    public function getData(): array
    {
        return $this->cache->get('my_key', function () {
            return $this->computeExpensiveData();
        });
    }
}
```

### Service Contracts (Autowiring)

```php
use Symfony\Contracts\Service\ServiceSubscriberInterface;
use Psr\Container\ContainerInterface;

class MyController implements ServiceSubscriberInterface
{
    public function __construct(private ContainerInterface $locator) {}

    public static function getSubscribedServices(): array
    {
        return [
            'logger' => LoggerInterface::class,
        ];
    }
}
```

### Declaring Implementation in composer.json

```json
{
    "provide": {
        "symfony/cache-implementation": "3.0"
    }
}
```

### Key Contract Packages

| Package | Interfaces |
|---------|-----------|
| `cache-contracts` | `CacheInterface`, `TagAwareCacheInterface`, `CallbackInterface` |
| `event-dispatcher-contracts` | `EventDispatcherInterface` |
| `http-client-contracts` | `HttpClientInterface`, `ResponseInterface` |
| `service-contracts` | `ServiceSubscriberInterface`, `ResetInterface`, `ServiceProviderInterface` |
| `translation-contracts` | `TranslatorInterface`, `LocaleAwareInterface` |
| `deprecation-contracts` | `trigger_deprecation()` function |

## Full Documentation

For complete details including design principles, differences from PHP-FIG PSRs, and all available interfaces, see [references/contracts.md](references/contracts.md).
