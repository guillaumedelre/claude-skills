---
name: "symfony-7-4-options-resolver"
description: "Symfony 7.4 OptionsResolver component reference for validating and normalizing options arrays. Use when configuring option systems with defaults, required options, type validation, value constraints, normalization, nested options, or lazy defaults. Triggers on: OptionsResolver, option configuration, defaults, required options, validation, normalization, nested options, allowed values, allowed types, setDefined, setRequired, setDefault, setAllowedValues, setAllowedTypes, setNormalizer, resolve, setOptions, setPrototype, setDeprecated, define."
---

# Symfony 7.4 OptionsResolver Component

GitHub: https://github.com/symfony/options-resolver
Docs: https://symfony.com/doc/7.4/components/options_resolver.html

## Quick Reference

### Basic Setup

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
```

### Required & Defined Options

```php
// Required options (must be provided)
$resolver->setRequired(['host', 'username']);

// Defined options (allowed but no default)
$resolver->setDefined(['port', 'encryption']);
```

### Type Validation

```php
$resolver->setAllowedTypes('host', 'string');
$resolver->setAllowedTypes('port', ['null', 'int']);
$resolver->setAllowedTypes('port', 'int|null');  // Symfony 7.3+
$resolver->setAllowedTypes('dates', 'DateTime[]');
$resolver->setAllowedTypes('logger', 'Psr\Log\LoggerInterface');
```

### Value Validation

```php
$resolver->setAllowedValues('transport', ['sendmail', 'mail', 'smtp']);

// With closure
$resolver->setAllowedValues('port', function ($value): bool {
    return $value > 0 && $value < 65536;
});
```

### Normalization

```php
use Symfony\Component\OptionsResolver\Options;

$resolver->setNormalizer('host', function (Options $options, string $value): string {
    if (!str_starts_with($value, 'http://')) {
        $value = 'http://' . $value;
    }
    return $value;
});
```

### Lazy Defaults (Depending on Other Options)

```php
$resolver->setDefault('port', function (Options $options): int {
    return $options['encryption'] === 'ssl' ? 465 : 25;
});
```

### Nested Options (Symfony 7.3+)

```php
$resolver->setOptions('spool', function (OptionsResolver $spoolResolver): void {
    $spoolResolver->setDefaults([
        'type' => 'file',
        'path' => '/path/to/spool',
    ]);
    $spoolResolver->setAllowedValues('type', ['file', 'memory']);
});

// Usage
$mailer = new Mailer([
    'spool' => ['type' => 'memory'],
]);
```

### Chained Configuration

```php
$resolver->define('host')
    ->required()
    ->default('smtp.example.org')
    ->allowedTypes('string')
    ->info('The IP address or hostname');
```

### Deprecation

```php
$resolver->setDeprecated(
    'hostname',
    'acme/package',
    '1.2',
    'The option "%name%" is deprecated, use "host" instead.'
);
```

## Full Documentation

For complete details including prototype options, introspection, performance optimization, exception types, and all configuration patterns, see [references/options-resolver.md](references/options-resolver.md).
