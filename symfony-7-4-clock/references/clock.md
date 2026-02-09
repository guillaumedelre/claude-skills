# Symfony 7.4 Clock Component - Complete Reference

## Overview

The Clock component decouples applications from the system clock, improving testability of time-sensitive logic. It provides a `ClockInterface` (PSR-20 compatible) with multiple implementations for different use cases.

## Installation

```bash
composer require symfony/clock
```

## Core Classes

### ClockInterface

PSR-20 compatible interface implemented by all clock classes.

```php
namespace Symfony\Component\Clock;

interface ClockInterface extends \Psr\Clock\ClockInterface
{
    public function now(): DatePoint;
    public function sleep(float|int $seconds): void;
    public function withTimeZone(\DateTimeZone|string $timezone): static;
}
```

### Clock (Global Clock)

Static wrapper that delegates to an underlying clock implementation. Default is `NativeClock`.

```php
use Symfony\Component\Clock\Clock;

// Get the global clock instance
$clock = Clock::get();

// Set a custom implementation
Clock::set(new MockClock());

// Get current time
$now = Clock::get()->now();

// With timezone
$parisClock = Clock::get()->withTimeZone('Europe/Paris');

// Sleep
Clock::get()->sleep(2.5);
```

### NativeClock

Default implementation providing real system clock access. Equivalent to `new \DateTimeImmutable()`.

```php
use Symfony\Component\Clock\NativeClock;

$clock = new NativeClock();
$now = $clock->now(); // Current system time

// With specific timezone
$clock = new NativeClock('Europe/Paris');
```

### MockClock

Test-friendly implementation with fixed time that only advances via `sleep()` or `modify()`.

```php
use Symfony\Component\Clock\MockClock;

// Create with fixed time
$clock = new MockClock('2024-01-15 10:00:00');

// Get time (always returns the fixed time)
$now = $clock->now(); // 2024-01-15 10:00:00

// Advance time by sleeping (instant, no actual delay)
$clock->sleep(600); // Now: 2024-01-15 10:10:00

// Set specific time via modify
$clock->modify('2024-01-15 15:00:00');
$clock->modify('+1 hour');

// With timezone
$clock = new MockClock('2024-01-15 10:00:00', 'Europe/Paris');
```

### MonotonicClock

High-resolution monotonic clock using `hrtime()`. Useful for precise performance measurements where wall-clock accuracy is not needed but monotonic progression is required.

```php
use Symfony\Component\Clock\MonotonicClock;

$clock = new MonotonicClock();
$start = $clock->now();
// ... work ...
$end = $clock->now();
```

Note: `sleep()` on MonotonicClock performs an actual sleep (unlike MockClock).

### DatePoint

Immutable datetime wrapper extending `\DateTimeImmutable` with enhancements.

```php
use Symfony\Component\Clock\DatePoint;

// Current time (uses global clock)
$dp = new DatePoint();

// With specific timezone
$dp = new DatePoint(timezone: new \DateTimeZone('UTC'));

// Relative date
$dp = new DatePoint('+1 month');

// From a reference date
$ref = new \DateTimeImmutable('2024-01-01');
$dp = new DatePoint('+1 month', reference: $ref);

// From timestamp (Symfony 7.1+)
$dp = DatePoint::createFromTimestamp(1129645656);

// Microsecond methods (Symfony 7.1+)
$dp->setMicrosecond(345);
$micro = $dp->getMicrosecond();
```

## Helper Function

```php
use function Symfony\Component\Clock\now;

// Get current time as DatePoint
$now = now();

// With modifier
$later = now('+3 hours');
$yesterday = now('-1 day');
```

## Using Clock in Services

### ClockAwareTrait

Trait that provides `$this->now()` and automatic clock injection via service autoconfiguration.

```php
namespace App\Service;

use Symfony\Component\Clock\ClockAwareTrait;

class MonthSensitive
{
    use ClockAwareTrait;

    public function isWinterMonth(): bool
    {
        $now = $this->now();

        return match ($now->format('F')) {
            'December', 'January', 'February', 'March' => true,
            default => false,
        };
    }
}
```

When using Symfony's service container with autoconfiguration, the clock is automatically injected. The trait also exposes `setClock(ClockInterface $clock)` for manual injection in tests.

### Constructor Injection

```php
use Symfony\Component\Clock\ClockInterface;

class ExpirationChecker
{
    public function __construct(
        private ClockInterface $clock,
    ) {}

    public function isExpired(\DateTimeInterface $validUntil): bool
    {
        return $this->clock->now() > $validUntil;
    }
}
```

## Testing

### ClockSensitiveTrait

Provides `mockTime()` method to control time in PHPUnit tests.

```php
namespace App\Tests;

use App\Service\MonthSensitive;
use PHPUnit\Framework\TestCase;
use Symfony\Component\Clock\Test\ClockSensitiveTrait;

class MonthSensitiveTest extends TestCase
{
    use ClockSensitiveTrait;

    public function testIsWinterMonth(): void
    {
        $clock = static::mockTime(new \DateTimeImmutable('2024-03-02'));

        $service = new MonthSensitive();
        $service->setClock($clock);

        $this->assertTrue($service->isWinterMonth());
    }

    public function testIsNotWinterMonth(): void
    {
        $clock = static::mockTime(new \DateTimeImmutable('2024-06-02'));

        $service = new MonthSensitive();
        $service->setClock($clock);

        $this->assertFalse($service->isWinterMonth());
    }
}
```

### Direct MockClock in Tests

```php
use PHPUnit\Framework\TestCase;
use Symfony\Component\Clock\MockClock;

class ExpirationCheckerTest extends TestCase
{
    public function testIsExpired(): void
    {
        $clock = new MockClock('2024-01-15 10:00:00');
        $checker = new ExpirationChecker($clock);
        $validUntil = new \DateTimeImmutable('2024-01-15 10:30:00');

        // Not expired yet
        $this->assertFalse($checker->isExpired($validUntil));

        // Advance past expiration
        $clock->sleep(1800 + 1);
        $this->assertTrue($checker->isExpired($validUntil));

        // Reset to before expiration
        $clock->modify('2024-01-15 10:00:00');
        $this->assertFalse($checker->isExpired($validUntil));
    }
}
```

### Setting Global Clock in Tests

```php
use Symfony\Component\Clock\Clock;
use Symfony\Component\Clock\MockClock;

// In test setUp
$mock = new MockClock('2024-01-15 10:00:00');
Clock::set($mock);

// Code using Clock::get()->now() or now() will get the mocked time

// In test tearDown - restore
Clock::set(new NativeClock());
```

## Doctrine Integration (Symfony 7.3+)

### Supported Doctrine Types

| Type | Extends | Class | Since |
|------|---------|-------|-------|
| `date_point` | `datetime_immutable` | `DatePointType` | 7.3 |
| `day_point` | `date_immutable` | `DayPointType` | 7.4 |
| `time_point` | `time_immutable` | `TimePointType` | 7.4 |

### Entity Example

```php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Clock\DatePoint;

#[ORM\Entity]
class Product
{
    #[ORM\Column]
    private DatePoint $createdAt;

    #[ORM\Column(type: 'date_point')]
    private DatePoint $updatedAt;

    #[ORM\Column(type: 'day_point')]
    public DatePoint $birthday;

    #[ORM\Column(type: 'time_point')]
    public DatePoint $openAt;
}
```

## Exception Handling

The Clock component uses PHP 8.3+ DateTime exceptions (polyfilled by `symfony/polyfill-php83`):

```php
use Symfony\Component\Clock\Clock;
use Symfony\Component\Clock\MockClock;

try {
    $clock = Clock::get()->withTimeZone('invalid timezone');
} catch (\DateInvalidTimeZoneException $e) {
    // Handle invalid timezone
}

try {
    $clock = new MockClock('invalid date');
} catch (\DateMalformedStringException $e) {
    // Handle invalid date string
}
```

## Timezone Support

```php
use Symfony\Component\Clock\Clock;

// Get clock with specific timezone
$parisClock = Clock::get()->withTimeZone('Europe/Paris');
$utcClock = Clock::get()->withTimeZone('UTC');

// Each returns DatePoint in the specified timezone
$parisNow = $parisClock->now();
$utcNow = $utcClock->now();
```
