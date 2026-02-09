# Symfony 7.4 VarDumper - Complete Reference

## Table of Contents

- [Installation](#installation)
- [Core Functions](#core-functions)
- [Custom Handler](#custom-handler)
- [The Dump Server](#the-dump-server)
- [DebugBundle and Twig Integration](#debugbundle-and-twig-integration)
- [Cloners](#cloners)
- [Dumpers](#dumpers)
- [Dumper Flags](#dumper-flags)
- [Casters](#casters)
- [Semantic Metadata Stubs](#semantic-metadata-stubs)
- [PHPUnit Integration](#phpunit-integration)
- [Environment Variables](#environment-variables)
- [Search in Dumps](#search-in-dumps)

---

## Installation

### Standalone Installation

```bash
composer require --dev symfony/var-dumper
```

In non-Symfony applications, ensure `vendor/autoload.php` is required.

### Symfony Application (Recommended)

```bash
composer require --dev symfony/debug-bundle
```

### Global Installation

Make `dump()` available globally across all PHP applications:

```bash
composer global require symfony/var-dumper
```

Add to `php.ini`:

```ini
auto_prepend_file = ${HOME}/.composer/vendor/autoload.php
```

Update periodically:

```bash
composer global update symfony/var-dumper
```

## Core Functions

### dump()

The `dump()` function provides:
- Specialized views for objects and resource types
- Configurable output formats (HTML or colored CLI)
- Internal reference tracking
- Output buffering compatibility

```php
require __DIR__.'/vendor/autoload.php';

$someVar = ['key' => 'value', 'nested' => ['a', 'b', 'c']];
dump($someVar);

// dump() returns the passed value, enabling method chaining
dump($someObject)->someMethod();

// Multiple variables
dump($var1, $var2, $var3);
```

Output format is selected automatically based on PHP SAPI:
- CLI SAPI: writes to `STDOUT`
- Other SAPIs: writes as HTML

### dd()

"Dump and Die" - dumps variables and immediately calls `exit()`:

```php
dd($variable);  // Equivalent to: dump($variable); exit();

// Multiple variables
dd($var1, $var2, $var3);
```

## Custom Handler

Override the default dump behavior:

```php
use Symfony\Component\VarDumper\Cloner\VarCloner;
use Symfony\Component\VarDumper\Dumper\CliDumper;
use Symfony\Component\VarDumper\Dumper\HtmlDumper;
use Symfony\Component\VarDumper\VarDumper;

VarDumper::setHandler(function (mixed $var): ?string {
    $cloner = new VarCloner();
    $dumper = 'cli' === PHP_SAPI ? new CliDumper() : new HtmlDumper();

    return $dumper->dump($cloner->cloneVar($var));
});
```

## The Dump Server

The dump server separates debug output from application output to avoid HTTP header corruption and keep HTML output clean.

### Starting the Server

```bash
# Output to console
php bin/console server:dump
  [OK] Server listening on tcp://0.0.0.0:9912

# Output to HTML file
php bin/console server:dump --format=html > dump.html

# Standalone binary
./vendor/bin/var-dump-server
```

### Symfony Configuration

**YAML:**

```yaml
# config/packages/debug.yaml
debug:
    dump_destination: "tcp://%env(VAR_DUMPER_SERVER)%"
```

**XML:**

```xml
<!-- config/packages/debug.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<container xmlns="http://symfony.com/schema/dic/services"
    xmlns:debug="http://symfony.com/schema/dic/debug">
    <debug:config dump-destination="tcp://%env(VAR_DUMPER_SERVER)%"/>
</container>
```

**PHP:**

```php
// config/packages/debug.php
return static function (ContainerConfigurator $container): void {
    $container->extension('debug', [
        'dump_destination' => 'tcp://%env(VAR_DUMPER_SERVER)%',
    ]);
};
```

### Non-Symfony Configuration

```php
use Symfony\Component\VarDumper\Cloner\VarCloner;
use Symfony\Component\VarDumper\Dumper\CliDumper;
use Symfony\Component\VarDumper\Dumper\HtmlDumper;
use Symfony\Component\VarDumper\Dumper\ServerDumper;
use Symfony\Component\VarDumper\Dumper\ContextProvider\CliContextProvider;
use Symfony\Component\VarDumper\Dumper\ContextProvider\SourceContextProvider;
use Symfony\Component\VarDumper\VarDumper;

$cloner = new VarCloner();
$fallbackDumper = \in_array(\PHP_SAPI, ['cli', 'phpdbg'])
    ? new CliDumper()
    : new HtmlDumper();

$dumper = new ServerDumper('tcp://127.0.0.1:9912', $fallbackDumper, [
    'cli' => new CliContextProvider(),
    'source' => new SourceContextProvider(),
]);

VarDumper::setHandler(function (mixed $var) use ($cloner, $dumper): ?string {
    return $dumper->dump($cloner->cloneVar($var));
});
```

## DebugBundle and Twig Integration

The DebugBundle configures `dump()` to output to the web debug toolbar, preventing HTTP header corruption.

### Twig Usage

```twig
{# Dumps to the web debug toolbar, doesn't modify template output #}
{% dump foo.bar %}

{# Dumps inline in the template output #}
{{ dump(foo.bar) }}

{# Dump all variables #}
{{ dump() }}
```

## Cloners

Cloners create an intermediate representation of PHP variables.

### VarCloner

```php
use Symfony\Component\VarDumper\Cloner\VarCloner;

$cloner = new VarCloner();
$data = $cloner->cloneVar($myVar);
```

### Configuration Methods

```php
$cloner = new VarCloner();

// Maximum items cloned past minimum depth (-1 = unlimited)
$cloner->setMaxItems(100);

// Minimum tree depth guaranteeing all items are cloned
$cloner->setMinDepth(2);

// Maximum string characters before truncation (-1 = unlimited)
$cloner->setMaxString(1000);
```

### Data Object Methods

```php
$data = $cloner->cloneVar($myVar);

// Limit depth
$data = $data->withMaxDepth(2);

// Limit items per level
$data = $data->withMaxItemsPerDepth(10);

// Remove internal reference handles
$data = $data->withRefHandles(false);

// Select sub-parts
$data = $data->seek(0, 1);
```

## Dumpers

### CliDumper

Outputs colored text for terminal:

```php
use Symfony\Component\VarDumper\Cloner\VarCloner;
use Symfony\Component\VarDumper\Dumper\CliDumper;

$cloner = new VarCloner();
$dumper = new CliDumper();
$dumper->dump($cloner->cloneVar($variable));
```

### HtmlDumper

Outputs HTML for browser:

```php
use Symfony\Component\VarDumper\Dumper\HtmlDumper;

$dumper = new HtmlDumper();

// Set theme ('dark' or 'light')
$dumper->setTheme('light');  // Default is 'dark'

// With options
$dumper->dump($cloner->cloneVar($var), null, [
    'maxDepth' => 1,
    'maxStringLength' => 160,
]);
```

### ServerDumper

Sends dumps to a remote server:

```php
use Symfony\Component\VarDumper\Dumper\ServerDumper;

$dumper = new ServerDumper('tcp://127.0.0.1:9912', $fallbackDumper, [
    'cli' => new CliContextProvider(),
    'source' => new SourceContextProvider(),
]);
```

### Getting Dump as String

**Using callback:**

```php
$cloner = new VarCloner();
$dumper = new CliDumper();
$output = '';

$dumper->dump(
    $cloner->cloneVar($variable),
    function (string $line, int $depth) use (&$output): void {
        if ($depth >= 0) {
            $output .= str_repeat('  ', $depth).$line."\n";
        }
    }
);
```

**Using memory stream:**

```php
$output = fopen('php://memory', 'r+b');
$dumper->dump($cloner->cloneVar($variable), $output);
$output = stream_get_contents($output, -1, 0);
```

**Return directly:**

```php
$output = $dumper->dump($cloner->cloneVar($variable), true);
```

## Dumper Flags

### Available Flags

```php
use Symfony\Component\VarDumper\Dumper\AbstractDumper;

// Show string length: 0 => (4) "test"
AbstractDumper::DUMP_STRING_LENGTH

// Light array format without "array" prefix
AbstractDumper::DUMP_LIGHT_ARRAY

// Use comma separator
AbstractDumper::DUMP_COMMA_SEPARATOR

// Trailing comma in arrays
AbstractDumper::DUMP_TRAILING_COMMA
```

### Using Flags

```php
use Symfony\Component\VarDumper\Dumper\AbstractDumper;
use Symfony\Component\VarDumper\Dumper\CliDumper;

// Single flag
$dumper = new CliDumper(null, null, AbstractDumper::DUMP_STRING_LENGTH);

// Multiple flags (combine with bitwise OR)
$dumper = new CliDumper(
    null,
    null,
    AbstractDumper::DUMP_STRING_LENGTH | AbstractDumper::DUMP_LIGHT_ARRAY
);
```

## Casters

Casters customize how specific classes or resources are represented.

### Implementing a Caster

```php
use Symfony\Component\VarDumper\Cloner\Stub;

function myCaster(
    mixed $object,
    array $array,
    Stub $stub,
    bool $isNested,
    int $filter
): array {
    // Modify or filter $array
    $array['customKey'] = 'customValue';
    unset($array['sensitiveData']);
    return $array;
}
```

### Registering Casters

```php
use Symfony\Component\VarDumper\Cloner\VarCloner;

$myCasters = [
    'FooClass' => 'myFooClassCaster',
    'App\Entity\User' => [UserCaster::class, 'cast'],
    ':bar resource' => 'myBarResourceCaster',  // prefix : for resources
];

// At construction
$cloner = new VarCloner($myCasters);

// Or add later
$cloner->addCasters($myCasters);
```

### Property Prefixes

When working with caster arrays, property names use special prefixes:

| Prefix | Meaning |
|--------|---------|
| `\0*\0` | Protected property |
| `\0ClassName\0` | Private property |
| `\0~\0` | Virtual (computed) property |
| `\0+\0` | Dynamic (runtime-added) property |

```php
function userCaster(User $user, array $array, Stub $stub): array
{
    // Access private property
    $privateValue = $array["\0User\0password"] ?? null;

    // Remove sensitive data
    unset($array["\0User\0password"]);

    // Add virtual property
    $array["\0~\0fullName"] = $user->getFirstName().' '.$user->getLastName();

    return $array;
}
```

## Semantic Metadata Stubs

Stubs add semantic meaning to dump values.

### Available Stubs

| Stub | Purpose |
|------|---------|
| `ConstStub` | PHP constant representation |
| `ClassStub` | PHP identifier (class, method, interface) |
| `CutStub` | Replace value with ellipses |
| `CutArrayStub` | Keep only specific array keys |
| `ImgStub` | Image wrapper |
| `EnumStub` | Virtual values set |
| `LinkStub` | Clickable links in HTML output |
| `TraceStub` | PHP trace |
| `FrameStub` | Single trace frame |
| `ArgsStub` | Function/method arguments |

### LinkStub Example

```php
use Symfony\Component\VarDumper\Caster\LinkStub;
use Symfony\Component\VarDumper\Cloner\Stub;

function productCaster(Product $object, array $array, Stub $stub): array
{
    // Make brochure URL clickable in HTML output
    $array['brochure'] = new LinkStub($array['brochure']);
    return $array;
}
```

### ClassStub Example

```php
use Symfony\Component\VarDumper\Caster\ClassStub;

function serviceCaster(Service $object, array $array, Stub $stub): array
{
    $array['handler'] = new ClassStub(SomeHandler::class);
    return $array;
}
```

## PHPUnit Integration

### VarDumperTestTrait

```php
use PHPUnit\Framework\TestCase;
use Symfony\Component\VarDumper\Test\VarDumperTestTrait;

class ExampleTest extends TestCase
{
    use VarDumperTestTrait;

    public function testDumpEquals(): void
    {
        $testedVar = [123, 'foo'];

        $expectedDump = <<<EOTXT
[
  123,
  "foo",
]
EOTXT;

        $this->assertDumpEquals($expectedDump, $testedVar);
    }

    public function testDumpMatchesFormat(): void
    {
        $this->assertDumpMatchesFormat('%d items', '5 items');
    }
}
```

### Configuring Test Casters and Flags

```php
use Symfony\Component\VarDumper\Cloner\Stub;
use Symfony\Component\VarDumper\Dumper\CliDumper;

protected function setUp(): void
{
    $casters = [
        \DateTimeInterface::class => static function (
            \DateTimeInterface $date,
            array $a,
            Stub $stub
        ): array {
            $stub->class = 'DateTime';
            return ['date' => $date->format('d/m/Y')];
        },
    ];

    $flags = CliDumper::DUMP_LIGHT_ARRAY | CliDumper::DUMP_COMMA_SEPARATOR;

    $this->setUpVarDumper($casters, $flags);
}

protected function tearDown(): void
{
    $this->tearDownVarDumper();
}
```

### Available Assertions

| Method | Description |
|--------|-------------|
| `assertDumpEquals($expected, $data)` | Exact dump match |
| `assertDumpMatchesFormat($expected, $data)` | Match with placeholders |
| `setUpVarDumper($casters, $flags)` | Configure casters and flags |
| `tearDownVarDumper()` | Reset configuration |

## Environment Variables

### VAR_DUMPER_FORMAT

Control output format via environment variable:

```bash
# Force HTML output
VAR_DUMPER_FORMAT=html php script.php

# Force CLI output
VAR_DUMPER_FORMAT=cli php script.php

# Send to dump server
VAR_DUMPER_FORMAT=server php script.php

# Custom server address
VAR_DUMPER_FORMAT=tcp://127.0.0.1:1234 php script.php
```

## Search in Dumps

In HTML output (browser):

- Click on dump contents and press `Ctrl+F` (or `Cmd+F` on macOS) for local search
- Use `Ctrl+G` (or `Cmd+G`) to navigate between results
- Press `Esc` to close search box

## Dump Examples

### Simple Array

```php
$var = [
    'a simple string' => "in an array of 5 elements",
    'a float' => 1.0,
    'an integer' => 1,
    'a boolean' => true,
    'an empty array' => [],
];
dump($var);
```

### Object with Properties

```php
class PropertyExample
{
    public string $publicProperty = 'The `+` prefix denotes public properties,';
    protected string $protectedProperty = '`#` protected ones and `-` private ones.';
    private string $privateProperty = 'Hovering a property shows a reminder.';
}

$var = new PropertyExample();
dump($var);
```

### Circular References

```php
class ReferenceExample
{
    public string $info = "Circular references are displayed as `#number`.";
}

$var = new ReferenceExample();
$var->aCircularReference = $var;
dump($var);
```

### Hard References

```php
$var = [];
$var[0] = 1;
$var[1] =& $var[0];  // Hard reference
$var[1] += 1;
dump($var);
// Output shows &1 prefix for referenced values
```

### Uninitialized Properties

```php
class Foo
{
    private int|float $foo;  // Uninitialized - shown specially
    public ?string $baz = null;
}

$var = new Foo();
dump($var);
```
