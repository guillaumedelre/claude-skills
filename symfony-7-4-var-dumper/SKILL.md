---
name: "symfony-7-4-var-dumper"
description: "Symfony 7.4 VarDumper component reference for debugging PHP variables with enhanced output formatting. Use when working with dump(), dd(), VarDumper class, Cloner classes (VarCloner), Dumper classes (HtmlDumper, CliDumper, ServerDumper), debug output formatting, DebugBundle integration, Casters, or any variable inspection-related Symfony code. Triggers on: VarDumper, dump(), dd(), Cloner, Dumper, debug output, DebugBundle, HtmlDumper, CliDumper, ServerDumper, VarCloner, Casters, dump server."
---

# Symfony 7.4 VarDumper Component

GitHub: https://github.com/symfony/var-dumper
Docs: https://symfony.com/doc/7.4/components/var_dumper.html

## Quick Reference

### Installation

```bash
# Standalone
composer require --dev symfony/var-dumper

# With Symfony (recommended)
composer require --dev symfony/debug-bundle
```

### Basic Usage - dump() and dd()

```php
// dump() - outputs variable and returns it (chainable)
dump($someVar);
dump($object)->someMethod();  // returns passed value

// dd() - "dump and die" - dumps then exits
dd($var1, $var2);
```

### Custom Handler

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

### Dump Server (Separate Debug Output)

```bash
# Start server
php bin/console server:dump

# Or use standalone binary
./vendor/bin/var-dump-server
```

```yaml
# config/packages/debug.yaml
debug:
    dump_destination: "tcp://%env(VAR_DUMPER_SERVER)%"
```

### Twig Integration

```twig
{% dump foo.bar %}      {# Dumps to toolbar #}
{{ dump(foo.bar) }}     {# Dumps inline in template #}
```

### Testing with VarDumperTestTrait

```php
use Symfony\Component\VarDumper\Test\VarDumperTestTrait;

class ExampleTest extends TestCase
{
    use VarDumperTestTrait;

    public function testDump(): void
    {
        $this->assertDumpEquals('[123, "foo"]', [123, 'foo']);
    }
}
```

### Cloner Configuration

```php
$cloner = new VarCloner();
$cloner->setMaxItems(100);      // Max items cloned (-1 = unlimited)
$cloner->setMinDepth(2);        // Minimum depth guaranteed
$cloner->setMaxString(1000);    // Max string length (-1 = unlimited)
```

### Dump as String

```php
$cloner = new VarCloner();
$dumper = new CliDumper();
$output = $dumper->dump($cloner->cloneVar($var), true);  // returns string
```

## Full Documentation

For complete details including all Dumper types, Caster implementation, Stub classes, dump server configuration, PHPUnit integration, property prefixes, and advanced customization, see [references/var-dumper.md](references/var-dumper.md).
