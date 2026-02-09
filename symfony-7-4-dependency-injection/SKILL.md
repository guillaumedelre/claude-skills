---
name: "symfony-7-4-dependency-injection"
description: "Symfony 7.4 DependencyInjection component reference for building and configuring service containers. Use when working with DI container, service container, autowiring, service definitions, compiler passes, service tags, service decoration, lazy services, setter injection, or any dependency injection-related Symfony code. Triggers on: DependencyInjection, ContainerBuilder, ContainerInterface, autowiring, service definitions, compiler passes, tags, service container, Reference, Definition, TaggedIterator, AsTaggedItem, AutoconfigureTag."
---

# Symfony 7.4 DependencyInjection Component

GitHub: https://github.com/symfony/dependency-injection
Docs: https://symfony.com/doc/7.4/components/dependency_injection.html

## Quick Reference

### Basic Service Registration

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();
$container->setParameter('mailer.transport', 'sendmail');
$container
    ->register('mailer', Mailer::class)
    ->addArgument('%mailer.transport%');
```

### Injecting Service References

```php
use Symfony\Component\DependencyInjection\Reference;

$container
    ->register('newsletter_manager', NewsletterManager::class)
    ->addArgument(new Reference('mailer'));
```

### Setter Injection

```php
$container
    ->register('newsletter_manager', NewsletterManager::class)
    ->addMethodCall('setMailer', [new Reference('mailer')]);
```

### PHP Configuration (services.php)

```php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return static function (ContainerConfigurator $container): void {
    $services = $container->services();

    $services->defaults()
        ->autowire()
        ->autoconfigure();

    $services->load('App\\', '../src/')
        ->exclude('../src/{DependencyInjection,Entity,Kernel.php}');

    $services->set('mailer', Mailer::class)
        ->args([param('mailer.transport')]);
};
```

### YAML Configuration (services.yaml)

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'
        exclude: '../src/{DependencyInjection,Entity,Kernel.php}'

    mailer:
        class: Mailer
        arguments: ['%mailer.transport%']

    newsletter_manager:
        class: NewsletterManager
        calls:
            - [setMailer, ['@mailer']]
```

### Loading Configuration Files

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

$container = new ContainerBuilder();
$loader = new YamlFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.yaml');
```

### Handling Missing Services

```php
use Symfony\Component\DependencyInjection\ContainerInterface;

// Available behaviors for missing references:
ContainerInterface::EXCEPTION_ON_INVALID_REFERENCE;         // throws at compile (default)
ContainerInterface::RUNTIME_EXCEPTION_ON_INVALID_REFERENCE;  // throws at runtime
ContainerInterface::NULL_ON_INVALID_REFERENCE;               // returns null
ContainerInterface::IGNORE_ON_INVALID_REFERENCE;             // ignores wrapping call
ContainerInterface::IGNORE_ON_UNINITIALIZED_REFERENCE;       // ignores uninitialized
```

## Full Documentation

For complete details including compiler passes, service tags, service decoration, lazy services, autowiring configuration, factory services, parent services, immutable setters, and all configuration formats, see [references/dependency-injection.md](references/dependency-injection.md).
