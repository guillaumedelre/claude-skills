---
name: "symfony-7-4-phpunit-bridge"
description: "Symfony PHPUnit Bridge component for testing utilities. Triggers on: PHPUnit Bridge, deprecation testing, clock mocking, DNS mocking, PHPUnit compatibility, ClockMock, DnsMock, SetUpTearDownTrait, ExpectDeprecationTrait, SYMFONY_DEPRECATIONS_HELPER, time-sensitive tests, simple-phpunit, deprecation notices, test mocking"
---

# Symfony PHPUnit Bridge

## Overview

The PHPUnit Bridge is a Symfony component that provides utilities for PHPUnit testing, including deprecation reporting, time/DNS/class mocking, and compatibility across multiple PHPUnit versions. It is essential for maintaining code quality and managing deprecations during Symfony upgrades.

- GitHub: https://github.com/symfony/phpunit-bridge
- Docs: https://symfony.com/doc/7.4/components/phpunit_bridge.html
- **Full Reference**: See `references/phpunit-bridge.md` for complete documentation

## Installation

```bash
composer require --dev symfony/phpunit-bridge
```

## Quick Reference

### Deprecation Testing with ExpectDeprecationTrait

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
        $this->expectDeprecation('Since vendor/package 5.1: Method "%s" is deprecated');

        // Code that triggers deprecation...
    }
}
```

### Triggering Deprecation Notices

```php
trigger_deprecation('vendor/package', '1.3', 'The "%s" method is deprecated', $methodName);
```

### Clock Mocking (Time-Sensitive Tests)

```php
use PHPUnit\Framework\TestCase;

/**
 * @group time-sensitive
 */
class TimeSensitiveTest extends TestCase
{
    public function testSleepIsMocked(): void
    {
        $start = time();
        sleep(10);  // Instant - no actual delay
        $elapsed = time() - $start;

        $this->assertEquals(10, $elapsed);
    }
}
```

### DNS Mocking

```php
use PHPUnit\Framework\TestCase;
use Symfony\Bridge\PhpUnit\DnsMock;

/**
 * @group dns-sensitive
 */
class DnsTest extends TestCase
{
    public function testDnsResolution(): void
    {
        DnsMock::withMockedHosts([
            'example.com' => [
                ['type' => 'A', 'ip' => '1.2.3.4'],
            ],
        ]);

        $this->assertEquals('1.2.3.4', gethostbyname('example.com'));
    }
}
```

### Class Existence Mocking

```php
use PHPUnit\Framework\TestCase;
use Symfony\Bridge\PhpUnit\ClassExistsMock;

class ClassExistsTest extends TestCase
{
    public function testWithoutDependency(): void
    {
        ClassExistsMock::register(MyClass::class);
        ClassExistsMock::withMockedClasses([
            SomeOptionalClass::class => false,
        ]);

        // Test behavior when optional class doesn't exist
    }
}
```

### SYMFONY_DEPRECATIONS_HELPER Configuration

```xml
<!-- phpunit.xml.dist -->
<phpunit>
    <php>
        <!-- Fail if more than 0 direct deprecations -->
        <env name="SYMFONY_DEPRECATIONS_HELPER" value="max[direct]=0"/>

        <!-- Baseline mode for existing deprecations -->
        <!-- <env name="SYMFONY_DEPRECATIONS_HELPER" value="baselineFile=./tests/allowed.json"/> -->

        <!-- Disable deprecation helper -->
        <!-- <env name="SYMFONY_DEPRECATIONS_HELPER" value="disabled=1"/> -->
    </php>
</phpunit>
```

### PHPUnit Configuration

```xml
<!-- phpunit.xml.dist -->
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/10.0/phpunit.xsd"
>
    <listeners>
        <listener class="Symfony\Bridge\PhpUnit\SymfonyTestsListener"/>
    </listeners>
</phpunit>
```

### Marking Legacy Tests

```php
// Method 1: @group annotation (recommended)
/**
 * @group legacy
 */
public function testDeprecatedFeature(): void { }

// Method 2: Class name prefix
class LegacyMyTest extends TestCase { }

// Method 3: Method name prefix
public function testLegacyFeature(): void { }
```

## Common Configuration Options

| Option | Description |
|--------|-------------|
| `max[total]=N` | Fail if total deprecations exceed N |
| `max[direct]=N` | Fail if direct deprecations exceed N |
| `max[self]=N` | Fail if deprecations from own code exceed N |
| `max[indirect]=N` | Fail if indirect deprecations exceed N |
| `disabled=1` | Disable deprecation helper entirely |
| `verbose=0` | Disable verbose deprecation output |
| `baselineFile=path` | Use baseline file for allowed deprecations |
| `ignoreFile=path` | Ignore deprecations matching patterns in file |

## Running Tests

```bash
# Run tests with simple-phpunit
./vendor/bin/simple-phpunit

# Generate deprecation baseline
SYMFONY_DEPRECATIONS_HELPER='generateBaseline=true&baselineFile=./tests/allowed.json' ./vendor/bin/simple-phpunit

# Run with specific deprecation threshold
SYMFONY_DEPRECATIONS_HELPER='max[direct]=0' ./vendor/bin/simple-phpunit
```
