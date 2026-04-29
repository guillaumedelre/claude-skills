---
name: "phpunit-12"
description: "PHPUnit 12 reference for unit testing in PHP. Use when writing tests, configuring phpunit.xml, working with assertions, data providers, mocks/stubs, attributes, code coverage, or risky-test settings. Triggers on: PHPUnit 12, TestCase, dataProvider, attributes #[Test], #[DataProvider], #[DataProviderExternal], #[CoversClass], #[CoversMethod], #[UsesClass], #[Group], #[TestDox], #[Depends], #[DependsExternal], #[DependsOnClass], #[BackupGlobals], #[BackupStaticProperties], #[PreserveGlobalState], #[RunInSeparateProcess], #[RunTestsInSeparateProcesses], #[RequiresPhp], #[RequiresPhpunit], #[RequiresOperatingSystem], #[RequiresFunction], #[RequiresMethod], #[Before], #[After], #[BeforeClass], #[AfterClass], #[Ticket], assertSame, assertEquals, assertInstanceOf, assertCount, assertStringContainsString, assertMatchesRegularExpression, expectException, expectExceptionMessage, expectNotToPerformAssertions, MockObject, createMock, createStub, createPartialMock, createMockForIntersectionOfInterfaces, willReturn, willReturnCallback, willReturnArgument, willThrowException, willReturnSelf, atLeast, exactly, never, once, any matcher, phpunit.xml, bootstrap, test suites, coverage, baseline, deprecation handling, --strict-coverage, --strict-global-state."
---

# PHPUnit 12

Docs: https://docs.phpunit.de/en/12.0/index.html
GitHub: https://github.com/sebastianbergmann/phpunit
Release: https://phpunit.de/announcements/phpunit-12.html

## PHPUnit 12 Highlights

- **Doc-comment annotations removed** — only PHP 8 attributes work for `@test`, `@dataProvider`, `@covers`, etc.
- `willReturnOnConsecutiveCalls()` and `withConsecutive()` removed (deprecated in 10/11). Use `$matcher = $this->exactly(N)` with callbacks or `willReturnCallback()`.
- New strict mode flags for source coverage and global state.
- Requires PHP 8.3+.

## Installation

```bash
composer require --dev phpunit/phpunit:^12.0
```

## phpunit.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
    bootstrap="tests/bootstrap.php"
    colors="true"
    cacheDirectory=".phpunit.cache"
    requireCoverageMetadata="true"
    beStrictAboutCoverageMetadata="true"
    beStrictAboutOutputDuringTests="true"
    failOnRisky="true"
    failOnWarning="true"
    failOnDeprecation="true"
    failOnNotice="true"
>
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory>tests/Integration</directory>
        </testsuite>
    </testsuites>

    <source restrictDeprecations="true" restrictNotices="true" restrictWarnings="true">
        <include>
            <directory>src</directory>
        </include>
        <exclude>
            <directory>src/Migrations</directory>
        </exclude>
    </source>

    <coverage>
        <report>
            <html outputDirectory="var/coverage" />
            <text outputFile="php://stdout" />
            <clover outputFile="var/clover.xml" />
        </report>
    </coverage>

    <php>
        <env name="APP_ENV" value="test" />
        <env name="KERNEL_CLASS" value="App\Kernel" />
    </php>
</phpunit>
```

## Quick Reference

### Basic Test

```php
namespace App\Tests\Unit;

use PHPUnit\Framework\Attributes\Test;
use PHPUnit\Framework\TestCase;

final class CalculatorTest extends TestCase
{
    #[Test]
    public function it_adds_two_numbers(): void
    {
        $result = (new Calculator())->add(2, 3);
        $this->assertSame(5, $result);
    }
}
```

Without `#[Test]`, the method must be named `test*`:

```php
public function testAddsTwoNumbers(): void { /* ... */ }
```

### Common Assertions

```php
$this->assertSame($expected, $actual);                        // === comparison
$this->assertEquals($expected, $actual);                      // == comparison
$this->assertNotSame(...);
$this->assertTrue($actual);
$this->assertFalse($actual);
$this->assertNull($actual);
$this->assertNotNull($actual);
$this->assertEmpty($actual);
$this->assertNotEmpty($actual);
$this->assertCount(3, $array);
$this->assertContains('needle', $haystack);
$this->assertContainsEquals(...);
$this->assertContainsOnly('integer', $array);
$this->assertContainsOnlyInstancesOf(User::class, $users);

$this->assertInstanceOf(MyClass::class, $obj);
$this->assertIsArray($var);
$this->assertIsString($var);
$this->assertIsInt($var);
$this->assertIsFloat($var);
$this->assertIsBool($var);
$this->assertIsCallable($var);

$this->assertStringContainsString('foo', $haystack);
$this->assertStringStartsWith('Hello', $string);
$this->assertStringEndsWith('!', $string);
$this->assertMatchesRegularExpression('/^\d+$/', $string);
$this->assertJson($jsonString);
$this->assertJsonStringEqualsJsonString($a, $b);

$this->assertGreaterThan(0, $n);
$this->assertGreaterThanOrEqual(0, $n);
$this->assertLessThan(100, $n);

$this->assertFileExists($path);
$this->assertFileEquals($expected, $actual);
$this->assertDirectoryExists($path);
```

### Data Providers (PHP 8 attributes only)

```php
use PHPUnit\Framework\Attributes\DataProvider;

final class CalculatorTest extends TestCase
{
    #[Test]
    #[DataProvider('additionCases')]
    public function it_adds(int $a, int $b, int $expected): void
    {
        $this->assertSame($expected, (new Calculator())->add($a, $b));
    }

    public static function additionCases(): iterable
    {
        yield 'positive' => [1, 2, 3];
        yield 'negative' => [-5, -3, -8];
        yield 'mixed' => [-1, 1, 0];
    }
}
```

External provider:

```php
use PHPUnit\Framework\Attributes\DataProviderExternal;

#[DataProviderExternal(CalculatorDataProviders::class, 'additionCases')]
public function it_adds(int $a, int $b, int $expected): void { /* ... */ }
```

### Exceptions

```php
public function test_it_throws(): void
{
    $this->expectException(InvalidArgumentException::class);
    $this->expectExceptionMessage('value must be positive');
    $this->expectExceptionCode(42);
    $this->expectExceptionMessageMatches('/positive/');

    new Money(-1, 'EUR');
}
```

### Fixtures (Setup/Teardown)

```php
use PHPUnit\Framework\Attributes\Before;
use PHPUnit\Framework\Attributes\After;
use PHPUnit\Framework\Attributes\BeforeClass;
use PHPUnit\Framework\Attributes\AfterClass;

final class MyTest extends TestCase
{
    private Calculator $calc;

    protected function setUp(): void { $this->calc = new Calculator(); }
    protected function tearDown(): void { /* ... */ }

    #[Before]
    public function clearCache(): void { /* ... */ }

    #[BeforeClass]
    public static function bootSchema(): void { /* ... */ }
}
```

### Mocks (Test Doubles)

```php
$repo = $this->createMock(UserRepository::class);
$repo->method('find')->willReturn(new User());
$repo->method('count')->willReturnCallback(fn () => rand(1, 10));
$repo->method('save')->willReturnArgument(0);
$repo->method('delete')->willThrowException(new \RuntimeException());

$repo = $this->createStub(UserRepository::class);    // No assertions on calls
```

Stricter:

```php
$repo->expects($this->once())->method('save');
$repo->expects($this->never())->method('delete');
$repo->expects($this->atLeast(2))->method('find');
$repo->expects($this->exactly(3))->method('publish');

// Replace withConsecutive (removed):
$matcher = $this->exactly(2);
$mock->expects($matcher)
    ->method('process')
    ->willReturnCallback(function (string $arg) use ($matcher): string {
        match ($matcher->numberOfInvocations()) {
            1 => $this->assertSame('first', $arg),
            2 => $this->assertSame('second', $arg),
        };
        return strtoupper($arg);
    });
```

Partial mocks:

```php
$mock = $this->createPartialMock(MyClass::class, ['onlyThisMethod']);
$mock->method('onlyThisMethod')->willReturn('mocked');
```

Mock for intersection types (12 supports):

```php
$mock = $this->createMockForIntersectionOfInterfaces([Countable::class, ArrayAccess::class]);
```

### Coverage Attributes

```php
use PHPUnit\Framework\Attributes\CoversClass;
use PHPUnit\Framework\Attributes\CoversMethod;
use PHPUnit\Framework\Attributes\UsesClass;

#[CoversClass(Calculator::class)]
#[UsesClass(Money::class)]
final class CalculatorTest extends TestCase { /* ... */ }
```

`requireCoverageMetadata="true"` in `phpunit.xml` makes missing `#[CoversClass]` a risky test.

### Groups, Test-doxing, Tickets

```php
use PHPUnit\Framework\Attributes\Group;
use PHPUnit\Framework\Attributes\TestDox;
use PHPUnit\Framework\Attributes\Ticket;

#[Group('slow')]
#[Group('db')]
#[TestDox('rejects invalid email format')]
#[Ticket('JIRA-1234')]
public function test_email_validation(): void { /* ... */ }
```

### Dependencies

```php
use PHPUnit\Framework\Attributes\Depends;

#[Test]
public function creates_user(): User { return new User(); }

#[Test]
#[Depends('creates_user')]
public function persists_user(User $user): void { /* ... */ }
```

### Conditional Skipping

```php
use PHPUnit\Framework\Attributes\RequiresPhp;
use PHPUnit\Framework\Attributes\RequiresPhpunit;
use PHPUnit\Framework\Attributes\RequiresOperatingSystem;
use PHPUnit\Framework\Attributes\RequiresFunction;
use PHPUnit\Framework\Attributes\RequiresPhpExtension;

#[RequiresPhp('>= 8.4')]
#[RequiresOperatingSystem('Linux')]
#[RequiresFunction('json_validate')]
#[RequiresPhpExtension('redis')]
public function test_thing(): void { /* ... */ }
```

### Process Isolation

```php
use PHPUnit\Framework\Attributes\RunInSeparateProcess;
use PHPUnit\Framework\Attributes\RunTestsInSeparateProcesses;
use PHPUnit\Framework\Attributes\PreserveGlobalState;

#[RunInSeparateProcess]
#[PreserveGlobalState(false)]
public function test_with_globals(): void { /* ... */ }

#[RunTestsInSeparateProcesses]
final class IsolatedSuite extends TestCase { /* ... */ }
```

## CLI Cheatsheet

```bash
vendor/bin/phpunit                              # Run all
vendor/bin/phpunit --testsuite=Unit
vendor/bin/phpunit --filter=test_name
vendor/bin/phpunit --group=slow
vendor/bin/phpunit --exclude-group=slow
vendor/bin/phpunit tests/Unit/MyTest.php
vendor/bin/phpunit tests/Unit/MyTest.php::test_method

# Coverage
vendor/bin/phpunit --coverage-text
vendor/bin/phpunit --coverage-html=var/coverage
vendor/bin/phpunit --coverage-clover=clover.xml
vendor/bin/phpunit --coverage-cobertura=cobertura.xml

# Strict
vendor/bin/phpunit --strict-coverage --strict-global-state
vendor/bin/phpunit --fail-on-deprecation --fail-on-warning --fail-on-notice

# Output
vendor/bin/phpunit --testdox
vendor/bin/phpunit --debug
vendor/bin/phpunit --stop-on-failure
vendor/bin/phpunit --stop-on-defect
```

## Full Documentation

For full assertion catalog, test doubles in depth, baselines, custom Printer/Logger, separate-process internals, and Symfony test integration: see [references/phpunit.md](references/phpunit.md).
