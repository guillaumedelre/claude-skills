# Symfony 7.4 TypeInfo Component - Complete Reference

## Overview

The TypeInfo component extracts PHP type information from properties, arguments, return types, and other PHP elements. It provides a powerful `Type` definition that handles complex type scenarios including unions, intersections, and generics.

## Installation

```bash
composer require symfony/type-info
```

For PHPDoc parsing support (recommended):

```bash
composer require phpstan/phpdoc-parser
```

## Core Concepts

### The Type Class

The `Type` class is the central abstraction representing PHP types. It provides:
- Static factory methods for creating type instances
- Methods for type introspection and comparison
- Support for complex types (unions, intersections, generics)

### Type Resolution

`TypeResolver` extracts type information from:
- Reflection objects (`ReflectionProperty`, `ReflectionParameter`, `ReflectionMethod`)
- String type declarations
- PHPDoc annotations (with phpstan/phpdoc-parser)

## Creating Types

### Simple Types

```php
use Symfony\Component\TypeInfo\Type;

// Scalar types
Type::int();
Type::string();
Type::float();
Type::bool();

// Special types
Type::array();
Type::null();
Type::mixed();
Type::void();
Type::never();
Type::true();
Type::false();
Type::callable();
Type::iterable();
Type::resource();
```

### Nullable Types

```php
use Symfony\Component\TypeInfo\Type;

// Nullable string (?string)
Type::nullable(Type::string());

// Nullable int (?int)
Type::nullable(Type::int());

// Nullable object
Type::nullable(Type::object(MyClass::class));
```

### Union Types

```php
use Symfony\Component\TypeInfo\Type;

// string|int
Type::union(Type::string(), Type::int());

// string|int|null (nullable union)
Type::union(Type::string(), Type::int(), Type::null());

// Multiple types
Type::union(
    Type::string(),
    Type::int(),
    Type::float(),
    Type::bool()
);
```

### Intersection Types

```php
use Symfony\Component\TypeInfo\Type;

// Stringable&Iterator
Type::intersection(
    Type::object(\Stringable::class),
    Type::object(\Iterator::class)
);

// Multiple interfaces
Type::intersection(
    Type::object(\Countable::class),
    Type::object(\ArrayAccess::class),
    Type::object(\Traversable::class)
);
```

### Object Types

```php
use Symfony\Component\TypeInfo\Type;

// Specific class
Type::object(MyClass::class);

// Interface
Type::object(MyInterface::class);

// Generic object (no specific class)
Type::object();
```

### Generic Types

```php
use Symfony\Component\TypeInfo\Type;

// Collection<int>
Type::generic(Type::object(Collection::class), Type::int());

// Collection<string, MyClass>
Type::generic(
    Type::object(Collection::class),
    Type::string(),
    Type::object(MyClass::class)
);

// ArrayObject<int, string>
Type::generic(
    Type::object(\ArrayObject::class),
    Type::int(),
    Type::string()
);
```

### Collection Types

```php
use Symfony\Component\TypeInfo\Type;

// List (int keys) - array<int, bool>
Type::list(Type::bool());

// Dictionary (string keys) - array<string, int>
Type::dict(Type::string(), Type::int());

// Nested collections - array<string, array<int, MyClass>>
Type::dict(
    Type::string(),
    Type::list(Type::object(MyClass::class))
);
```

### Creating Types from Values (Symfony 7.3+)

```php
use Symfony\Component\TypeInfo\Type;

Type::fromValue(42);         // Type::int()
Type::fromValue(3.14);       // Type::float()
Type::fromValue('hello');    // Type::string()
Type::fromValue(true);       // Type::true()
Type::fromValue(false);      // Type::false()
Type::fromValue(null);       // Type::null()
Type::fromValue([1, 2, 3]);  // Type::array()
Type::fromValue(new MyClass()); // Type::object(MyClass::class)
```

## Type Resolution

### Using TypeResolver

```php
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

// Create resolver instance
$typeResolver = TypeResolver::create();
```

### Resolving from Reflection

```php
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

class Example
{
    public int $intProperty;
    public ?string $nullableString;
    public array $arrayProperty;

    public function __construct(
        public int $id,
        /** @var string[] */
        public array $tags,
    ) {
    }

    public function getValue(): string|int
    {
        return $this->id;
    }
}

$typeResolver = TypeResolver::create();
$reflClass = new \ReflectionClass(Example::class);

// Resolve property type
$intType = $typeResolver->resolve($reflClass->getProperty('intProperty'));
// Result: int Type

// Resolve nullable property
$nullableType = $typeResolver->resolve($reflClass->getProperty('nullableString'));
// Result: nullable string Type

// Resolve constructor parameter
$reflMethod = $reflClass->getMethod('__construct');
$params = $reflMethod->getParameters();
$idType = $typeResolver->resolve($params[0]);
// Result: int Type

// Resolve with PHPDoc (requires phpstan/phpdoc-parser)
$tagsType = $typeResolver->resolve($reflClass->getProperty('tags'));
// Result: collection with int keys and string values

// Resolve return type
$returnType = $typeResolver->resolve($reflClass->getMethod('getValue'));
// Result: union of string and int
```

### Resolving from String

```php
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

$typeResolver = TypeResolver::create();

// Simple types
$typeResolver->resolve('int');     // int Type
$typeResolver->resolve('string');  // string Type
$typeResolver->resolve('bool');    // bool Type
$typeResolver->resolve('float');   // float Type

// Nullable
$typeResolver->resolve('?string'); // nullable string Type

// Union types
$typeResolver->resolve('string|int'); // union Type
```

### PHPDoc Parsing

With `phpstan/phpdoc-parser` installed, TypeResolver can parse complex PHPDoc annotations:

```php
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

class Dummy
{
    /** @var array<string, int> */
    public array $mapping;

    /** @var Collection<User> */
    public Collection $users;

    /** @var string[] */
    public array $names;

    /**
     * @param array<int, string> $items
     * @return iterable<string, mixed>
     */
    public function process(array $items): iterable
    {
        // ...
    }
}

$typeResolver = TypeResolver::create();
$reflClass = new \ReflectionClass(Dummy::class);

// Resolves to: array<string, int>
$mappingType = $typeResolver->resolve($reflClass->getProperty('mapping'));

// Resolves to: Collection<User>
$usersType = $typeResolver->resolve($reflClass->getProperty('users'));

// Resolves to: array<int, string>
$namesType = $typeResolver->resolve($reflClass->getProperty('names'));
```

## Type Introspection

### Type Identification

```php
use Symfony\Component\TypeInfo\Type;
use Symfony\Component\TypeInfo\TypeIdentifier;

$type = Type::int();

// Check specific type
$type->isIdentifiedBy(TypeIdentifier::INT);    // true
$type->isIdentifiedBy(TypeIdentifier::STRING); // false
$type->isIdentifiedBy(TypeIdentifier::FLOAT);  // false
```

### Union Type Identification

```php
use Symfony\Component\TypeInfo\Type;
use Symfony\Component\TypeInfo\TypeIdentifier;

$type = Type::union(Type::string(), Type::int());

// Matches any member of the union
$type->isIdentifiedBy(TypeIdentifier::INT);    // true
$type->isIdentifiedBy(TypeIdentifier::STRING); // true
$type->isIdentifiedBy(TypeIdentifier::FLOAT);  // false
```

### Object Type Identification

```php
use Symfony\Component\TypeInfo\Type;
use Symfony\Component\TypeInfo\TypeIdentifier;

class Dummy extends DummyParent implements DummyInterface {}

$type = Type::object(Dummy::class);

// Type identifier check
$type->isIdentifiedBy(TypeIdentifier::OBJECT); // true

// Class check
$type->isIdentifiedBy(Dummy::class);           // true

// Parent class check (inheritance)
$type->isIdentifiedBy(DummyParent::class);     // true

// Interface check (implementation)
$type->isIdentifiedBy(DummyInterface::class);  // true

// Unrelated class
$type->isIdentifiedBy(OtherClass::class);      // false
```

### Nullability Check

```php
use Symfony\Component\TypeInfo\Type;

$nullable = Type::nullable(Type::string());
$nullable->isNullable(); // true

$notNullable = Type::string();
$notNullable->isNullable(); // false

$unionWithNull = Type::union(Type::string(), Type::null());
$unionWithNull->isNullable(); // true
```

### Value Acceptance (Symfony 7.3+)

```php
use Symfony\Component\TypeInfo\Type;

$intType = Type::int();
$intType->accepts(123);    // true
$intType->accepts('abc');  // false
$intType->accepts(3.14);   // false
$intType->accepts(null);   // false

$unionType = Type::union(Type::string(), Type::int());
$unionType->accepts(123);   // true
$unionType->accepts('abc'); // true
$unionType->accepts(3.14);  // false

$nullableType = Type::nullable(Type::string());
$nullableType->accepts('abc'); // true
$nullableType->accepts(null);  // true
$nullableType->accepts(123);   // false
```

### Custom Validation with isSatisfiedBy

```php
use Symfony\Component\TypeInfo\Type;
use Symfony\Component\TypeInfo\TypeIdentifier;
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

class Entity
{
    private int $id;
    private string $name;
    private ?float $score;
    private \DateTimeInterface $createdAt;
}

$reflClass = new \ReflectionClass(Entity::class);
$resolver = TypeResolver::create();

// Define custom validation callable
$isNumeric = function (Type $type): bool {
    return $type->isIdentifiedBy(TypeIdentifier::INT)
        || $type->isIdentifiedBy(TypeIdentifier::FLOAT);
};

$isNonNullableNumeric = function (Type $type): bool {
    if ($type->isNullable()) {
        return false;
    }
    return $type->isIdentifiedBy(TypeIdentifier::INT)
        || $type->isIdentifiedBy(TypeIdentifier::FLOAT);
};

$idType = $resolver->resolve($reflClass->getProperty('id'));
$nameType = $resolver->resolve($reflClass->getProperty('name'));
$scoreType = $resolver->resolve($reflClass->getProperty('score'));

$idType->isSatisfiedBy($isNumeric);             // true
$nameType->isSatisfiedBy($isNumeric);           // false
$scoreType->isSatisfiedBy($isNumeric);          // true

$idType->isSatisfiedBy($isNonNullableNumeric);    // true
$scoreType->isSatisfiedBy($isNonNullableNumeric); // false (nullable)
```

## Collection Type Methods

### Getting Collection Key Type

```php
use Symfony\Component\TypeInfo\Type;

$listType = Type::list(Type::string());
$keyType = $listType->getCollectionKeyType();
// Result: int Type (lists always have int keys)

$dictType = Type::dict(Type::string(), Type::int());
$keyType = $dictType->getCollectionKeyType();
// Result: string Type
```

### Getting Collection Value Type

```php
use Symfony\Component\TypeInfo\Type;

$listType = Type::list(Type::string());
$valueType = $listType->getCollectionValueType();
// Result: string Type

$dictType = Type::dict(Type::string(), Type::object(User::class));
$valueType = $dictType->getCollectionValueType();
// Result: User object Type
```

### Method Chaining

```php
use Symfony\Component\TypeInfo\Type;
use Symfony\Component\TypeInfo\TypeIdentifier;

$type = Type::list(Type::nullable(Type::bool()));

// Get value type and check nullability
$isValueNullable = $type->getCollectionValueType()->isNullable();
// Result: true

// Get value type and check identifier
$isValueBool = $type->getCollectionValueType()->isIdentifiedBy(TypeIdentifier::BOOL);
// Result: true
```

## TypeIdentifier Enumeration

The `TypeIdentifier` enum provides constants for all PHP type identifiers:

```php
use Symfony\Component\TypeInfo\TypeIdentifier;

// Scalar types
TypeIdentifier::INT       // int
TypeIdentifier::STRING    // string
TypeIdentifier::FLOAT     // float
TypeIdentifier::BOOL      // bool

// Special boolean types
TypeIdentifier::TRUE      // true (literal)
TypeIdentifier::FALSE     // false (literal)

// Compound types
TypeIdentifier::ARRAY     // array
TypeIdentifier::OBJECT    // object
TypeIdentifier::CALLABLE  // callable
TypeIdentifier::ITERABLE  // iterable

// Special types
TypeIdentifier::NULL      // null
TypeIdentifier::MIXED     // mixed
TypeIdentifier::VOID      // void
TypeIdentifier::NEVER     // never
TypeIdentifier::RESOURCE  // resource
```

## Type String Representation

Types can be cast to strings for debugging or display:

```php
use Symfony\Component\TypeInfo\Type;

(string) Type::int();                    // "int"
(string) Type::nullable(Type::string()); // "?string"
(string) Type::union(Type::string(), Type::int()); // "string|int"
(string) Type::intersection(
    Type::object(\Stringable::class),
    Type::object(\Iterator::class)
);                                       // "Stringable&Iterator"
(string) Type::list(Type::bool());       // "array<int, bool>"
```

## Use Cases

### 1. Library Development

When building libraries that need type information:

```php
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;
use Symfony\Component\TypeInfo\TypeIdentifier;

class PropertyAnalyzer
{
    private TypeResolver $resolver;

    public function __construct()
    {
        $this->resolver = TypeResolver::create();
    }

    public function analyzeClass(string $className): array
    {
        $reflClass = new \ReflectionClass($className);
        $analysis = [];

        foreach ($reflClass->getProperties() as $property) {
            $type = $this->resolver->resolve($property);

            $analysis[$property->getName()] = [
                'type' => (string) $type,
                'nullable' => $type->isNullable(),
                'isNumeric' => $type->isIdentifiedBy(TypeIdentifier::INT)
                    || $type->isIdentifiedBy(TypeIdentifier::FLOAT),
            ];
        }

        return $analysis;
    }
}
```

### 2. Dependency Injection Type Validation

```php
use Symfony\Component\TypeInfo\Type;
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

class ServiceValidator
{
    private TypeResolver $resolver;

    public function validateConstructorArgs(string $className, array $args): bool
    {
        $reflClass = new \ReflectionClass($className);
        $constructor = $reflClass->getConstructor();

        if (!$constructor) {
            return empty($args);
        }

        foreach ($constructor->getParameters() as $i => $param) {
            if (!isset($args[$i])) {
                if (!$param->isOptional()) {
                    return false;
                }
                continue;
            }

            $expectedType = $this->resolver->resolve($param);
            $actualType = Type::fromValue($args[$i]);

            if (!$expectedType->accepts($args[$i])) {
                return false;
            }
        }

        return true;
    }
}
```

### 3. Form Field Type Detection

```php
use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;
use Symfony\Component\TypeInfo\TypeIdentifier;

class FormFieldGuesser
{
    private TypeResolver $resolver;

    public function guessFieldType(\ReflectionProperty $property): string
    {
        $type = $this->resolver->resolve($property);

        if ($type->isIdentifiedBy(TypeIdentifier::INT)) {
            return 'integer';
        }

        if ($type->isIdentifiedBy(TypeIdentifier::FLOAT)) {
            return 'number';
        }

        if ($type->isIdentifiedBy(TypeIdentifier::BOOL)) {
            return 'checkbox';
        }

        if ($type->isIdentifiedBy(TypeIdentifier::STRING)) {
            return 'text';
        }

        if ($type->isIdentifiedBy(\DateTimeInterface::class)) {
            return 'datetime';
        }

        return 'text';
    }
}
```

## Version Compatibility

| Feature | Symfony Version |
|---------|----------------|
| Core TypeInfo | 6.4+ |
| `Type::fromValue()` | 7.3+ |
| `Type::accepts()` | 7.3+ |
| Current documentation | 7.4 |

## Additional Resources

- **Official Documentation**: https://symfony.com/doc/7.4/components/type_info.html
- **GitHub Repository**: https://github.com/symfony/type-info
- **Related Component**: PropertyInfo component for extracting property metadata
