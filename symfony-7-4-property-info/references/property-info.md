# Symfony PropertyInfo Component - Complete Reference

## Overview

The PropertyInfo component allows you to extract information about class properties using different metadata sources. Unlike the PropertyAccess component (which reads/writes values), PropertyInfo works with class definitions to provide type information, visibility, and accessibility details.

**GitHub:** https://github.com/symfony/property-info
**Documentation:** https://symfony.com/doc/7.4/components/property_info.html

## Installation

```bash
composer require symfony/property-info
```

Optional dependencies for additional extractors:
```bash
composer require phpdocumentor/reflection-docblock  # For PhpDocExtractor
composer require phpstan/phpdoc-parser              # For PhpStanExtractor (7.3+)
composer require symfony/serializer                  # For SerializerExtractor
composer require doctrine/orm symfony/doctrine-bridge  # For DoctrineExtractor
```

## Core Usage

### Basic Setup

```php
use Symfony\Component\PropertyInfo\Extractor\PhpDocExtractor;
use Symfony\Component\PropertyInfo\Extractor\ReflectionExtractor;
use Symfony\Component\PropertyInfo\PropertyInfoExtractor;

$phpDocExtractor = new PhpDocExtractor();
$reflectionExtractor = new ReflectionExtractor();

$propertyInfo = new PropertyInfoExtractor(
    listExtractors: [$reflectionExtractor],
    typeExtractors: [$phpDocExtractor, $reflectionExtractor],
    descriptionExtractors: [$phpDocExtractor],
    accessExtractors: [$reflectionExtractor],
    propertyInitializableExtractors: [$reflectionExtractor]
);

// Always pass class name, not object instance
$properties = $propertyInfo->getProperties(YourClass::class);
```

## Extractable Information

### 1. List Information - `getProperties()`

Returns array of property names:

```php
$properties = $propertyInfo->getProperties($class);
// Result: ['username', 'password', 'active']
```

### 2. Type Information - `getTypes()`

Returns array of `Type` objects with detailed type data:

```php
$types = $propertyInfo->getTypes($class, $property);
// Returns Type objects with built-in type, nullable status, class info, collection data
```

### 3. Documentation Block - `getDocBlock()` (Symfony 7.1+)

Extracts full DocComment as string:

```php
$docBlock = $propertyInfo->getDocBlock($class, $property);
```

### 4. Description Information - `getShortDescription()` / `getLongDescription()`

Extracts documentation descriptions:

```php
$title = $propertyInfo->getShortDescription($class, $property);
$paragraph = $propertyInfo->getLongDescription($class, $property);
```

### 5. Access Information - `isReadable()` / `isWritable()`

Checks property accessibility:

```php
$propertyInfo->isReadable($class, $property);  // bool
$propertyInfo->isWritable($class, $property);  // bool
```

### 6. Initializable Information - `isInitializable()`

Checks if property can be initialized via constructor:

```php
$propertyInfo->isInitializable($class, $property);  // bool
```

## Type Objects

The `Type` class provides six methods for detailed type information:

### `Type::getBuiltInType()`

Returns PHP built-in type: `array`, `bool`, `callable`, `float`, `int`, `iterable`, `null`, `object`, `resource`, `string`

```php
$type->getBuiltinType();  // 'string', 'int', etc.
```

### `Type::isNullable()`

Returns boolean if property can be null:

```php
$type->isNullable();  // bool
```

### `Type::getClassName()`

Returns fully-qualified class name for object types:

```php
$type->getClassName();  // 'App\Entity\User', etc.
```

### `Type::isCollection()`

Returns boolean if property is a collection:

```php
$type->isCollection();  // bool
```

Collections are detected via:
- Built-in type is `array`
- Method prefixes: `add` or `remove`
- PHPDoc annotations: `@var SomeClass<Type>`, `Collection<Entity>`

### `Type::getCollectionKeyTypes()` / `Type::getCollectionValueTypes()`

Returns `Type` objects for collection key/value types:

```php
$keyTypes = $type->getCollectionKeyTypes();
$valueTypes = $type->getCollectionValueTypes();
```

## Available Extractors

### 1. ReflectionExtractor

Uses PHP reflection to extract:
- List, type, and access information
- Constructor argument types
- Property initialization info
- Supports return and scalar type hints

```php
$reflectionExtractor = new ReflectionExtractor();
$reflectionExtractor->getProperties($class);
$reflectionExtractor->getTypes($class, $property);
$reflectionExtractor->isReadable($class, $property);
$reflectionExtractor->isWritable($class, $property);
$reflectionExtractor->isInitializable($class, $property);
```

**Note:** Automatically registered in Symfony when `property_info` is enabled.

### 2. PhpDocExtractor

Parses phpDocumentor comments for type and description info.

**Dependency:** `phpdocumentor/reflection-docblock`

```php
$phpDocExtractor = new PhpDocExtractor();
$phpDocExtractor->getTypes($class, $property);
$phpDocExtractor->getShortDescription($class, $property);
$phpDocExtractor->getLongDescription($class, $property);
$phpDocExtractor->getDocBlock($class, $property);  // 7.1+
```

### 3. PhpStanExtractor (Symfony 7.3+)

Uses PHPStan parser for `@var`, `@param`, `@return` annotations.

**Dependencies:** `phpstan/phpdoc-parser`, `phpdocumentor/reflection-docblock`

```php
$phpStanExtractor = new PhpStanExtractor();
$phpStanExtractor->getTypesFromConstructor($class, $property);
$phpStanExtractor->getShortDescription($class, $property);  // 7.3+
$phpStanExtractor->getLongDescription($class, $property);   // 7.3+
```

### 4. SerializerExtractor

Extracts list info from Serializer component groups metadata.

**Dependency:** `symfony/serializer`

```php
use Symfony\Component\PropertyInfo\Extractor\SerializerExtractor;
use Symfony\Component\Serializer\Mapping\Factory\ClassMetadataFactory;
use Symfony\Component\Serializer\Mapping\Loader\AttributeLoader;

$classMetadataFactory = new ClassMetadataFactory(new AttributeLoader());
$serializerExtractor = new SerializerExtractor($classMetadataFactory);
$serializerExtractor->getProperties($class, ['serializer_groups' => ['mygroup']]);
```

### 5. DoctrineExtractor

Extracts list and type info from Doctrine ORM entity mappings.

**Dependencies:** `symfony/doctrine-bridge`, `doctrine/orm`

```php
use Symfony\Component\PropertyInfo\Extractor\DoctrineExtractor;

$doctrineExtractor = new DoctrineExtractor($entityManager);
$doctrineExtractor->getProperties($class);
$doctrineExtractor->getTypes($class, $property);
```

### 6. ConstructorExtractor

Extracts property info from constructor arguments using PhpStanExtractor or ReflectionExtractor.

```php
use Symfony\Component\PropertyInfo\Extractor\ConstructorExtractor;

$constructorExtractor = new ConstructorExtractor([new ReflectionExtractor()]);
$constructorExtractor->getTypes(Foo::class, 'bar')[0]->getBuiltinType();
```

## Extractor Ordering

Order matters - first non-null result is returned. Recommended setup:

```php
$propertyInfo = new PropertyInfoExtractor(
    // List extractors - ReflectionExtractor first for all properties
    [$reflectionExtractor, $doctrineExtractor],
    // Type extractors - DoctrineExtractor first for accurate entity types
    [$doctrineExtractor, $reflectionExtractor],
    // Description extractors
    [$phpDocExtractor],
    // Access extractors
    [$reflectionExtractor],
    // Initializable extractors
    [$reflectionExtractor]
);
```

## Creating Custom Extractors

Implement one or more of these interfaces:

- `PropertyListExtractorInterface` - for list info
- `PropertyTypeExtractorInterface` - for type info
- `PropertyDescriptionExtractorInterface` - for descriptions
- `PropertyAccessExtractorInterface` - for access info
- `PropertyInitializableExtractorInterface` - for constructor initialization
- `ConstructorArgumentTypeExtractorInterface` - for constructor argument types

### Example Custom Extractor

```php
namespace App\PropertyInfo;

use Symfony\Component\PropertyInfo\PropertyListExtractorInterface;
use Symfony\Component\PropertyInfo\PropertyTypeExtractorInterface;

class CustomExtractor implements PropertyListExtractorInterface, PropertyTypeExtractorInterface
{
    public function getProperties(string $class, array $context = []): ?array
    {
        // Return property names or null to delegate to next extractor
        return null;
    }

    public function getTypes(string $class, string $property, array $context = []): ?array
    {
        // Return array of Type objects or null to delegate
        return null;
    }
}
```

### Register as Service (Symfony Framework)

```yaml
# config/services.yaml
services:
  App\PropertyInfo\CustomExtractor:
    tags:
      - property_info.list_extractor
      - property_info.type_extractor
      - property_info.description_extractor
      - property_info.access_extractor
      - property_info.initializable_extractor
      - property_info.constructor_extractor  # 7.3+
```

You can also set priority for ordering:

```yaml
services:
  App\PropertyInfo\CustomExtractor:
    tags:
      - { name: property_info.type_extractor, priority: 10 }
```

## Caching

Use `PropertyInfoCacheExtractor` to cache results:

```php
use Symfony\Component\PropertyInfo\PropertyInfoCacheExtractor;
use Symfony\Component\Cache\Adapter\ArrayAdapter;

$cache = new ArrayAdapter();
$cachedPropertyInfo = new PropertyInfoCacheExtractor($propertyInfo, $cache);
```

## Integration with Symfony Framework

When using the full Symfony framework, PropertyInfo is automatically configured:

```yaml
# config/packages/framework.yaml
framework:
    property_info:
        enabled: true
```

The service is then available as `property_info` or `Symfony\Component\PropertyInfo\PropertyInfoExtractorInterface`.

## Key Notes

- **Always pass class name:** `YourClass::class` not `$object`
- **PhpDocExtractor** can return multiple `Type` instances (e.g., `int|string`)
- **ReflectionExtractor** looks for getter/setter/isser/hasser methods (PSR-1 camelCase)
- **Collection pseudo-type** is returned as array with integer key type
- **Context array** can be passed to extractors for additional options

## Extractor Capabilities Matrix

| Extractor | List | Types | Description | Access | Initializable |
|-----------|------|-------|-------------|--------|---------------|
| ReflectionExtractor | Yes | Yes | No | Yes | Yes |
| PhpDocExtractor | No | Yes | Yes | No | No |
| PhpStanExtractor | No | Yes | Yes | No | No |
| SerializerExtractor | Yes | No | No | No | No |
| DoctrineExtractor | Yes | Yes | No | No | No |
| ConstructorExtractor | No | Yes | No | No | No |
