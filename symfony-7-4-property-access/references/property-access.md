# Symfony PropertyAccess Component - Full Reference

## Overview

The PropertyAccess component provides functions to read and write from/to objects or arrays using a simple string notation called "property paths".

- **GitHub**: https://github.com/symfony/property-access
- **Documentation**: https://symfony.com/doc/7.4/components/property_access.html
- **License**: MIT
- **PHP Requirement**: PHP 8.2+

## Installation

```bash
composer require symfony/property-access
```

## Creating a PropertyAccessor

```php
use Symfony\Component\PropertyAccess\PropertyAccess;

$propertyAccessor = PropertyAccess::createPropertyAccessor();
```

## Reading from Arrays

### Basic Array Access

Use bracket notation `[key]` to read array values:

```php
$person = [
    'first_name' => 'Wouter',
];

$propertyAccessor->getValue($person, '[first_name]'); // 'Wouter'
$propertyAccessor->getValue($person, '[age]');        // null (missing index)
```

### Handling Invalid Indices

By default, missing indices return `null`. Enable exceptions for stricter behavior:

```php
$propertyAccessor = PropertyAccess::createPropertyAccessorBuilder()
    ->enableExceptionOnInvalidIndex()
    ->getPropertyAccessor();

// Throws NoSuchIndexException
$value = $propertyAccessor->getValue($person, '[age]');

// Use nullsafe operator to avoid exception
$value = $propertyAccessor->getValue($person, '[age?]');
```

### Multidimensional Arrays

```php
$persons = [
    ['first_name' => 'Wouter'],
    ['first_name' => 'Ryan'],
];

$propertyAccessor->getValue($persons, '[0][first_name]'); // 'Wouter'
$propertyAccessor->getValue($persons, '[1][first_name]'); // 'Ryan'
```

### Escaping Special Characters

Escape dots and left brackets in array keys with backslash:

```php
$data = ['first.name' => 'Wouter'];
$propertyAccessor->getValue($data, '[first\.name]'); // 'Wouter'
```

Right brackets don't need escaping.

## Reading from Objects

### Accessing Public Properties

Use dot notation for object properties:

```php
class Person
{
    public string $firstName = 'Wouter';
}

$person = new Person();
$propertyAccessor->getValue($person, 'firstName'); // 'Wouter'
```

### Using Getters

The component automatically looks for getter methods with `get` prefix:

```php
class Person
{
    private string $firstName = 'Wouter';

    public function getFirstName(): string
    {
        return $this->firstName;
    }
}

$person = new Person();
$propertyAccessor->getValue($person, 'firstName');   // 'Wouter'
$propertyAccessor->getValue($person, 'first_name');  // 'Wouter' (snake_case converted)
```

Property names are automatically converted to camelCase (e.g., `first_name` -> `getFirstName()`).

### Using Hassers and Issers

Support for `is` and `has` prefixes for boolean-like properties:

```php
class Person
{
    private bool $author = true;
    private array $children = [];

    public function isAuthor(): bool
    {
        return $this->author;
    }

    public function hasChildren(): bool
    {
        return count($this->children) > 0;
    }
}

$person = new Person();
$propertyAccessor->getValue($person, 'author');   // true (calls isAuthor())
$propertyAccessor->getValue($person, 'children'); // true/false (calls hasChildren())
```

### Accessing Non-Existing Properties

By default, throws `NoSuchPropertyException`. Disable with:

```php
$propertyAccessor = PropertyAccess::createPropertyAccessorBuilder()
    ->disableExceptionOnInvalidPropertyPath()
    ->getPropertyAccessor();

// Returns null instead of throwing exception
$value = $propertyAccessor->getValue($person, 'birthday');
```

### Nullable Property Paths (Nullsafe Operator)

Use the `?` operator for nullable properties to avoid exceptions:

```php
class Comment
{
    public ?Person $person = null;
    public string $message;
}

$comment = new Comment();
$comment->message = 'test';

// Throws UnexpectedTypeException
$propertyAccessor->getValue($comment, 'person.firstname');

// Returns null safely
$propertyAccessor->getValue($comment, 'person?.firstname'); // null
```

### Magic `__get()` Method

```php
class Person
{
    private array $children = ['Wouter' => ['age' => 25]];

    public function __get($id): mixed
    {
        return $this->children[$id];
    }

    public function __isset($id): bool
    {
        return isset($this->children[$id]);
    }
}

$person = new Person();
$propertyAccessor->getValue($person, 'Wouter'); // ['age' => 25]
```

**Note:** Both `__get()` and `__isset()` must be implemented for magic get to work.

### Magic `__call()` Method

Enable via PropertyAccessorBuilder:

```php
class Person
{
    private array $children = ['wouter' => ['age' => 25]];

    public function __call($name, $args): mixed
    {
        $property = lcfirst(substr($name, 3));
        if ('get' === substr($name, 0, 3)) {
            return $this->children[$property] ?? null;
        }
    }
}

$propertyAccessor = PropertyAccess::createPropertyAccessorBuilder()
    ->enableMagicCall()
    ->getPropertyAccessor();

$propertyAccessor->getValue($person, 'wouter'); // ['age' => 25]
```

## Writing to Arrays

```php
$person = [];

$propertyAccessor->setValue($person, '[first_name]', 'Wouter');

$propertyAccessor->getValue($person, '[first_name]'); // 'Wouter'
$person['first_name']; // 'Wouter'
```

## Writing to Objects

### Basic Property Writing

```php
class Person
{
    public string $firstName;
    private string $lastName;

    public function setLastName(string $name): void
    {
        $this->lastName = $name;
    }

    public function getLastName(): string
    {
        return $this->lastName;
    }
}

$person = new Person();
$propertyAccessor->setValue($person, 'firstName', 'Wouter');  // direct property access
$propertyAccessor->setValue($person, 'lastName', 'de Jong');  // calls setLastName()

$person->firstName;        // 'Wouter'
$person->getLastName();    // 'de Jong'
```

### Using Magic `__call()` for Writing

```php
class Person
{
    private array $children = [];

    public function __call($name, $args): mixed
    {
        $property = lcfirst(substr($name, 3));
        if ('set' === substr($name, 0, 3)) {
            $value = count($args) ? $args[0] : null;
            $this->children[$property] = $value;
        }
    }
}

$propertyAccessor = PropertyAccess::createPropertyAccessorBuilder()
    ->enableMagicCall()
    ->getPropertyAccessor();

$propertyAccessor->setValue($person, 'wouter', ['age' => 25]);
```

### Writing to Array Properties (Adders/Removers)

The PropertyAccessor uses `add<Singular>()` and `remove<Singular>()` methods for collection properties:

```php
class Person
{
    private array $children = [];

    public function getChildren(): array
    {
        return $this->children;
    }

    public function addChild(string $name): void
    {
        $this->children[$name] = $name;
    }

    public function removeChild(string $name): void
    {
        unset($this->children[$name]);
    }
}

$person = new Person();
$propertyAccessor->setValue($person, 'children', ['kevin', 'wouter']);

$person->getChildren(); // ['kevin', 'wouter']
```

The singular form is derived automatically using the Symfony String component inflector.

### Using Non-Standard Adder/Remover Methods

Specify custom prefixes via `ReflectionExtractor`:

```php
use Symfony\Component\PropertyInfo\Extractor\ReflectionExtractor;
use Symfony\Component\PropertyAccess\PropertyAccessor;

class Team
{
    private array $team = [];

    public function getTeam(): array
    {
        return $this->team;
    }

    public function joinTeam(string $person): void
    {
        $this->team[] = $person;
    }

    public function leaveTeam(string $person): void
    {
        foreach ($this->team as $id => $item) {
            if ($person === $item) {
                unset($this->team[$id]);
                break;
            }
        }
    }
}

$reflectionExtractor = new ReflectionExtractor(null, null, ['join', 'leave']);
$propertyAccessor = new PropertyAccessor(
    PropertyAccessor::DISALLOW_MAGIC_METHODS,
    PropertyAccessor::THROW_ON_INVALID_PROPERTY_PATH,
    null,
    $reflectionExtractor,
    $reflectionExtractor
);

$team = new Team();
$propertyAccessor->setValue($team, 'team', ['kevin', 'wouter']);
$team->getTeam(); // ['kevin', 'wouter']
```

## Checking Property Paths

### Check if Readable

```php
$person = new Person();

if ($propertyAccessor->isReadable($person, 'firstName')) {
    // Safe to call getValue()
    $value = $propertyAccessor->getValue($person, 'firstName');
}
```

### Check if Writable

```php
$person = new Person();

if ($propertyAccessor->isWritable($person, 'firstName')) {
    // Safe to call setValue()
    $propertyAccessor->setValue($person, 'firstName', 'Wouter');
}
```

## Mixing Objects and Arrays

```php
class Person
{
    public string $firstName;
    private array $children = [];

    public function getChildren(): array
    {
        return $this->children;
    }

    public function setChildren(array $children): void
    {
        $this->children = $children;
    }
}

$person = new Person();

$propertyAccessor->setValue($person, 'children[0]', new Person());
// Equivalent to: $person->getChildren()[0] = new Person()

$propertyAccessor->setValue($person, 'children[0].firstName', 'Wouter');
// Equivalent to: $person->getChildren()[0]->firstName = 'Wouter'

$propertyAccessor->getValue($person, 'children[0].firstName'); // 'Wouter'
```

## PropertyAccessorBuilder Configuration

Use `PropertyAccessorBuilder` to configure features:

```php
$propertyAccessor = PropertyAccess::createPropertyAccessorBuilder()
    ->enableMagicCall()      // enables magic __call()
    ->enableMagicGet()       // enables magic __get()
    ->enableMagicSet()       // enables magic __set()
    ->enableMagicMethods()   // enables all magic methods
    ->getPropertyAccessor();
```

### Builder Configuration Methods

| Method | Description |
|--------|-------------|
| `enableMagicCall()` | Enable magic `__call()` method support |
| `disableMagicCall()` | Disable magic `__call()` method support |
| `enableMagicGet()` | Enable magic `__get()` method support |
| `disableMagicGet()` | Disable magic `__get()` method support |
| `enableMagicSet()` | Enable magic `__set()` method support |
| `disableMagicSet()` | Disable magic `__set()` method support |
| `enableMagicMethods()` | Enable all magic methods |
| `disableMagicMethods()` | Disable all magic methods |
| `isMagicCallEnabled()` | Check if `__call()` is enabled |
| `isMagicGetEnabled()` | Check if `__get()` is enabled |
| `isMagicSetEnabled()` | Check if `__set()` is enabled |
| `enableExceptionOnInvalidIndex()` | Throw exception on missing array index |
| `disableExceptionOnInvalidPropertyPath()` | Return null on missing property path |

### Constructor Alternative (Not Recommended)

```php
use Symfony\Component\PropertyAccess\PropertyAccessor;

// Enable magic __call and __set but not __get
$propertyAccessor = new PropertyAccessor(
    PropertyAccessor::MAGIC_CALL | PropertyAccessor::MAGIC_SET
);
```

## Key Classes and Interfaces

| Class/Interface | Description |
|-----------------|-------------|
| `PropertyAccessor` | Main class for reading/writing properties |
| `PropertyAccessorBuilder` | Builder for configuring PropertyAccessor |
| `PropertyAccess` | Factory class with `createPropertyAccessor()` |
| `PropertyAccessorInterface` | Interface defining getValue, setValue, isReadable, isWritable |
| `PropertyPathInterface` | Interface for property path objects |

## Exceptions

| Exception | When Thrown |
|-----------|-------------|
| `NoSuchPropertyException` | Property path does not exist (unless disabled) |
| `NoSuchIndexException` | Array index does not exist (when enabled) |
| `UnexpectedTypeException` | Trying to access property on null (without nullsafe) |
| `InvalidArgumentException` | Invalid property path syntax |

## Property Path Syntax Summary

| Syntax | Description | Example |
|--------|-------------|---------|
| `property` | Object property or getter | `firstName` -> `getFirstName()` |
| `[key]` | Array index | `[first_name]` |
| `.` | Property separator | `person.firstName` |
| `?` | Nullsafe operator | `person?.firstName` |
| `\.` | Escaped dot in key | `[first\.name]` |
| `\[` | Escaped bracket in key | `[first\[name]` |

## Best Practices

1. **Use the builder pattern** for configuration rather than constructor arguments
2. **Enable only needed magic methods** for better performance and predictability
3. **Use nullsafe operator** (`?`) when dealing with nullable properties
4. **Check isReadable/isWritable** before accessing unknown property paths
5. **Implement both `__get()` and `__isset()`** when using magic get methods
6. **Use adders/removers** for collection properties instead of direct setters
