---
name: "symfony-7-4-yaml"
description: "Symfony 7.4 YAML component reference for parsing and dumping YAML files. Use when working with YAML parsing, YAML dumping, Yaml::parse, Yaml::dump, Parser, Dumper, YAML flags, PARSE_CUSTOM_TAGS, DUMP_OBJECT, TaggedValue, or any YAML-related Symfony code. Triggers on: Yaml, YAML parsing, YAML dumping, Yaml::parse, Yaml::dump, Parser, Dumper, YAML flags, PARSE_CUSTOM_TAGS, DUMP_OBJECT."
---

# Symfony 7.4 YAML Component

## Overview

The Symfony YAML component loads and dumps YAML files. It parses YAML strings into PHP arrays and converts PHP arrays back into YAML strings. It provides a real YAML parser with clear error messages and full support for dates, integers, octal, booleans, merge keys, references, and aliases.

## Quick Reference

### Installation

```bash
composer require symfony/yaml
```

### Core Classes

| Class | Purpose |
|-------|---------|
| `Yaml` | Wrapper class for common parsing/dumping operations |
| `Parser` | Parses YAML strings to PHP values |
| `Dumper` | Dumps PHP values to YAML strings |
| `TaggedValue` | Represents custom-tagged YAML values |

### Basic Usage

```php
use Symfony\Component\Yaml\Yaml;
use Symfony\Component\Yaml\Exception\ParseException;

// Parse YAML string
$value = Yaml::parse("foo: bar");
// ['foo' => 'bar']

// Parse YAML file
$value = Yaml::parseFile('/path/to/file.yaml');

// Dump PHP array to YAML
$yaml = Yaml::dump(['foo' => 'bar', 'nested' => ['key' => 'value']]);

// Error handling
try {
    $value = Yaml::parse($content);
} catch (ParseException $e) {
    printf('Unable to parse YAML: %s', $e->getMessage());
}
```

### Dump Options

```php
// Control inline level (default: 2)
Yaml::dump($array, 2);        // Expand to 2 levels
Yaml::dump($array, 1);        // More inline

// Control indentation (default: 4)
Yaml::dump($array, 2, 4);     // 4 spaces
Yaml::dump($array, 2, 8);     // 8 spaces

// With flags
Yaml::dump($data, 2, 4, Yaml::DUMP_OBJECT | Yaml::DUMP_MULTI_LINE_LITERAL_BLOCK);
```

### Common Flags

**Parsing Flags:**
- `PARSE_CUSTOM_TAGS` - Parse custom tags as TaggedValue objects
- `PARSE_OBJECT` - Parse `!php/object` tags to objects
- `PARSE_OBJECT_FOR_MAP` - Parse mappings as objects
- `PARSE_DATETIME` - Parse dates as DateTime objects
- `PARSE_CONSTANT` - Parse `!php/const` and `!php/enum` tags
- `PARSE_EXCEPTION_ON_INVALID_TYPE` - Throw on invalid types

**Dumping Flags:**
- `DUMP_OBJECT` - Dump objects with `!php/object` tag
- `DUMP_OBJECT_AS_MAP` - Dump objects as YAML maps
- `DUMP_MULTI_LINE_LITERAL_BLOCK` - Use literal block style for multiline strings
- `DUMP_EXCEPTION_ON_INVALID_TYPE` - Throw on invalid types
- `DUMP_NULL_AS_TILDE` - Dump null as `~`
- `DUMP_NULL_AS_EMPTY` - Dump null as empty value (7.3+)
- `DUMP_NUMERIC_KEY_AS_STRING` - Quote numeric keys
- `DUMP_FORCE_DOUBLE_QUOTES_ON_VALUES` - Force double quotes (7.3+)
- `DUMP_COMPACT_NESTED_MAPPING` - Compact nested mappings (7.3+)

### Custom Tags

```php
use Symfony\Component\Yaml\Yaml;
use Symfony\Component\Yaml\Tag\TaggedValue;

// Parse custom tags
$data = "!my_tag { foo: bar }";
$parsed = Yaml::parse($data, Yaml::PARSE_CUSTOM_TAGS);
// TaggedValue('my_tag', ['foo' => 'bar'])

$tagName = $parsed->getTag();    // 'my_tag'
$tagValue = $parsed->getValue(); // ['foo' => 'bar']

// Dump custom tags
$data = new TaggedValue('my_tag', ['foo' => 'bar']);
$yaml = Yaml::dump($data);
// !my_tag { foo: bar }
```

## Full Documentation
- **Documentation:** https://symfony.com/doc/7.4/components/yaml.html
- **GitHub:** https://github.com/symfony/yaml (3.9k stars, MIT license)
- **Detailed Reference:** See [references/yaml.md](references/yaml.md)
