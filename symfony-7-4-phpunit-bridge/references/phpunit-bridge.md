# Symfony PHPUnit Bridge - Complete Reference

## Overview

The PHPUnit Bridge is a Symfony component that provides utilities for testing, including deprecation reporting, time/DNS mocking, and compatibility across multiple PHPUnit versions.

- **GitHub**: https://github.com/symfony/phpunit-bridge
- **Documentation**: https://symfony.com/doc/7.4/components/phpunit_bridge.html
- **License**: MIT
- **Latest Release**: v8.0.3

## Installation

```bash
composer require --dev symfony/phpunit-bridge
```

If installing outside Symfony, manually require the autoloader:

```php
require_once 'vendor/autoload.php';
```

## Core Features

### 1. Deprecation Reporting

The bridge automatically tracks and reports deprecated code usage during test execution:

```bash
./vendor/bin/simple-phpunit
```

Output categorizes deprecations as:
- **Unsilenced**: Deprecations triggered without the `@` operator
- **Legacy**: Tests explicitly testing deprecated features (marked with `@group legacy`)
- **Remaining/Other**: Other deprecation notices

### 2. Marking Tests as Legacy

Three methods to mark tests as legacy (testing deprecated features):

```php
// Method 1: @group annotation (Recommended)
/**
 * @group legacy
 */
public function testDeprecatedFeature(): void
{
    // Test deprecated functionality
}

// Method 2: Class name prefix
class LegacyMyTest extends TestCase
{
    // All tests in this class are legacy
}

// Method 3: Method name prefix
public function testLegacyFeature(): void
{
    // This specific test is legacy
}
```

### 3. Triggering Deprecation Notices

Use `trigger_deprecation()` to emit standardized deprecation notices:

```php
trigger_deprecation(
    'vendor-name/package-name',
    '1.3',
    'Your deprecation message'
);

// With format arguments
trigger_deprecation(
    'vendor-name/package-name',
    '1.3',
    'Value "%s" is deprecated, use "%s" instead',
    $oldValue,
    $newValue
);
```

### 4. Writing Assertions About Deprecations

Use `ExpectDeprecationTrait` to assert that code triggers expected deprecations:

```php
use PHPUnit\Framework\TestCase;
use Symfony\Bridge\PhpUnit\ExpectDeprecationTrait;

class MyTest extends TestCase
{
    use ExpectDeprecationTrait;

    /**
     * @group legacy
     */
    public function testDeprecatedCode(): void
    {
        // Expect this deprecation message (supports %s placeholders)
        $this->expectDeprecation('Since vendor-name/package-name 5.1: This "%s" method is deprecated');

        // Code that triggers the deprecation...

        // Can expect multiple deprecations in one test
        $this->expectDeprecation('Since vendor-name/package-name 4.4: The second argument is deprecated.');
    }
}
```

## SYMFONY_DEPRECATIONS_HELPER Configuration

### Making Tests Fail Based on Thresholds

Configure in `phpunit.xml.dist`:

```xml
<phpunit>
    <php>
        <env name="SYMFONY_DEPRECATIONS_HELPER" value="max[total]=320"/>
    </php>
</phpunit>
```

### Available Threshold Keys

| Key | Description |
|-----|-------------|
| `max[total]` | Total deprecation threshold |
| `max[self]` | Deprecations from your code only (excludes vendor/) |
| `max[direct]` | Direct deprecations (code you wrote directly) |
| `max[indirect]` | Deprecations from dependencies |

### Recommended Configurations

| Configuration | Use Case |
|---------------|----------|
| `max[total]=0` | Active projects with robust/no dependencies |
| `max[direct]=0` | Projects with dependencies lagging behind |
| `max[self]=0` | Libraries using deprecation system |

### Ignoring Deprecations with Patterns

Create a text file with regex patterns:

```text
# deprecation-ignore.txt
# Comments start with #
%The "Symfony\\Component\\Validator\\Context\\ExecutionContextInterface::.*\(\)" method is considered internal%
%The "PHPUnit\\Framework\\TestCase::addWarning\(\)" method%
```

Use the ignore file:

```bash
SYMFONY_DEPRECATIONS_HELPER='ignoreFile=./tests/baseline-ignore' ./vendor/bin/simple-phpunit
```

### Baseline Deprecations

Generate and maintain a baseline of allowed deprecations:

```bash
# Generate baseline
SYMFONY_DEPRECATIONS_HELPER='generateBaseline=true&baselineFile=./tests/allowed.json' ./vendor/bin/simple-phpunit

# Use baseline
SYMFONY_DEPRECATIONS_HELPER='baselineFile=./tests/allowed.json' ./vendor/bin/simple-phpunit
```

### Other Configuration Options

```php
// Disable verbose output
SYMFONY_DEPRECATIONS_HELPER=verbose=0

// Hide specific deprecation types
SYMFONY_DEPRECATIONS_HELPER='quiet[]=indirect&quiet[]=other'

// Disable deprecation helper entirely
SYMFONY_DEPRECATIONS_HELPER=disabled=1

// Log to file instead of stdout
SYMFONY_DEPRECATIONS_HELPER='logFile=/path/deprecations.log'

// Display stack trace for matching deprecations
SYMFONY_DEPRECATIONS_HELPER='/foobar/'

// Combined settings (URL-encoded format)
SYMFONY_DEPRECATIONS_HELPER='max[total]=42&max[self]=0&verbose=0'
```

## PHPUnit Configuration

Register the Symfony test listener in `phpunit.xml.dist`:

```xml
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/10.0/phpunit.xsd"
>
    <listeners>
        <listener class="Symfony\Bridge\PhpUnit\SymfonyTestsListener"/>
    </listeners>
</phpunit>
```

## Time-Sensitive Tests (ClockMock)

### Overview

ClockMock mocks PHP time functions to make time-sensitive tests instant and deterministic:
- `time()`
- `microtime()`
- `sleep()`
- `usleep()`
- `gmdate()`
- `hrtime()`
- `strtotime()`
- `date()`

### Using @group time-sensitive

```php
use PHPUnit\Framework\TestCase;
use Symfony\Component\Stopwatch\Stopwatch;

/**
 * @group time-sensitive
 */
class MyTest extends TestCase
{
    public function testSomething(): void
    {
        $stopwatch = new Stopwatch();

        $stopwatch->start('event_name');
        sleep(10);  // Instant in mocked time - no actual delay
        $duration = $stopwatch->stop('event_name')->getDuration();

        $this->assertEquals(10000, $duration);
    }
}
```

### Manual ClockMock Registration

```php
use Symfony\Bridge\PhpUnit\ClockMock;

public function testExample(): void
{
    // Register the class(es) that will use mocked time functions
    ClockMock::register(MyClass::class);

    // Enable clock mocking
    ClockMock::withClockMock(true);

    // Test code using mocked time...

    // Disable clock mocking
    ClockMock::withClockMock(false);
}
```

### Setting Specific Time

```php
use Symfony\Bridge\PhpUnit\ClockMock;

ClockMock::withClockMock(1234567890); // Set to specific Unix timestamp
```

## DNS-Sensitive Tests (DnsMock)

### Overview

DnsMock mocks DNS-related PHP functions:
- `checkdnsrr()`
- `dns_check_record()`
- `getmxrr()`
- `dns_get_mx()`
- `gethostbyaddr()`
- `gethostbyname()`
- `gethostbynamel()`
- `dns_get_record()`

### Using @group dns-sensitive

```php
use PHPUnit\Framework\TestCase;
use Symfony\Bridge\PhpUnit\DnsMock;

/**
 * @group dns-sensitive
 */
class DomainValidatorTest extends TestCase
{
    public function testEmails(): void
    {
        DnsMock::withMockedHosts([
            'example.com' => [
                ['type' => 'A', 'ip' => '1.2.3.4'],
                ['type' => 'AAAA', 'ipv6' => '::12'],
            ],
        ]);

        $validator = new DomainValidator(['checkDnsRecord' => true]);
        $isValid = $validator->validate('example.com');

        $this->assertTrue($isValid);
    }
}
```

### MX Record Mocking

```php
DnsMock::withMockedHosts([
    'example.com' => [
        ['type' => 'MX', 'host' => 'mail.example.com', 'pri' => 10],
    ],
]);
```

## Class Existence Mocking (ClassExistsMock)

### Overview

ClassExistsMock mocks class/interface/trait/enum existence checks:
- `class_exists()`
- `interface_exists()`
- `trait_exists()`
- `enum_exists()`

### Usage

```php
use PHPUnit\Framework\TestCase;
use Symfony\Bridge\PhpUnit\ClassExistsMock;
use Vendor\DependencyClass;

class MyClassTest extends TestCase
{
    public function testBehaviorWithoutDependency(): void
    {
        // Register the class that checks for existence
        ClassExistsMock::register(MyClass::class);

        // Mock the dependency as non-existent
        ClassExistsMock::withMockedClasses([
            DependencyClass::class => false
        ]);

        $class = new MyClass();
        $result = $class->hello();

        $this->assertEquals('The default behavior.', $result);
    }

    public function testBehaviorWithDependency(): void
    {
        ClassExistsMock::withMockedClasses([
            DependencyClass::class => true
        ]);

        $class = new MyClass();
        $result = $class->hello();

        $this->assertEquals('Using dependency.', $result);
    }
}
```

### Mocking Enums

```php
ClassExistsMock::withMockedEnums([
    MyEnum::class => true
]);
```

## Locale Configuration

Set the test locale (default is `C`):

```xml
<phpunit>
    <php>
        <env name="SYMFONY_PHPUNIT_LOCALE" value="fr_FR"/>
    </php>
</phpunit>
```

Or via environment variable:

```bash
SYMFONY_PHPUNIT_LOCALE="fr_FR" ./vendor/bin/simple-phpunit
```

## Parallel Test Execution

Run multiple test suites in parallel by organizing tests with separate `phpunit.xml.dist` files:

```
tests/
├── Functional/
│   ├── ...
│   └── phpunit.xml.dist
└── Unit/
    ├── ...
    └── phpunit.xml.dist
```

Run all suites:

```bash
./vendor/bin/simple-phpunit tests/
```

Control search depth:

```bash
SYMFONY_PHPUNIT_MAX_DEPTH=2 ./vendor/bin/simple-phpunit tests/
```

## simple-phpunit Script Features

The `simple-phpunit` script provides additional features over standard PHPUnit:

- Isolated vendor directory (no conflicts)
- Parallel test execution
- Skip/replay functionality via `SYMFONY_PHPUNIT_SKIPPED_TESTS`
- Multiple PHPUnit version support

### Environment Variables

| Variable | Description |
|----------|-------------|
| `SYMFONY_PHPUNIT_DIR` | Custom directory for PHPUnit installation |
| `SYMFONY_PHPUNIT_VERSION` | PHPUnit version to use (e.g., `5.5`) |
| `SYMFONY_MAX_PHPUNIT_VERSION` | Maximum version constraint (e.g., `9.5`) |
| `SYMFONY_PHPUNIT_REMOVE` | Packages to remove (e.g., `symfony/yaml`) |
| `SYMFONY_PHPUNIT_REQUIRE` | Additional packages to require |
| `SYMFONY_PHPUNIT_REMOVE_RETURN_TYPEHINT` | Remove void return types for legacy PHP |

## PHPUnit Version Compatibility

The bridge provides compatibility across PHPUnit versions:

- Polyfills for missing methods
- Namespaced aliases for non-namespaced classes
- Return type handling for legacy PHP

```php
// Modern PHPUnit 8+ syntax works with legacy versions
public function testExample(): void
{
    $this->expectException(RuntimeException::class);
    // Compatible with PHPUnit 4+
}
```

## Code Coverage Listener

Automatically add `@covers` annotations based on test class naming:

```xml
<phpunit>
    <listeners>
        <listener class="Symfony\Bridge\PhpUnit\CoverageListener"/>
    </listeners>
</phpunit>
```

### Custom SUT (System Under Test) Solver

```xml
<listeners>
    <listener class="Symfony\Bridge\PhpUnit\CoverageListener">
        <arguments>
            <string>My\Namespace\SutSolver::solve</string>
        </arguments>
    </listener>
</listeners>
```

## Troubleshooting

### Convention-based Namespace Resolution

If automatic namespace detection fails (removes `Tests\` from test namespace), configure explicitly:

```xml
<phpunit>
    <listeners>
        <listener class="Symfony\Bridge\PhpUnit\SymfonyTestsListener">
            <arguments>
                <array>
                    <element key="time-sensitive">
                        <string>Symfony\Component\HttpFoundation</string>
                    </element>
                </array>
            </arguments>
        </listener>
    </listeners>
</phpunit>
```

Or register in bootstrap file:

```php
// config/bootstrap.php
use Symfony\Bridge\PhpUnit\ClockMock;

if ('test' === $_SERVER['APP_ENV']) {
    ClockMock::register('Acme\\MyClassTest\\');
}
```

### Common Issues

1. **Deprecations not being detected**: Ensure `SymfonyTestsListener` is registered in `phpunit.xml.dist`

2. **ClockMock not working**: The class using time functions must be registered with `ClockMock::register()` or use `@group time-sensitive`

3. **DnsMock not affecting tests**: Ensure the class is in a namespace that gets mocked and use `@group dns-sensitive`

4. **Baseline file not reducing deprecations**: Ensure the file path is correct and regenerate if deprecation messages changed

## Complete phpunit.xml.dist Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/10.0/phpunit.xsd"
    bootstrap="vendor/autoload.php"
    colors="true"
    failOnRisky="true"
    failOnWarning="true"
>
    <php>
        <ini name="display_errors" value="1"/>
        <ini name="error_reporting" value="-1"/>
        <env name="SYMFONY_DEPRECATIONS_HELPER" value="max[direct]=0"/>
        <env name="SYMFONY_PHPUNIT_LOCALE" value="C"/>
    </php>

    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Functional">
            <directory>tests/Functional</directory>
        </testsuite>
    </testsuites>

    <coverage>
        <include>
            <directory suffix=".php">src</directory>
        </include>
    </coverage>

    <listeners>
        <listener class="Symfony\Bridge\PhpUnit\SymfonyTestsListener"/>
        <listener class="Symfony\Bridge\PhpUnit\CoverageListener"/>
    </listeners>
</phpunit>
```
