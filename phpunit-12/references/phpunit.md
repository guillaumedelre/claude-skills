# PHPUnit 12 - Complete Reference

## Table of Contents

- [Architecture and Lifecycle](#architecture-and-lifecycle)
- [Attributes Catalog](#attributes-catalog)
- [Assertions Catalog](#assertions-catalog)
- [Test Doubles](#test-doubles)
- [Data Providers](#data-providers)
- [Test Dependencies](#test-dependencies)
- [Process Isolation](#process-isolation)
- [Backup and Global State](#backup-and-global-state)
- [Coverage](#coverage)
- [Risky Tests](#risky-tests)
- [Baselines](#baselines)
- [Configuration Schema](#configuration-schema)
- [Custom Printers and Loggers](#custom-printers-and-loggers)
- [Symfony Integration](#symfony-integration)
- [Migration from PHPUnit 11](#migration-from-phpunit-11)

---

## Architecture and Lifecycle

A `TestCase` instance is created **per test method**. Lifecycle:

1. `setUpBeforeClass()` (once) / `#[BeforeClass]` methods.
2. `setUp()` (each test) / `#[Before]` methods.
3. `setUpRequiredMethods` (handled by data provider).
4. Test method.
5. `tearDown()` / `#[After]` methods.
6. `tearDownAfterClass()` (once) / `#[AfterClass]` methods.

`onNotSuccessfulTest(Throwable $t)` allows custom handling (rethrow to fail).

## Attributes Catalog

### Test Identification

| Attribute | Effect |
|-----------|--------|
| `#[Test]` | Mark method as a test (alternative to `test*` naming) |
| `#[TestDox(string)]` | Custom documentation in `--testdox` output |
| `#[Ticket(string)]` | Reference to issue tracker |
| `#[Group(string)]` | Tag for `--group`/`--exclude-group` filtering |
| `#[Small]`/`#[Medium]`/`#[Large]` | Predefined groups for size classes |

### Data Providers

| Attribute | Effect |
|-----------|--------|
| `#[DataProvider(string)]` | Method on same class returning `iterable` |
| `#[DataProviderExternal(class, method)]` | Method on different class |
| `#[TestWith([...])]` | Inline single dataset |
| `#[TestWithJson(string)]` | Inline JSON dataset |

### Dependencies

| Attribute | Effect |
|-----------|--------|
| `#[Depends(method)]` | Pass return value of named method |
| `#[DependsUsingDeepClone(method)]` | Deep-cloned dependency value |
| `#[DependsUsingShallowClone(method)]` | Shallow-cloned |
| `#[DependsExternal(class, method)]` | Dependency on test in other class |
| `#[DependsOnClass(class)]` | Class-level dependency (suite ordering) |

### Conditional Skipping

| Attribute | Effect |
|-----------|--------|
| `#[RequiresPhp('>= 8.3')]` | PHP version |
| `#[RequiresPhpunit('>= 12.0')]` | PHPUnit version |
| `#[RequiresOperatingSystem('Linux')]` | OS regex |
| `#[RequiresOperatingSystemFamily('Linux')]` | Family (Linux/Windows/Darwin/BSD) |
| `#[RequiresFunction('json_validate')]` | Function exists |
| `#[RequiresMethod(class, method)]` | Class method exists |
| `#[RequiresSetting('ini.key', 'value')]` | php.ini value |
| `#[RequiresPhpExtension('redis', '5.0')]` | Extension version |
| `#[RequiresEnvironmentVariable('CI', '1')]` | Env var |

### Coverage

| Attribute | Effect |
|-----------|--------|
| `#[CoversClass(class)]` | Class under test |
| `#[CoversMethod(class, method)]` | Specific method |
| `#[CoversFunction(name)]` | Plain function |
| `#[CoversTrait(class)]` | Trait |
| `#[CoversNothing]` | Explicit zero-coverage tests |
| `#[UsesClass(class)]` | Used but not tested (not credited) |
| `#[UsesMethod(class, method)]` | Same |
| `#[UsesFunction(name)]` | Same |
| `#[UsesTrait(class)]` | Same |

### Process Control

| Attribute | Effect |
|-----------|--------|
| `#[RunInSeparateProcess]` | Run this test in subprocess |
| `#[RunTestsInSeparateProcesses]` | Apply to whole class |
| `#[PreserveGlobalState(bool)]` | Pass parent globals to subprocess |
| `#[BackupGlobals(bool)]` | Snapshot/restore $GLOBALS |
| `#[BackupStaticProperties(bool)]` | Snapshot/restore static props |
| `#[ExcludeGlobalVariableFromBackup(name)]` | Exempt named global |
| `#[ExcludeStaticPropertyFromBackup(class, name)]` | Exempt static prop |
| `#[DoesNotPerformAssertions]` | Skip "no assertions" warning |
| `#[WithoutErrorHandler]` | Disable PHPUnit's error handler |

### Hooks

| Attribute | When |
|-----------|------|
| `#[Before]` | Before each test |
| `#[After]` | After each test |
| `#[BeforeClass]` | Once before class |
| `#[AfterClass]` | Once after class |
| `#[PreCondition]` | After `setUp` |
| `#[PostCondition]` | Before `tearDown` |

## Assertions Catalog

Equality:
```
assertSame, assertNotSame
assertEquals, assertNotEquals
assertEqualsCanonicalizing, assertEqualsIgnoringCase, assertEqualsWithDelta
```

Boolean:
```
assertTrue, assertFalse, assertNull, assertNotNull
```

Type:
```
assertIsArray, assertIsBool, assertIsCallable, assertIsFloat, assertIsInt,
assertIsIterable, assertIsNumeric, assertIsObject, assertIsResource,
assertIsString, assertIsScalar, assertIsClosedResource
assertInstanceOf, assertNotInstanceOf
```

Collections:
```
assertCount, assertNotCount, assertSameSize, assertNotSameSize
assertContains, assertNotContains, assertContainsEquals
assertContainsOnly, assertContainsOnlyInstancesOf, assertContainsNotOnly
assertEmpty, assertNotEmpty
```

Strings:
```
assertStringContainsString[IgnoringCase]
assertStringNotContainsString[IgnoringCase]
assertStringStartsWith, assertStringEndsWith
assertStringMatchesFormat, assertStringMatchesFormatFile
assertMatchesRegularExpression, assertDoesNotMatchRegularExpression
assertJson, assertJsonStringEqualsJsonString, assertJsonStringEqualsJsonFile
assertXmlStringEqualsXmlString, assertXmlStringEqualsXmlFile
```

Numerics:
```
assertGreaterThan, assertGreaterThanOrEqual
assertLessThan, assertLessThanOrEqual
assertNan, assertInfinite, assertFinite
```

Objects/Properties:
```
assertObjectHasProperty, assertObjectNotHasProperty
assertObjectEquals(expected, actual, method='equals')
```

Arrays:
```
assertArrayHasKey, assertArrayNotHasKey, assertArrayIsList
assertArraySubset (deprecated; use custom)
```

Files/Directories:
```
assertFileExists, assertFileDoesNotExist, assertFileEquals, assertFileEqualsCanonicalizing
assertFileIsReadable, assertFileIsWritable, assertFileIsNotReadable
assertDirectoryExists, assertDirectoryDoesNotExist
assertDirectoryIsReadable, assertDirectoryIsWritable
```

Exceptions:
```
expectException, expectExceptionMessage, expectExceptionCode
expectExceptionMessageMatches, expectExceptionObject
expectNotToPerformAssertions
expectError, expectWarning, expectNotice, expectDeprecation
expectErrorMessage, expectWarningMessage, expectNoticeMessage, expectDeprecationMessage
expectErrorMessageMatches, ... etc
expectUserDeprecationMessage, expectUserDeprecationMessageMatches
```

## Test Doubles

### Mock vs Stub

- **Stub**: replaces a method's return value. No call expectations. Use `createStub()` when you only care about the return.
- **Mock**: stub + can verify how methods are called. Use `createMock()` and `expects()`.

### Creating Doubles

```php
// Pure stub - no implicit assertions
$stub = $this->createStub(Repository::class);
$stub->method('find')->willReturn($entity);

// Mock with verification
$mock = $this->createMock(Repository::class);
$mock->expects($this->once())->method('save')->with($entity);

// Partial mock - only listed methods are mocked, rest pass through to real impl
$partial = $this->createPartialMock(Service::class, ['heavyMethod']);

// Configured mock (chainable)
$mock = $this->createConfiguredMock(Repository::class, [
    'find' => $entity,
    'count' => 5,
]);

// Mock for multiple interfaces (intersection)
$mock = $this->createMockForIntersectionOfInterfaces([
    Countable::class,
    ArrayAccess::class,
]);
```

### Return Strategies

```php
->willReturn($value)
->willReturnArgument(int $index)
->willReturnSelf()
->willReturnReference($reference)
->willReturnMap([
    [$arg1, $arg2, $returnValue1],
    [$argA, $argB, $returnValueA],
])
->willReturnCallback(fn (...$args) => /* ... */)
->willThrowException(new \RuntimeException())
```

### Constraint Matchers

```php
->with(
    $this->equalTo('foo'),
    $this->isInstanceOf(User::class),
    $this->callback(fn ($x) => $x > 5),
)
```

### Replacing `withConsecutive` / `willReturnOnConsecutiveCalls`

These were removed in PHPUnit 12. Use `$matcher->numberOfInvocations()`:

```php
$matcher = $this->exactly(3);
$mock->expects($matcher)
    ->method('process')
    ->willReturnCallback(function (string $arg) use ($matcher): string {
        match ($matcher->numberOfInvocations()) {
            1 => $this->assertSame('a', $arg),
            2 => $this->assertSame('b', $arg),
            3 => $this->assertSame('c', $arg),
        };
        return strtoupper($arg);
    });
```

### Test Stubs for Anonymous Classes

```php
$stub = new class implements Repository {
    public function find(int $id): ?User { return new User($id); }
};
```

Often clearer than `createStub()` for narrow interfaces.

## Data Providers

Static methods returning `iterable`:

```php
public static function emails(): iterable
{
    return [
        'simple'   => ['a@b.c', true],
        'invalid'  => ['not-email', false],
        'empty'    => ['', false],
    ];
}
```

Inline:

```php
#[TestWith(['simple', 'a@b.c', true])]
#[TestWith(['empty', '', false])]
public function test_email(string $name, string $email, bool $valid): void { /* ... */ }
```

JSON inline:

```php
#[TestWithJson('["a@b.c", true]')]
public function test_email(string $email, bool $valid): void { /* ... */ }
```

## Test Dependencies

```php
#[Test]
public function creates_user(): User { return new User('alice'); }

#[Test]
#[Depends('creates_user')]
public function uses_user(User $user): void { /* ... */ }

#[Test]
#[DependsUsingDeepClone('creates_user')]
public function uses_clone(User $user): void { /* mutate freely */ }
```

If a depended-on test fails or skips, dependent tests are skipped automatically.

## Process Isolation

Subprocesses serialize the test instance to disk; PHPUnit restores via `unserialize`. Caveats:
- Closures cannot be unserialized — store callable strings.
- `PreserveGlobalState(true)` (default) snapshots globals; expensive.
- Use isolation only when truly needed (e.g. testing global side effects).

```php
#[RunInSeparateProcess]
#[PreserveGlobalState(false)]
public function test_modifies_globals(): void
{
    $_SERVER['HTTP_HOST'] = 'localhost';
    // Globals reset on next test
}
```

## Backup and Global State

```php
#[BackupGlobals(true)]
#[BackupStaticProperties(true)]
#[ExcludeGlobalVariableFromBackup('GLOBALS')]
#[ExcludeStaticPropertyFromBackup(MyClass::class, 'cache')]
final class StatefulTest extends TestCase {}
```

In `phpunit.xml`:

```xml
<phpunit backupGlobals="true" backupStaticProperties="true" />
```

## Coverage

Drivers:
- **PCOV** (`pecl install pcov`) — fastest, line+branch.
- **Xdebug** — slower; supports path coverage with `xdebug.mode=coverage`.

In `phpunit.xml`:

```xml
<source restrictDeprecations="true" restrictNotices="true" restrictWarnings="true">
    <include>
        <directory>src</directory>
    </include>
</source>

<coverage pathCoverage="true" includeUncoveredFiles="true" ignoreDeprecatedCodeUnits="true">
    <report>
        <html outputDirectory="var/coverage" />
        <text outputFile="php://stdout" />
        <clover outputFile="var/clover.xml" />
        <cobertura outputFile="var/cobertura.xml" />
        <crap4j outputFile="var/crap.xml" />
        <xml outputDirectory="var/coverage-xml" />
    </report>
</coverage>
```

`requireCoverageMetadata="true"` makes tests without `#[CoversClass]` etc. risky. `beStrictAboutCoverageMetadata="true"` (synonym in 12).

## Risky Tests

`failOnRisky="true"` flags:
- Tests producing output (`beStrictAboutOutputDuringTests="true"`).
- Tests not asserting anything (`beStrictAboutTestsThatDoNotTestAnything="true"`).
- Tests changing global state without backup.
- Tests covering code without metadata.

Mark intentionally:
```php
$this->expectNotToPerformAssertions();
```

## Baselines

Generate a baseline of known deprecations/issues to ignore on subsequent runs:

```bash
vendor/bin/phpunit --generate-baseline=baseline.xml
vendor/bin/phpunit --use-baseline=baseline.xml
```

`<phpunit baseline="baseline.xml" />` in config.

Useful when migrating large code bases with deprecation noise; prevents new issues without forcing immediate cleanup.

## Configuration Schema

Top-level `<phpunit>` attributes (subset):

| Attribute | Default |
|-----------|---------|
| `bootstrap` | none |
| `cacheDirectory` | `.phpunit.cache` |
| `colors` | `false` |
| `processIsolation` | `false` |
| `failOnDeprecation` | `false` |
| `failOnNotice` | `false` |
| `failOnWarning` | `false` |
| `failOnRisky` | `false` |
| `failOnSkipped` | `false` |
| `failOnIncomplete` | `false` |
| `stopOnFailure` | `false` |
| `stopOnError` | `false` |
| `stopOnDefect` | `false` |
| `beStrictAboutOutputDuringTests` | `false` |
| `beStrictAboutTestsThatDoNotTestAnything` | `true` |
| `beStrictAboutCoverageMetadata` | `false` |
| `requireCoverageMetadata` | `false` |
| `displayDetailsOnIncompleteTests` | `false` |
| `displayDetailsOnSkippedTests` | `false` |
| `displayDetailsOnTestsThatTriggerDeprecations` | `false` |
| `executionOrder` | `default` |
| `resolveDependencies` | `true` |

Subelements: `<testsuites>`, `<source>`, `<coverage>`, `<groups>`, `<extensions>`, `<php>`, `<logging>`.

## Custom Printers and Loggers

Custom test runner extensions implement `PHPUnit\Runner\Extension\Extension`:

```php
final class MyExtension implements Extension
{
    public function bootstrap(Configuration $config, Facade $facade, ParameterCollection $params): void
    {
        $facade->registerSubscriber(new TestFinishedSubscriber());
    }
}
```

```xml
<extensions>
    <bootstrap class="App\Test\MyExtension">
        <parameter name="key" value="value" />
    </bootstrap>
</extensions>
```

## Symfony Integration

Use `symfony/phpunit-bridge` for clock mocking, deprecation reporting, parallelization helpers. PHPUnit 12 works with Symfony 7.x (`symfony/phpunit-bridge ^7.2`).

`KernelTestCase` (boots kernel, no HTTP):

```php
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

class ServiceTest extends KernelTestCase
{
    public function test(): void
    {
        $kernel = self::bootKernel();
        $service = self::getContainer()->get(MyService::class);
        $this->assertSame('result', $service->run());
    }
}
```

`WebTestCase` (boots client):

```php
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

class ApiTest extends WebTestCase
{
    public function test(): void
    {
        $client = static::createClient();
        $client->request('GET', '/api/health');
        self::assertResponseIsSuccessful();
    }
}
```

`DAMA\DoctrineTestBundle` wraps each test in a transaction (rolled back at end) for isolation.

## Migration from PHPUnit 11

Key removals/changes:

1. **Doc-comment annotations**: `@test`, `@dataProvider`, `@covers`, etc. — REMOVED. Convert to PHP 8 attributes.
2. `withConsecutive(...)` and `willReturnOnConsecutiveCalls(...)` — REMOVED. Use `$matcher->numberOfInvocations()`.
3. `at()` matcher — REMOVED. Use callback-based matchers.
4. `MockBuilder::getMockForAbstractClass()` — DEPRECATED. Use anonymous classes.
5. `expectError()` for E_*-level — DEPRECATED. Configure `failOnNotice/Warning/Deprecation` and use `expectUserDeprecationMessage` for `trigger_error`.
6. `setMethods()` / `setMethodsExcept()` — REMOVED. Use `onlyMethods()` / `addMethods()` (the latter also REMOVED in 12; only mock declared methods).
7. PHP 8.3 minimum.
8. Stricter type enforcement on data providers (must be `static` and return `iterable`).

Run rector recipe:

```bash
composer require --dev rector/rector
vendor/bin/rector process tests --config=vendor/rector/rector/config/set/phpunit/phpunit-100.php
# (apply all phpunit/phpunit-* config sets up to phpunit-120.php)
```
