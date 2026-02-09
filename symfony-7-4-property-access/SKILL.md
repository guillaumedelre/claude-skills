---
name: "symfony-7-4-property-access"
description: "Symfony PropertyAccess component for reading/writing object and array properties using string notation. Triggers on: PropertyAccess, property paths, getter/setter, array access, magic methods, PropertyAccessor, getValue, setValue, isReadable, isWritable, dot notation, camelCase conversion, hassers, issers, adders, removers."
---

# Symfony PropertyAccess Component (7.4)

## Overview

The PropertyAccess component provides functions to read and write from/to objects or arrays using a simple string notation (property paths). It supports getters, setters, hassers, issers, adders, removers, and magic methods.

## Quick Reference

### Installation

```bash
composer require symfony/property-access
```

### Creating a PropertyAccessor

```php
use Symfony\Component\PropertyAccess\PropertyAccess;

$propertyAccessor = PropertyAccess::createPropertyAccessor();
```

### Reading Values

```php
// From arrays (use bracket notation)
$person = ['first_name' => 'Wouter'];
$propertyAccessor->getValue($person, '[first_name]'); // 'Wouter'

// From objects (use dot notation)
$propertyAccessor->getValue($person, 'firstName'); // calls getFirstName()

// Nested access
$propertyAccessor->getValue($person, 'children[0].firstName');

// Nullsafe operator
$propertyAccessor->getValue($comment, 'person?.firstName'); // returns null if person is null
```

### Writing Values

```php
// To arrays
$propertyAccessor->setValue($person, '[first_name]', 'Wouter');

// To objects
$propertyAccessor->setValue($person, 'firstName', 'Wouter'); // calls setFirstName()

// Uses adders/removers for collections
$propertyAccessor->setValue($person, 'children', ['kevin', 'wouter']); // calls addChild() for each
```

### Checking Readability/Writability

```php
$propertyAccessor->isReadable($person, 'firstName');  // true if readable
$propertyAccessor->isWritable($person, 'firstName');  // true if writable
```

### Configuring with Builder

```php
$propertyAccessor = PropertyAccess::createPropertyAccessorBuilder()
    ->enableMagicCall()                      // enable __call()
    ->enableMagicGet()                       // enable __get()
    ->enableMagicSet()                       // enable __set()
    ->enableExceptionOnInvalidIndex()        // throw on missing array index
    ->disableExceptionOnInvalidPropertyPath() // return null on missing property
    ->getPropertyAccessor();
```

### Property Name Resolution

- `first_name` -> `getFirstName()` / `setFirstName()`
- `author` -> `isAuthor()` / `hasAuthor()` (for boolean checks)
- `children` -> `addChild()` / `removeChild()` (for collections)

## Full Documentation
- Docs: https://symfony.com/doc/7.4/components/property_access.html
- GitHub: https://github.com/symfony/property-access
- **Full Reference**: See `references/property-access.md` for complete documentation
