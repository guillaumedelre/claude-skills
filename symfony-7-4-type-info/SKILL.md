---
name: "symfony-7-4-type-info"
description: "Symfony 7.4 TypeInfo component reference for extracting and manipulating PHP type information. Use when working with type introspection, type resolution from reflection, union types, intersection types, generic types, nullable types, collection types, or any type system-related Symfony code. Triggers on: TypeInfo, type system, Type, TypeResolver, TypeContext, PHP type introspection, union types, intersection types, generic types, TypeIdentifier, type factories, type checking, isIdentifiedBy, isSatisfiedBy, accepts, TypeFactoryTrait, PHPDoc parsing."
---

# Symfony 7.4 TypeInfo Component

## Overview

The TypeInfo component extracts type information from PHP elements like properties, arguments, and return types. It provides a powerful `Type` definition that handles unions, intersections, and generics, enabling sophisticated type introspection and validation.

## Installation

```bash
composer require symfony/type-info

# For PHPDoc parsing support
composer require phpstan/phpdoc-parser
```

## Quick Reference

### Creating Types

```php
use Symfony\Component\TypeInfo\Type;

// Simple types
Type::int();
Type::string();
Type::float();
Type::bool();
Type::array();
Type::null();
Type::mixed();
Type::void();
Type::never();
Type::true();
Type::false();

// Nullable types
Type::nullable(Type::string());

// Union types
Type::union(Type::string(), Type::int());

// Intersection types
Type::intersection(Type::object(\Stringable::class), Type::object(\Iterator::class));

// Generic types
Type::generic(Type::object(Collection::class), Type::int());

// Collection types
Type::list(Type::bool());
Type::dict(Type::string(), Type::int());

// Object types
Type::object(MyClass::class);

// From values (Symfony 7.3+)
Type::fromValue(1.1);   // Type::float()
Type::fromValue('...');  // Type::string()
Type::fromValue(false); // Type::false()
```

### Resolving Types from Reflection

```php
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

class Dummy
{
    public function __construct(
        public int $id,
        /** @var string[] $tags */
        public array $tags,
    ) {
    }
}

$typeResolver = TypeResolver::create();

// Resolve from reflection
$typeResolver->resolve(new \ReflectionProperty(Dummy::class, 'id'));
// Returns: int Type

// Resolve from string
$typeResolver->resolve('bool');
// Returns: bool Type

// With PHPDoc parsing (requires phpstan/phpdoc-parser)
$typeResolver->resolve(new \ReflectionProperty(Dummy::class, 'tags'));
// Returns: collection with int keys and string values
```

### Type Introspection

```php
use Symfony\Component\TypeInfo\Type;
use Symfony\Component\TypeInfo\TypeIdentifier;

// Check type identity
$type = Type::int();
$type->isIdentifiedBy(TypeIdentifier::INT);    // true
$type->isIdentifiedBy(TypeIdentifier::STRING); // false

// Union types match any member
$type = Type::union(Type::string(), Type::int());
$type->isIdentifiedBy(TypeIdentifier::INT);    // true
$type->isIdentifiedBy(TypeIdentifier::STRING); // true

// Object types with inheritance
class Dummy extends DummyParent implements DummyInterface {}

$type = Type::object(Dummy::class);
$type->isIdentifiedBy(TypeIdentifier::OBJECT);   // true
$type->isIdentifiedBy(Dummy::class);             // true
$type->isIdentifiedBy(DummyParent::class);       // true (inheritance)
$type->isIdentifiedBy(DummyInterface::class);    // true (implementation)

// Check nullability
$type = Type::nullable(Type::string());
$type->isNullable(); // true

// Check if accepts value (Symfony 7.3+)
$type = Type::int();
$type->accepts(123);  // true
$type->accepts('z');  // false

$type = Type::union(Type::string(), Type::int());
$type->accepts(123);  // true
$type->accepts('z');  // true
```

### Collection Type Methods

```php
$type = Type::list(Type::nullable(Type::bool()));

// Get key type
$keyType = $type->getCollectionKeyType(); // int Type

// Get value type
$valueType = $type->getCollectionValueType(); // nullable bool Type

// Chain methods
$isValueNullable = $type->getCollectionValueType()->isNullable(); // true
```

### Custom Type Validation with Callables

```php
use Symfony\Component\TypeInfo\Type;
use Symfony\Component\TypeInfo\TypeIdentifier;
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

class Foo
{
    private int $integer;
    private string $string;
    private ?float $float;
}

$reflClass = new \ReflectionClass(Foo::class);
$resolver = TypeResolver::create();

$integerType = $resolver->resolve($reflClass->getProperty('integer'));
$stringType = $resolver->resolve($reflClass->getProperty('string'));
$floatType = $resolver->resolve($reflClass->getProperty('float'));

// Define callable for custom validation
$isNonNullableNumber = function (Type $type): bool {
    if ($type->isNullable()) {
        return false;
    }

    if ($type->isIdentifiedBy(TypeIdentifier::INT) ||
        $type->isIdentifiedBy(TypeIdentifier::FLOAT)) {
        return true;
    }

    return false;
};

$integerType->isSatisfiedBy($isNonNullableNumber); // true
$stringType->isSatisfiedBy($isNonNullableNumber);  // false
$floatType->isSatisfiedBy($isNonNullableNumber);   // false (nullable)
```

## TypeIdentifier Constants

```php
use Symfony\Component\TypeInfo\TypeIdentifier;

TypeIdentifier::INT
TypeIdentifier::STRING
TypeIdentifier::FLOAT
TypeIdentifier::BOOL
TypeIdentifier::ARRAY
TypeIdentifier::OBJECT
TypeIdentifier::NULL
TypeIdentifier::MIXED
TypeIdentifier::VOID
TypeIdentifier::NEVER
TypeIdentifier::TRUE
TypeIdentifier::FALSE
TypeIdentifier::CALLABLE
TypeIdentifier::ITERABLE
TypeIdentifier::RESOURCE
```

## Key Classes

| Class | Description |
|-------|-------------|
| `Type` | Main class representing PHP types with factory methods |
| `TypeResolver` | Resolves types from reflection or strings |
| `TypeIdentifier` | Enumeration of type identifiers |
| `TypeFactoryTrait` | Provides factory methods for type creation |

## Full Documentation
- GitHub: [symfony/type-info](https://github.com/symfony/type-info)
- Docs: [Symfony TypeInfo Component](https://symfony.com/doc/7.4/components/type_info.html)
- **Full Reference**: See [references/type-info.md](references/type-info.md) for complete documentation
