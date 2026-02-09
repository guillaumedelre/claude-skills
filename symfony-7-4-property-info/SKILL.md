---
name: "symfony-7-4-property-info"
description: "Symfony PropertyInfo component for extracting property metadata from PHP classes. Triggers on: PropertyInfo, type extraction, property metadata, reflection, docblock, PropertyInfoExtractorInterface, ReflectionExtractor, PhpDocExtractor, getTypes, getProperties, isReadable, isWritable, PropertyTypeExtractorInterface, PropertyAccessExtractorInterface, PropertyListExtractorInterface"
---

# Symfony PropertyInfo Component (7.4)

## Overview

The PropertyInfo component extracts metadata about PHP class properties using multiple sources (reflection, PHPDoc, Doctrine, Serializer). It provides type information, accessibility, descriptions, and initialization details.

**GitHub:** https://github.com/symfony/property-info
**Documentation:** https://symfony.com/doc/7.4/components/property_info.html

For complete documentation, see [references/property-info.md](references/property-info.md).

## Quick Reference
### Installation

```bash
composer require symfony/property-info
```

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
```

## Common Operations

### Get Property List

```php
$properties = $propertyInfo->getProperties(User::class);
// ['id', 'username', 'email', 'roles']
```

### Get Property Types

```php
$types = $propertyInfo->getTypes(User::class, 'email');
// Returns array of Type objects with builtin type, class name, nullable status
```

### Check Accessibility

```php
$propertyInfo->isReadable(User::class, 'username');  // true/false
$propertyInfo->isWritable(User::class, 'username');  // true/false
```

### Get Descriptions

```php
$short = $propertyInfo->getShortDescription(User::class, 'email');
$long = $propertyInfo->getLongDescription(User::class, 'email');
```

### Check Initialization

```php
$propertyInfo->isInitializable(User::class, 'username');  // true if settable via constructor
```

## Available Extractors

| Extractor | List | Types | Description | Access | Initializable |
|-----------|------|-------|-------------|--------|---------------|
| ReflectionExtractor | Yes | Yes | No | Yes | Yes |
| PhpDocExtractor | No | Yes | Yes | No | No |
| PhpStanExtractor (7.3+) | No | Yes | Yes | No | No |
| SerializerExtractor | Yes | No | No | No | No |
| DoctrineExtractor | Yes | Yes | No | No | No |
| ConstructorExtractor | No | Yes | No | No | No |

## Type Object Methods

```php
$type->getBuiltinType();       // 'string', 'int', 'array', 'object', etc.
$type->isNullable();           // bool
$type->getClassName();         // FQCN for object types
$type->isCollection();         // bool
$type->getCollectionKeyTypes();    // Type[] for collection keys
$type->getCollectionValueTypes();  // Type[] for collection values
```

## Service Tags (Symfony Framework)

Register custom extractors with these tags:

- `property_info.list_extractor`
- `property_info.type_extractor`
- `property_info.description_extractor`
- `property_info.access_extractor`
- `property_info.initializable_extractor`
- `property_info.constructor_extractor` (7.3+)

## Full Documentation

For complete details including advanced type extraction, custom extractors, Doctrine integration, and caching strategies, see [references/property-info.md](references/property-info.md).
