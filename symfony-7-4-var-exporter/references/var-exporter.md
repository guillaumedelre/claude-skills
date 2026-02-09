# Symfony VarExporter Component - Complete Reference

## Overview

The VarExporter component exports any serializable PHP data structure to plain PHP code and allows instantiation and population of objects without calling their constructors.

**Repository**: https://github.com/symfony/var-exporter
**Documentation**: https://symfony.com/doc/7.4/components/var_exporter.html
**License**: MIT
**Latest Version**: v7.4.0 (November 2025)

## Installation

```bash
composer require symfony/var-exporter
```

---

## 1. Exporting/Serializing Variables

### VarExporter::export()

Export PHP data structures to plain PHP code, similar to `var_export()` but with better performance:

```php
use Symfony\Component\VarExporter\VarExporter;

$exported = VarExporter::export($someVariable);
// Store in file or cache
file_put_contents('exported.php', '<?php return '.$exported.';');

// Later, regenerate the variable
$regeneratedVariable = require 'exported.php';
```

### Advantages over serialize()/igbinary

- **Performance**: OPcache optimization makes it significantly faster and more memory efficient
- **Semantics**: Preserves `__wakeup()`, `__sleep()`, and `Serializable` interfaces
- **Reference Handling**: Preserves references in `SplObjectStorage`, `ArrayObject`, `ArrayIterator`
- **Class Validation**: Throws `ClassNotFoundException` for missing classes instead of `PHP_Incomplete_Class`
- **Restrictions**: Throws exception for `Reflection*`, `IteratorIterator`, `RecursiveIteratorIterator`

### Generated Output Example

For a class hierarchy:

```php
abstract class AbstractClass {
    protected int $foo;
    private int $bar;

    protected function setBar($bar): void {
        $this->bar = $bar;
    }
}

class ConcreteClass extends AbstractClass {
    public function __construct() {
        $this->foo = 123;
        $this->setBar(234);
    }
}
```

Generated PHP:

```php
return \Symfony\Component\VarExporter\Internal\Hydrator::hydrate(
    $o = [
        clone (\Symfony\Component\VarExporter\Internal\Registry::$prototypes['Symfony\\Component\\VarExporter\\Tests\\ConcreteClass'] ?? \Symfony\Component\VarExporter\Internal\Registry::p('Symfony\\Component\\VarExporter\\Tests\\ConcreteClass')),
    ],
    null,
    [
        'Symfony\\Component\\VarExporter\\Tests\\AbstractClass' => [
            'foo' => [123],
            'bar' => [234],
        ],
    ],
    $o[0],
    []
);
```

---

## 2. Instantiating PHP Classes

### Instantiator::instantiate()

Create objects and set properties without calling constructors:

```php
use Symfony\Component\VarExporter\Instantiator;

// Create empty instance
$fooObject = Instantiator::instantiate(Foo::class);

// Create with properties
$fooObject = Instantiator::instantiate(Foo::class, ['propertyName' => $propertyValue]);

// Populate parent class private properties
$fooObject = Instantiator::instantiate(Foo::class, [], [
    Bar::class => ['privateBarProperty' => $propertyValue],
]);
```

### Special Cases for Collections

```php
// SplObjectStorage (use "\0" for internal value)
$storage = Instantiator::instantiate(SplObjectStorage::class, [
    "\0" => [$object1, $info1, $object2, $info2],
]);

// ArrayObject
$arrayObject = Instantiator::instantiate(ArrayObject::class, [
    "\0" => [$inputArray],
]);

// ArrayIterator
$arrayIterator = Instantiator::instantiate(ArrayIterator::class, [
    "\0" => [$inputArray],
]);
```

---

## 3. Hydrating PHP Classes

### Hydrator::hydrate()

Populate properties of existing objects:

```php
use Symfony\Component\VarExporter\Hydrator;

$object = new Foo();
Hydrator::hydrate($object, ['propertyName' => $propertyValue]);

// Populate parent class private properties (recommended syntax)
$object = new Foo();
Hydrator::hydrate($object, [], [
    Bar::class => ['privateBarProperty' => $propertyValue],
]);

// Alternative syntax using null-byte notation
Hydrator::hydrate($object, ["\0Bar\0privateBarProperty" => $propertyValue]);
```

### Special Cases for Collections

```php
// SplObjectStorage
$storage = new SplObjectStorage();
Hydrator::hydrate($storage, [
    "\0" => [$object1, $info1, $object2, $info2],
]);

// ArrayObject
$arrayObject = new ArrayObject();
Hydrator::hydrate($arrayObject, [
    "\0" => [$inputArray],
]);

// ArrayIterator
$arrayIterator = new ArrayIterator();
Hydrator::hydrate($arrayIterator, [
    "\0" => [$inputArray],
]);
```

---

## 4. Creating Lazy Objects

Lazy objects are instantiated empty and populated on demand, useful for heavy computations.

### Modern Approach (PHP 8.4+)

Use `ProxyHelper` to generate lazy proxy code:

```php
use Symfony\Component\VarExporter\ProxyHelper;

$proxyCode = ProxyHelper::generateLazyProxy(new \ReflectionClass(SomeInterface::class));
// In production, dump $proxyCode to a file instead of using eval()
eval('class ProxyDecorator'.$proxyCode);

$proxy = ProxyDecorator::createLazyProxy(initializer: function (): SomeInterface {
    // Heavy logic here
    $instance = new SomeHeavyClass(...$dependencies);
    // Call setters, etc.
    return $instance;
});
```

### Legacy: LazyGhostTrait (Deprecated in Symfony 7.3)

**Note**: Use native lazy objects instead for PHP 8.4+.

Ghost objects are empty and populate on first method call:

```php
use Symfony\Component\VarExporter\LazyGhostTrait;

class HashProcessor {
    use LazyGhostTrait;

    public readonly string $hash;

    public function __construct() {
        self::createLazyGhost(initializer: $this->populateHash(...), instance: $this);
    }

    private function populateHash(array $data): void {
        // Compute $this->hash
    }
}
```

#### Converting Non-Lazy Classes to Lazy

```php
// Original class
class HashProcessor {
    public readonly string $hash;

    public function __construct(array $data) {
        $this->populateHash($data);
    }

    private function populateHash(array $data): void {
        // Heavy computation
    }

    public function validateHash(): bool {
        // Validation logic
    }
}

// Create lazy version
class LazyHashProcessor extends HashProcessor {
    use LazyGhostTrait;
}

// Usage
$processor = LazyHashProcessor::createLazyGhost(initializer: function (HashProcessor $instance): void {
    $data = /* retrieve data */;
    $instance->__construct(...$data);
    $instance->validateHash();
});
```

### Legacy: LazyProxyTrait (Deprecated in Symfony 7.3)

**Note**: Use native lazy objects instead for PHP 8.4+.

Virtual proxies for abstract/internal classes using the Liskov Substitution Principle:

```php
use Symfony\Component\VarExporter\ProxyHelper;

interface ProcessorInterface {
    public function getHash(): bool;
}

abstract class AbstractProcessor implements ProcessorInterface {
    protected string $hash;

    public function getHash(): bool {
        return $this->hash;
    }
}

class HashProcessor extends AbstractProcessor {
    public function __construct(array $data) {
        $this->populateHash($data);
    }

    private function populateHash(array $data): void {
        // Heavy computation
    }
}

// Generate proxy
$proxyCode = ProxyHelper::generateLazyProxy(new \ReflectionClass(AbstractProcessor::class));
eval('class HashProcessorProxy'.$proxyCode);

// Create lazy proxy
$processor = HashProcessorProxy::createLazyProxy(initializer: function (): ProcessorInterface {
    $data = /* retrieve data */;
    $instance = new HashProcessor(...$data);
    return $instance;
});
```

---

## 5. Comparison Table

| Feature | VarExporter | serialize() | var_export() |
|---------|-------------|-------------|--------------|
| Performance | High (OPcache) | Low | Medium |
| Semantics Preserved | Yes | Yes | No |
| Reference Handling | Yes | Yes | Limited |
| PSR-2 Format | Yes | No | Partial |
| Class Validation | Exception | PHP_Incomplete_Class | N/A |

---

## 6. Common Use Cases

### Caching Configuration Objects

```php
use Symfony\Component\VarExporter\VarExporter;

// Export configuration for fast retrieval via OPcache
$config = loadAndParseConfiguration();
$exported = VarExporter::export($config);
file_put_contents('cache/config.php', '<?php return '.$exported.';');

// Fast retrieval
$config = require 'cache/config.php';
```

### Dependency Injection Container

```php
use Symfony\Component\VarExporter\Instantiator;

// Instantiate service without executing constructor
$service = Instantiator::instantiate(HeavyService::class, [
    'dependency1' => $dep1,
    'dependency2' => $dep2,
]);
```

### Testing Fixtures

```php
use Symfony\Component\VarExporter\Instantiator;

// Create test fixtures without running constructors
$entity = Instantiator::instantiate(User::class, [
    'id' => 1,
    'email' => 'test@example.com',
    'createdAt' => new \DateTimeImmutable('2024-01-01'),
]);
```

### Lazy Loading Heavy Objects

```php
use Symfony\Component\VarExporter\ProxyHelper;

// Create lazy proxy for expensive service
$proxyCode = ProxyHelper::generateLazyProxy(new \ReflectionClass(ExpensiveService::class));
eval('class LazyExpensiveService'.$proxyCode);

$service = LazyExpensiveService::createLazyProxy(initializer: function () use ($container) {
    return $container->get(ExpensiveService::class);
});
```

---

## 7. Error Handling

### ClassNotFoundException

Thrown when exporting an object whose class does not exist:

```php
use Symfony\Component\VarExporter\Exception\ClassNotFoundException;

try {
    $exported = VarExporter::export($object);
} catch (ClassNotFoundException $e) {
    // Handle missing class
}
```

### ExceptionInterface

Base interface for all VarExporter exceptions:

```php
use Symfony\Component\VarExporter\Exception\ExceptionInterface;

try {
    $exported = VarExporter::export($object);
} catch (ExceptionInterface $e) {
    // Handle any VarExporter exception
}
```

---

## 8. Internal Classes

The following classes are used internally and should not be used directly:

- `Symfony\Component\VarExporter\Internal\Hydrator` - Used by generated export code
- `Symfony\Component\VarExporter\Internal\Registry` - Manages class prototypes

---

## Resources

- **Official Documentation**: https://symfony.com/doc/7.4/components/var_exporter.html
- **GitHub Repository**: https://github.com/symfony/var-exporter
- **Symfony Components**: https://symfony.com/components
