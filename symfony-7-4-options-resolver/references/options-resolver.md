# Symfony 7.4 OptionsResolver - Complete Reference

## Table of Contents

- [Installation](#installation)
- [Basic Usage](#basic-usage)
- [Setting Defaults](#setting-defaults)
- [Required Options](#required-options)
- [Defined Options](#defined-options)
- [Type Validation](#type-validation)
- [Value Validation](#value-validation)
- [Normalization](#normalization)
- [Lazy Options](#lazy-options)
- [Nested Options](#nested-options)
- [Prototype Options](#prototype-options)
- [Deprecating Options](#deprecating-options)
- [Chained Configuration](#chained-configuration)
- [Ignoring Undefined Options](#ignoring-undefined-options)
- [Performance Optimization](#performance-optimization)
- [Introspection](#introspection)
- [Exception Types](#exception-types)

---

## Installation

```bash
composer require symfony/options-resolver
```

## Basic Usage

### The Problem

Without OptionsResolver, handling configuration arrays requires boilerplate code that doesn't catch typos:

```php
class Mailer
{
    protected array $options;

    public function __construct(array $options = [])
    {
        $this->options = array_replace([
            'host'     => 'smtp.example.org',
            'username' => 'user',
            'password' => 'pa$$word',
            'port'     => 25,
        ], $options);
    }
}

// Typo 'usernme' silently ignored - bug!
$mailer = new Mailer(['usernme' => 'admin']);
```

### The Solution

```php
use Symfony\Component\OptionsResolver\OptionsResolver;

class Mailer
{
    protected array $options;

    public function __construct(array $options = [])
    {
        $resolver = new OptionsResolver();
        $this->configureOptions($resolver);
        $this->options = $resolver->resolve($options);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'host'       => 'smtp.example.org',
            'username'   => 'user',
            'password'   => 'pa$$word',
            'port'       => 25,
            'encryption' => null,
        ]);
    }
}

// Throws UndefinedOptionsException for 'usernme'
$mailer = new Mailer(['usernme' => 'admin']);
```

### Subclassing

```php
class GoogleMailer extends Mailer
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        parent::configureOptions($resolver);

        $resolver->setDefaults([
            'host'       => 'smtp.google.com',
            'encryption' => 'ssl',
        ]);
    }
}
```

## Setting Defaults

### setDefault / setDefaults

```php
// Multiple defaults
$resolver->setDefaults([
    'host'     => 'smtp.example.org',
    'username' => 'user',
    'password' => 'pa$$word',
    'port'     => 25,
]);

// Single default
$resolver->setDefault('host', 'smtp.example.org');
```

### Checking Defaults

```php
// Check if option has a default
$resolver->hasDefault('host'); // true/false

// Get all defined options
$resolver->getDefinedOptions();
```

## Required Options

Mark options that must be provided by the user:

```php
// Single option
$resolver->setRequired('host');

// Multiple options
$resolver->setRequired(['host', 'username', 'password']);
```

### Checking Required Status

```php
// Check if required
$resolver->isRequired('host'); // true

// Get all required options
$resolver->getRequiredOptions(); // ['host', 'username', 'password']

// Check if missing (required but no default)
$resolver->isMissing('host');

// Get all missing options
$resolver->getMissingOptions();
```

### Exception

```php
$resolver->setRequired('host');
$resolver->resolve([]);
// MissingOptionsException: The required option "host" is missing.
```

**Note:** A required option with a default value will use that default if not provided.

## Defined Options

Define options that are allowed but have no default. This lets you distinguish between user-provided values and defaults:

```php
$resolver->setDefined(['port', 'encryption']);

// Or single
$resolver->setDefined('port');
```

### Checking Defined Status

```php
$resolver->isDefined('port'); // true
$resolver->getDefinedOptions(); // all defined options
```

### Usage Pattern

```php
class Mailer
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefined('port');
    }

    public function sendMail(string $from, string $to): void
    {
        if (array_key_exists('port', $this->options)) {
            echo 'Port was explicitly set!';
        } else {
            echo 'Port was not provided.';
        }
    }
}

$mailer = new Mailer();
$mailer->sendMail($from, $to);
// => Port was not provided.

$mailer = new Mailer(['port' => 25]);
$mailer->sendMail($from, $to);
// => Port was explicitly set!
```

## Type Validation

### setAllowedTypes / addAllowedTypes

```php
// Single type
$resolver->setAllowedTypes('host', 'string');

// Multiple types (array syntax)
$resolver->setAllowedTypes('port', ['null', 'int']);

// Multiple types (pipe syntax, Symfony 7.3+)
$resolver->setAllowedTypes('port', 'int|null');

// Array type validation
$resolver->setAllowedTypes('dates', 'DateTime[]');
$resolver->setAllowedTypes('ports', 'int[]');
$resolver->setAllowedTypes('endpoints', '(int|string)[]');

// Class/Interface validation
$resolver->setAllowedTypes('logger', 'Psr\Log\LoggerInterface');

// Add types in subclass (extends allowed types)
$resolver->addAllowedTypes('port', 'string');
```

### Supported Type Strings

- Primitive types: `'string'`, `'int'`, `'float'`, `'bool'`, `'array'`, `'object'`, `'null'`
- Class/Interface names: `'DateTime'`, `'Psr\Log\LoggerInterface'`
- Array notation: `'int[]'`, `'DateTime[]'`, `'(int|string)[]'`
- Union types (7.3+): `'int|null'`, `'string|int'`

### Exception

```php
$resolver->setAllowedTypes('host', 'string');
$resolver->resolve(['host' => 25]);
// InvalidOptionsException: The option "host" with value "25" is
// expected to be of type "string", but is of type "int".
```

## Value Validation

### setAllowedValues / addAllowedValues

```php
// Static values
$resolver->setDefault('transport', 'sendmail');
$resolver->setAllowedValues('transport', ['sendmail', 'mail', 'smtp']);

// With closure for custom validation
$resolver->setAllowedValues('transport', function (string $value): bool {
    return in_array($value, ['sendmail', 'mail', 'smtp']);
});

// With Validator component
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Validation;

$resolver->setAllowedValues('transport', Validation::createIsValidCallable(
    new Assert\Length(min: 10)
));

// Port range validation
$resolver->setAllowedValues('port', function ($value): bool {
    return $value > 0 && $value < 65536;
});

// Add additional values in subclass
$resolver->addAllowedValues('transport', 'postfix');
```

### Exception

```php
$resolver->setAllowedValues('transport', ['sendmail', 'mail', 'smtp']);
$resolver->resolve(['transport' => 'send-mail']);
// InvalidOptionsException: The option "transport" with value "send-mail"
// is invalid. Accepted values are: "sendmail", "mail", "smtp".
```

## Normalization

Transform option values after validation:

### setNormalizer

```php
use Symfony\Component\OptionsResolver\Options;

$resolver->setNormalizer('host', function (Options $options, string $value): string {
    if (!str_starts_with($value, 'http://')) {
        $value = 'http://' . $value;
    }
    return $value;
});
```

### Normalizer Depending on Other Options

```php
$resolver->setNormalizer('host', function (Options $options, string $value): string {
    if (!str_starts_with($value, 'http://') && !str_starts_with($value, 'https://')) {
        if ('ssl' === $options['encryption']) {
            $value = 'https://' . $value;
        } else {
            $value = 'http://' . $value;
        }
    }
    return $value;
});
```

### addNormalizer

Add additional normalizers (appended by default):

```php
// Appended normalizer (receives previously normalized value)
$resolver->addNormalizer('host', function (Options $options, string $value): string {
    return strtolower($value);
});

// Prepended normalizer (receives original value)
$resolver->addNormalizer('host', function (Options $options, string $value): string {
    return strtoupper($value);
}, true);
```

## Lazy Options

### Default Values Depending on Another Option

Use closures for defaults that depend on other options:

```php
use Symfony\Component\OptionsResolver\Options;

$resolver->setDefault('encryption', null);

$resolver->setDefault('port', function (Options $options): int {
    if ('ssl' === $options['encryption']) {
        return 465;
    }
    return 25;
});
```

### Accessing Previous Default Value

In subclasses, access the parent's default value:

```php
class Mailer
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'encryption' => null,
            'host'       => 'example.org',
        ]);
    }
}

class GoogleMailer extends Mailer
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        parent::configureOptions($resolver);

        // Second parameter is the previous default value
        $resolver->setDefault('host', function (Options $options, string $previousValue): string {
            if ('ssl' === $options['encryption']) {
                return 'secure.example.org';
            }
            return $previousValue;  // 'example.org'
        });
    }
}
```

**Important:** Type hint the first parameter as `Options` or it will be treated as the previous value.

## Nested Options

Define hierarchical option structures (Symfony 7.3+):

### setOptions

```php
class Mailer
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setOptions('spool', function (OptionsResolver $spoolResolver): void {
            $spoolResolver->setDefaults([
                'type' => 'file',
                'path' => '/path/to/spool',
            ]);
            $spoolResolver->setAllowedValues('type', ['file', 'memory']);
            $spoolResolver->setAllowedTypes('path', 'string');
        });
    }

    public function sendMail(string $from, string $to): void
    {
        if ('memory' === $this->options['spool']['type']) {
            // Handle memory spool
        }
    }
}

$mailer = new Mailer([
    'spool' => [
        'type' => 'memory',
    ],
]);
```

### Nested Option Depending on Parent

Access parent options from nested resolver:

```php
$resolver->setDefault('sandbox', false);

$resolver->setOptions('spool', function (OptionsResolver $spoolResolver, Options $parent): void {
    $spoolResolver->setDefaults([
        'type' => $parent['sandbox'] ? 'memory' : 'file',
    ]);
});
```

### Parent Accessing Nested Options

```php
$resolver->setOptions('spool', function (OptionsResolver $spoolResolver): void {
    $spoolResolver->setDefaults([
        'type' => 'file',
    ]);
});

$resolver->setDefault('profiling', function (Options $options): bool {
    return $options['spool']['type'] === 'file';
});
```

## Prototype Options

Define repeating array structures with identical schemas:

```php
$resolver->setOptions('connections', function (OptionsResolver $connResolver): void {
    $connResolver
        ->setPrototype(true)
        ->setRequired(['host', 'database'])
        ->setDefaults(['user' => 'root', 'password' => null]);
});
```

### Usage

```php
$options = $resolver->resolve([
    'connections' => [
        'default' => [
            'host'     => '127.0.0.1',
            'database' => 'symfony',
        ],
        'test' => [
            'host'     => '127.0.0.1',
            'database' => 'symfony_test',
            'user'     => 'test',
            'password' => 'test',
        ],
    ],
]);

// Each entry in 'connections' is validated against the same schema
```

## Deprecating Options

### setDeprecated

```php
$resolver
    ->setDefined(['hostname', 'host'])

    // Generic message
    ->setDeprecated('hostname', 'acme/package', '1.2')
    // "Since acme/package 1.2: The option "hostname" is deprecated."

    // Custom message
    ->setDeprecated(
        'hostname',
        'acme/package',
        '1.2',
        'The option "%name%" is deprecated, use "host" instead.'
    );
```

### Conditional Deprecation

```php
$resolver
    ->setDefault('encryption', null)
    ->setDefault('port', null)
    ->setAllowedTypes('port', ['null', 'int'])
    ->setDeprecated('port', 'acme/package', '1.2', function (Options $options, ?int $value): string {
        if (null === $value) {
            return 'Passing "null" to option "port" is deprecated, pass an integer instead.';
        }

        if ('ssl' === $options['encryption'] && 456 !== $value) {
            return 'Passing a different port than "456" when "encryption" is "ssl" is deprecated.';
        }

        return '';  // Empty string = not deprecated
    });
```

### Suppressing Deprecation Warnings

```php
// Triggers deprecation
$options[$key];

// Suppresses warning (for internal library use)
$options->offsetGet($key, false);
```

## Chained Configuration

Use `define()` for readable, fluent configuration:

```php
class InvoiceMailer
{
    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->define('host')
            ->required()
            ->default('smtp.example.org')
            ->allowedTypes('string')
            ->info('The IP address or hostname');

        $resolver->define('transport')
            ->required()
            ->default('sendmail')
            ->allowedValues('sendmail', 'mail', 'smtp');

        $resolver->define('port')
            ->allowedTypes('int', 'null')
            ->default(25)
            ->normalize(function (Options $options, $value) {
                return (int) $value;
            });
    }
}
```

### Available Chain Methods

```php
$resolver->define('option_name')
    ->required()                          // Mark as required
    ->default($value)                     // Set default
    ->allowedTypes('type1', 'type2')      // Set allowed types
    ->allowedValues('val1', 'val2')       // Set allowed values
    ->allowedValues(fn($v) => $v > 0)     // Closure validation
    ->normalize(fn($opts, $v) => $v)      // Set normalizer
    ->deprecated('pkg', '1.0', 'msg')     // Mark deprecated
    ->info('Description');                // Set description
```

## Ignoring Undefined Options

Allow undefined options to pass through without error:

```php
$resolver
    ->setDefined(['hostname'])
    ->setIgnoreUndefined(true);

// 'version' is ignored instead of throwing exception
$resolver->resolve([
    'hostname' => 'acme/package',
    'version'  => '1.2.3',
]);
```

## Performance Optimization

Cache OptionsResolver instances to avoid reconfiguration overhead:

```php
class Mailer
{
    private static array $resolversByClass = [];
    protected array $options;

    public function __construct(array $options = [])
    {
        $class = $this::class;

        if (!isset(self::$resolversByClass[$class])) {
            self::$resolversByClass[$class] = new OptionsResolver();
            $this->configureOptions(self::$resolversByClass[$class]);
        }

        $this->options = self::$resolversByClass[$class]->resolve($options);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        // Configuration...
    }

    // Clear cache if needed (for long-running apps)
    public static function clearOptionsConfig(): void
    {
        self::$resolversByClass = [];
    }
}
```

## Introspection

Use `OptionsResolverIntrospector` to inspect resolver configuration:

```php
use Symfony\Component\OptionsResolver\Debug\OptionsResolverIntrospector;
use Symfony\Component\OptionsResolver\OptionsResolver;

$resolver = new OptionsResolver();
$resolver->setDefaults([
    'host' => 'smtp.example.org',
    'port' => 25,
]);
$resolver->setAllowedTypes('host', 'string');
$resolver->setRequired('host');

$introspector = new OptionsResolverIntrospector($resolver);

$introspector->getDefault('host');       // 'smtp.example.org'
$introspector->getAllowedTypes('host');  // ['string']
// etc.
```

### Available Introspection Methods

- `getDefault(string $option)` - Get default value
- `getAllowedTypes(string $option)` - Get allowed types
- `getAllowedValues(string $option)` - Get allowed values
- `getNormalizer(string $option)` - Get normalizer closure
- `getDeprecation(string $option)` - Get deprecation info

## Exception Types

| Exception | When Thrown |
|-----------|-------------|
| `UndefinedOptionsException` | Unknown option passed (not defined or with default) |
| `MissingOptionsException` | Required option not provided and no default |
| `InvalidOptionsException` | Type or value validation fails |
| `NoSuchOptionException` | Introspector method called for non-existent option |
| `OptionDefinitionException` | Invalid configuration (e.g., circular dependencies) |

## Method Summary

| Method | Purpose |
|--------|---------|
| `setDefault($option, $value)` | Set single default value |
| `setDefaults(array $defaults)` | Set multiple default values |
| `setRequired($optionNames)` | Mark option(s) as required |
| `setDefined($optionNames)` | Define option(s) without default |
| `setAllowedTypes($option, $types)` | Validate option type |
| `addAllowedTypes($option, $types)` | Add allowed types |
| `setAllowedValues($option, $values)` | Restrict to specific values |
| `addAllowedValues($option, $values)` | Add allowed values |
| `setNormalizer($option, $normalizer)` | Transform option value |
| `addNormalizer($option, $normalizer, $prepend)` | Add normalizer |
| `setOptions($option, $configurator)` | Define nested options |
| `setPrototype(bool $prototype)` | Enable prototype mode |
| `setDeprecated($option, $package, $version, $message)` | Mark as deprecated |
| `setIgnoreUndefined(bool $ignore)` | Allow undefined options |
| `define($option)` | Start chained configuration |
| `hasDefault($option)` | Check if has default |
| `isRequired($option)` | Check if required |
| `isMissing($option)` | Check if required without default |
| `isDefined($option)` | Check if defined |
| `getDefinedOptions()` | Get all defined options |
| `getRequiredOptions()` | Get all required options |
| `getMissingOptions()` | Get required options without defaults |
| `resolve(array $options)` | Resolve and validate options |
| `clear()` | Reset all configuration |
