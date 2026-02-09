---
name: "symfony-7-4-clock"
description: "Symfony 7.4 Clock component reference for decoupling applications from the system clock and testing time-sensitive logic. Use when working with Clock, ClockInterface, MockClock, NativeClock, MonotonicClock, DatePoint, ClockAwareTrait, ClockSensitiveTrait, time testing, now() helper, or Doctrine date_point/day_point/time_point types. Triggers on: Clock, ClockInterface, MockClock, NativeClock, DatePoint, time testing, ClockAwareTrait, ClockSensitiveTrait, now()."
---

# Symfony 7.4 Clock Component

GitHub: https://github.com/symfony/clock
Docs: https://symfony.com/doc/7.4/components/clock.html

## Quick Reference

### Installation

```bash
composer require symfony/clock
```

### Getting Current Time

```php
use Symfony\Component\Clock\Clock;
use function Symfony\Component\Clock\now;

// Via global clock
$now = Clock::get()->now();

// Via helper function
$now = now();
$later = now('+3 hours');
```

### Using in Services (ClockAwareTrait)

```php
use Symfony\Component\Clock\ClockAwareTrait;

class MyService
{
    use ClockAwareTrait;

    public function isExpired(\DateTimeInterface $until): bool
    {
        return $this->now() > $until;
    }
}
```

### Constructor Injection

```php
use Symfony\Component\Clock\ClockInterface;

class ExpirationChecker
{
    public function __construct(private ClockInterface $clock) {}

    public function isExpired(\DateTimeInterface $until): bool
    {
        return $this->clock->now() > $until;
    }
}
```

### Testing with MockClock

```php
use Symfony\Component\Clock\MockClock;

$clock = new MockClock('2024-01-15 10:00:00');
$clock->sleep(600);        // Advance 600 seconds instantly
$clock->modify('+1 hour'); // Advance by modifier
```

### Testing with ClockSensitiveTrait

```php
use Symfony\Component\Clock\Test\ClockSensitiveTrait;

class MyTest extends TestCase
{
    use ClockSensitiveTrait;

    public function testTime(): void
    {
        $clock = static::mockTime('2024-01-15 10:00:00');
        $service = new MyService();
        $service->setClock($clock);
        // assertions...
    }
}
```

### DatePoint

```php
use Symfony\Component\Clock\DatePoint;

$dp = new DatePoint();                          // now
$dp = new DatePoint('+1 month');                // relative
$dp = DatePoint::createFromTimestamp(1700000000); // from timestamp
```

### Doctrine Types (Symfony 7.3+)

| Type | Extends | Since |
|------|---------|-------|
| `date_point` | `datetime_immutable` | 7.3 |
| `day_point` | `date_immutable` | 7.4 |
| `time_point` | `time_immutable` | 7.4 |

## Full Documentation

For complete details including all clock implementations (NativeClock, MonotonicClock), DatePoint methods, Doctrine entity examples, exception handling, and advanced testing patterns, see [references/clock.md](references/clock.md).
