# Symfony 7.4 YAML Component - Complete Reference

## Installation

```bash
composer require symfony/yaml
```

## Core Classes

### Yaml Class

The `Yaml` class is a wrapper that simplifies common parsing and dumping operations.

```php
use Symfony\Component\Yaml\Yaml;
```

**Static Methods:**
- `parse(string $input, int $flags = 0): mixed` - Parse a YAML string
- `parseFile(string $filename, int $flags = 0): mixed` - Parse a YAML file
- `dump(mixed $input, int $inline = 2, int $indent = 4, int $flags = 0): string` - Dump a PHP value to YAML

### Parser Class

Lower-level class for parsing YAML content.

```php
use Symfony\Component\Yaml\Parser;

$parser = new Parser();
$value = $parser->parse($yamlContent);
```

### Dumper Class

Lower-level class for dumping PHP values to YAML.

```php
use Symfony\Component\Yaml\Dumper;

$dumper = new Dumper();
$yaml = $dumper->dump($array);
```

### TaggedValue Class

Represents a YAML value with a custom tag.

```php
use Symfony\Component\Yaml\Tag\TaggedValue;

$tagged = new TaggedValue('my_tag', ['key' => 'value']);
$tag = $tagged->getTag();      // 'my_tag'
$value = $tagged->getValue();  // ['key' => 'value']
```

## Parsing YAML

### Parse a String

```php
use Symfony\Component\Yaml\Yaml;

$value = Yaml::parse("foo: bar");
// $value = ['foo' => 'bar']
```

### Parse a File

```php
$value = Yaml::parseFile('/path/to/file.yaml');
```

### Error Handling

```php
use Symfony\Component\Yaml\Yaml;
use Symfony\Component\Yaml\Exception\ParseException;

try {
    $value = Yaml::parse($yamlContent);
} catch (ParseException $exception) {
    printf('Unable to parse the YAML string: %s', $exception->getMessage());
}
```

## Dumping YAML

### Basic Dump

```php
use Symfony\Component\Yaml\Yaml;

$array = [
    'foo' => 'bar',
    'bar' => ['foo' => 'bar', 'bar' => 'baz'],
];

$yaml = Yaml::dump($array);
file_put_contents('/path/to/file.yaml', $yaml);
```

**Output (expanded, default):**
```yaml
foo: bar
bar:
    foo: bar
    bar: baz
```

### Inline Level Control

The second parameter controls how deep the array nesting goes before using inline notation.

```php
// Expand to 2 levels (default)
echo Yaml::dump($array, 2);
// foo: bar
// bar:
//     foo: bar
//     bar: baz

// Expand to 1 level only
echo Yaml::dump($array, 1);
// foo: bar
// bar: { foo: bar, bar: baz }

// Fully inline
echo Yaml::dump($array, 0);
// { foo: bar, bar: { foo: bar, bar: baz } }
```

### Indentation Control

The third parameter controls the number of spaces for indentation (default: 4).

```php
// Use 8 spaces for indentation
echo Yaml::dump($array, 2, 8);
```

## Parsing Flags

### PARSE_CUSTOM_TAGS

Parse custom tags as TaggedValue objects instead of throwing an exception.

```php
$data = "!my_tag { foo: bar }";
$parsed = Yaml::parse($data, Yaml::PARSE_CUSTOM_TAGS);

// $parsed is a TaggedValue object
$tagName = $parsed->getTag();    // 'my_tag'
$tagValue = $parsed->getValue(); // ['foo' => 'bar']
```

### PARSE_OBJECT

Parse `!php/object` tags back to PHP objects (via unserialize).

```php
$yaml = '!php/object \'O:8:"stdClass":1:{s:3:"foo";s:3:"bar";}\'';
$parsed = Yaml::parse($yaml, Yaml::PARSE_OBJECT);

var_dump(is_object($parsed)); // true
echo $parsed->foo;            // bar
```

### PARSE_OBJECT_FOR_MAP

Parse YAML mappings as stdClass objects instead of arrays.

```php
$yaml = "data:\n    foo: bar";
$parsed = Yaml::parse($yaml, Yaml::PARSE_OBJECT_FOR_MAP);

var_dump(is_object($parsed->data)); // true
echo $parsed->data->foo;             // bar
```

### PARSE_DATETIME

Parse date strings as DateTime objects instead of Unix timestamps.

```php
// Default behavior: Unix timestamp
Yaml::parse('2016-05-27');
// 1464307200

// With PARSE_DATETIME: DateTime object
$date = Yaml::parse('2016-05-27', Yaml::PARSE_DATETIME);
var_dump($date::class); // DateTime
```

### PARSE_CONSTANT

Parse `!php/const` tags to PHP constants and `!php/enum` tags to enum cases.

```php
$yaml = '{ foo: PHP_INT_SIZE, bar: !php/const PHP_INT_SIZE }';
$parameters = Yaml::parse($yaml, Yaml::PARSE_CONSTANT);
// ['foo' => 'PHP_INT_SIZE', 'bar' => 8]
```

**With Enumerations:**

```php
enum FooEnum: string {
    case Foo = 'foo';
    case Bar = 'bar';
}

$yaml = '{ foo: FooEnum::Foo, bar: !php/enum FooEnum::Foo }';
$parameters = Yaml::parse($yaml, Yaml::PARSE_CONSTANT);
// ['foo' => 'FooEnum::Foo', 'bar' => FooEnum::Foo]

// Get all enum cases
$yaml = '{ bar: !php/enum FooEnum }';
$parameters = Yaml::parse($yaml, Yaml::PARSE_CONSTANT);
// ['bar' => ['foo', 'bar']]
```

### PARSE_EXCEPTION_ON_INVALID_TYPE

Throw an exception when encountering invalid types during parsing.

```php
$yaml = '!php/object \'O:8:"stdClass":1:{s:3:"foo";s:3:"bar";}\'';
Yaml::parse($yaml, Yaml::PARSE_EXCEPTION_ON_INVALID_TYPE);
// Throws ParseException
```

## Dumping Flags

### DUMP_OBJECT

Dump PHP objects using the `!php/object` tag (via serialize).

```php
$object = new \stdClass();
$object->foo = 'bar';

$dumped = Yaml::dump($object, 2, 4, Yaml::DUMP_OBJECT);
// !php/object 'O:8:"stdClass":1:{s:3:"foo";s:3:"bar";}'
```

### DUMP_OBJECT_AS_MAP

Dump PHP objects as YAML mappings (key-value pairs).

```php
$object = new \stdClass();
$object->foo = 'bar';

$dumped = Yaml::dump(['data' => $object], 2, 4, Yaml::DUMP_OBJECT_AS_MAP);
// data:
//     foo: bar
```

### DUMP_MULTI_LINE_LITERAL_BLOCK

Dump multi-line strings using YAML literal block style (`|`).

```php
$string = ["string" => "Multiple\nLine\nString"];

// Default: inline with escape sequences
$yaml = Yaml::dump($string);
// string: "Multiple\nLine\nString"

// With literal block style
$yaml = Yaml::dump($string, 2, 4, Yaml::DUMP_MULTI_LINE_LITERAL_BLOCK);
// string: |
//     Multiple
//     Line
//     String
```

### DUMP_EXCEPTION_ON_INVALID_TYPE

Throw an exception when encountering types that cannot be dumped.

```php
$data = new \stdClass();
Yaml::dump($data, 2, 4, Yaml::DUMP_EXCEPTION_ON_INVALID_TYPE);
// Throws DumpException
```

### DUMP_NULL_AS_TILDE

Dump null values as `~` instead of `null`.

```php
// Default
$dumped = Yaml::dump(['foo' => null]);
// foo: null

// As tilde
$dumped = Yaml::dump(['foo' => null], 2, 4, Yaml::DUMP_NULL_AS_TILDE);
// foo: ~
```

### DUMP_NULL_AS_EMPTY (Symfony 7.3+)

Dump null values as empty values.

```php
$dumped = Yaml::dump(['foo' => null], 2, 4, Yaml::DUMP_NULL_AS_EMPTY);
// foo:
```

### DUMP_NUMERIC_KEY_AS_STRING

Quote numeric keys to ensure they remain strings.

```php
// Default
$dumped = Yaml::dump([200 => 'foo']);
// 200: foo

// As string
$dumped = Yaml::dump([200 => 'foo'], 2, 4, Yaml::DUMP_NUMERIC_KEY_AS_STRING);
// '200': foo
```

### DUMP_FORCE_DOUBLE_QUOTES_ON_VALUES (Symfony 7.3+)

Force double quotes on string values.

```php
$dumped = Yaml::dump([
    'foo' => 'bar',
    'some foo' => 'some bar',
    'x' => 3.14,
], 2, 4, Yaml::DUMP_FORCE_DOUBLE_QUOTES_ON_VALUES);
// "foo": "bar", "some foo": "some bar", "x": 3.14
```

### DUMP_COMPACT_NESTED_MAPPING (Symfony 7.3+)

Use compact style for nested mappings within sequences.

```php
$planets = [
    'planets' => [
        ['name' => 'Mercury', 'distance' => 57910000],
        ['name' => 'Venus', 'distance' => 108200000],
    ]
];

// Default
$dumped = Yaml::dump($planets, 4);
// planets:
//     -
//         name: Mercury
//         distance: 57910000

// Compact
$dumped = Yaml::dump($planets, 4, 4, Yaml::DUMP_COMPACT_NESTED_MAPPING);
// planets:
//     - name: Mercury
//       distance: 57910000
```

## Combining Flags

Flags can be combined using the bitwise OR operator (`|`).

```php
$yaml = Yaml::dump($data, 2, 4,
    Yaml::DUMP_OBJECT_AS_MAP |
    Yaml::DUMP_MULTI_LINE_LITERAL_BLOCK |
    Yaml::DUMP_NULL_AS_TILDE
);

$parsed = Yaml::parse($content,
    Yaml::PARSE_DATETIME |
    Yaml::PARSE_CUSTOM_TAGS
);
```

## Binary Data

Binary data is automatically base64 encoded/decoded with the `!!binary` tag.

```php
// Dump binary data
$imageContents = file_get_contents(__DIR__.'/images/logo.png');
$dumped = Yaml::dump(['logo' => $imageContents]);
// logo: !!binary iVBORw0KGgoAAAANSUhEUgAAA6oaAADqCAY...

// Parse binary data
$dumped = 'logo: !!binary iVBORw0KGgoAAAANSUhEUgAAA6oaAADqCAY...';
$parsed = Yaml::parse($dumped);
$imageContents = $parsed['logo'];
```

## Numeric Literals

YAML supports underscores in numeric literals for readability.

```yaml
parameters:
    credit_card_number: 1234_5678_9012_3456
    long_number: 10_000_000_000
    pi: 3.14159_26535_89793
    hex_words: 0x_CAFE_F00D
```

Underscores are removed during parsing:

```php
$value = Yaml::parse("number: 1_000_000");
// ['number' => 1000000]
```

## YAML Linting (CLI)

### Setup

```bash
composer require symfony/console
```

Create a lint script:

```php
// lint.php
use Symfony\Component\Console\Application;
use Symfony\Component\Yaml\Command\LintCommand;

(new Application('yaml/lint'))
    ->add(new LintCommand())
    ->getApplication()
    ->setDefaultCommand('lint:yaml', true)
    ->run();
```

### Usage

```bash
# Lint a single file
php lint.php path/to/file.yaml

# Lint multiple files
php lint.php path/to/file1.yaml path/to/file2.yaml

# Lint a directory
php lint.php path/to/directory

# Lint from STDIN
cat path/to/file.yaml | php lint.php

# Exclude files
php lint.php path/to/directory --exclude=path/to/file.yaml

# JSON output format
php lint.php path/to/file.yaml --format=json
```

## Exception Classes

### ParseException

Thrown when YAML parsing fails.

```php
use Symfony\Component\Yaml\Exception\ParseException;

try {
    Yaml::parse($invalidYaml);
} catch (ParseException $e) {
    echo $e->getMessage();
    echo $e->getParsedFile();  // File being parsed (if any)
    echo $e->getParsedLine();  // Line number where error occurred
    echo $e->getSnippet();     // Code snippet around the error
}
```

### DumpException

Thrown when YAML dumping fails (with DUMP_EXCEPTION_ON_INVALID_TYPE).

```php
use Symfony\Component\Yaml\Exception\DumpException;

try {
    Yaml::dump($resource, 2, 4, Yaml::DUMP_EXCEPTION_ON_INVALID_TYPE);
} catch (DumpException $e) {
    echo $e->getMessage();
}
```

## Best Practices

1. **Always handle ParseException** when parsing user-provided or external YAML
2. **Use appropriate inline levels** - deeper nesting (higher values) produces more readable YAML
3. **Use PARSE_CUSTOM_TAGS** when working with application-specific tags
4. **Prefer DUMP_OBJECT_AS_MAP** over DUMP_OBJECT for more portable YAML
5. **Use DUMP_MULTI_LINE_LITERAL_BLOCK** for configuration files with long strings
6. **Validate YAML files** using the lint command in CI/CD pipelines

## Resources

- **Documentation:** https://symfony.com/doc/7.4/components/yaml.html
- **GitHub Repository:** https://github.com/symfony/yaml
- **License:** MIT
- **Latest Release:** v8.0.1 (December 2025)
