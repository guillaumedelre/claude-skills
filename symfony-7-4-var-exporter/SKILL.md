---
name: "symfony-7-4-var-exporter"
description: "Symfony VarExporter component for exporting PHP variables, instantiating objects without constructors, and creating lazy objects. Use when working with VarExporter, variable export, Instantiator, Hydrator, lazy objects, lazy ghost, LazyProxyTrait, LazyGhostTrait, or object serialization."
---

# Symfony VarExporter Component (7.4)

## Overview

The VarExporter component exports any serializable PHP data structure to plain PHP code and allows instantiation and population of objects without calling their constructors. It provides significant performance benefits through OPcache optimization.

## Quick Reference

### Installation

```bash
composer require symfony/var-exporter
```

### VarExporter::export()

Export PHP data structures to plain PHP code:

```php
use Symfony\Component\VarExporter\VarExporter;

$exported = VarExporter::export($someVariable);
file_put_contents('exported.php', '<?php return '.$exported.';');

// Regenerate the variable later
$regeneratedVariable = require 'exported.php';
```

### Instantiator

Create objects without calling constructors:

```php
use Symfony\Component\VarExporter\Instantiator;

// Create empty instance
$object = Instantiator::instantiate(Foo::class);

// Create with properties
$object = Instantiator::instantiate(Foo::class, ['propertyName' => $value]);

// Set parent class private properties
$object = Instantiator::instantiate(Foo::class, [], [
    ParentClass::class => ['privateProperty' => $value],
]);
```

### Hydrator

Populate properties of existing objects:

```php
use Symfony\Component\VarExporter\Hydrator;

$object = new Foo();
Hydrator::hydrate($object, ['propertyName' => $value]);

// Populate parent class private properties
Hydrator::hydrate($object, [], [
    ParentClass::class => ['privateProperty' => $value],
]);
```

### Lazy Objects (PHP 8.4+)

Create lazy proxies for deferred initialization:

```php
use Symfony\Component\VarExporter\ProxyHelper;

$proxyCode = ProxyHelper::generateLazyProxy(new \ReflectionClass(SomeInterface::class));
eval('class ProxyDecorator'.$proxyCode);

$proxy = ProxyDecorator::createLazyProxy(initializer: function (): SomeInterface {
    return new SomeHeavyClass(...$dependencies);
});
```

### Legacy: LazyGhostTrait (Deprecated in 7.3)

```php
use Symfony\Component\VarExporter\LazyGhostTrait;

class LazyProcessor extends Processor {
    use LazyGhostTrait;
}

$processor = LazyProcessor::createLazyGhost(initializer: function (Processor $instance): void {
    $instance->__construct($data);
});
```

### Legacy: LazyProxyTrait (Deprecated in 7.3)

```php
use Symfony\Component\VarExporter\ProxyHelper;

$proxyCode = ProxyHelper::generateLazyProxy(new \ReflectionClass(AbstractProcessor::class));
eval('class ProcessorProxy'.$proxyCode);

$processor = ProcessorProxy::createLazyProxy(initializer: function (): ProcessorInterface {
    return new ConcreteProcessor($data);
});
```

## Key Advantages

| Feature | VarExporter | serialize() | var_export() |
|---------|-------------|-------------|--------------|
| OPcache Performance | Yes | No | No |
| Preserves __wakeup/__sleep | Yes | Yes | No |
| Reference Handling | Yes | Yes | Limited |
| PSR-2 Format | Yes | No | Partial |

## Full Documentation
- Docs: https://symfony.com/doc/7.4/components/var_exporter.html
- GitHub: https://github.com/symfony/var-exporter

For complete API documentation and advanced usage, see [references/var-exporter.md](references/var-exporter.md).
