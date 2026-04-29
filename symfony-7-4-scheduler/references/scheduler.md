# Symfony 7.4 Scheduler - Complete Reference

## Table of Contents

- [Installation](#installation)
- [Architecture](#architecture)
- [Schedule Provider](#schedule-provider)
- [Recurring Messages](#recurring-messages)
- [Triggers](#triggers)
- [Attributes](#attributes)
- [Transport](#transport)
- [Stateful Scheduler](#stateful-scheduler)
- [Locking](#locking)
- [Events](#events)
- [Multiple Schedules](#multiple-schedules)
- [Debugging](#debugging)
- [Redispatching](#redispatching)
- [Error Handling](#error-handling)

---

## Installation

```bash
composer require symfony/scheduler
```

Requires `symfony/messenger`. Auto-configured by Symfony Flex.

## Architecture

The Scheduler is built on top of Messenger:

1. A `Schedule` declares one or more `RecurringMessage` items with triggers.
2. A virtual transport (`schedule://<name>`) acts as a Messenger source.
3. `messenger:consume scheduler_<name>` reads from this transport.
4. When a trigger's `getNextRunDate()` matches the clock, the message is dispatched onto the bus and routed to its handler.

Key interfaces and classes:
- `Symfony\Component\Scheduler\ScheduleProviderInterface`
- `Symfony\Component\Scheduler\Schedule`
- `Symfony\Component\Scheduler\RecurringMessage`
- `Symfony\Component\Scheduler\Trigger\TriggerInterface`
- `Symfony\Component\Scheduler\Generator\MessageGenerator`
- `Symfony\Component\Scheduler\Messenger\SchedulerTransport`

## Schedule Provider

```php
use Symfony\Component\Scheduler\Attribute\AsSchedule;
use Symfony\Component\Scheduler\Schedule;
use Symfony\Component\Scheduler\ScheduleProviderInterface;

#[AsSchedule('default')]
final class DefaultSchedule implements ScheduleProviderInterface
{
    public function getSchedule(): Schedule
    {
        return (new Schedule())
            ->with(
                RecurringMessage::cron('@daily', new DailyReportMessage()),
            );
    }
}
```

`#[AsSchedule]` accepts a single `name` argument. Default is `default`.

A schedule provider can produce its `Schedule` lazily; `getSchedule()` is cached after first call within a single worker.

## Recurring Messages

`RecurringMessage` static factories:

| Factory | Purpose |
|---------|---------|
| `cron(string $expression, object $message, ?\DateTimeZone $tz = null)` | Cron-driven |
| `every(string|int $frequency, object $message, ?string $from = null, ?string $until = null)` | Interval-driven |
| `trigger(TriggerInterface $trigger, object $message)` | Custom trigger |

Methods:

```php
$msg = RecurringMessage::cron('0 8 * * *', new MyMessage());
$msg->getMessages();       // iterable<object>
$msg->getTrigger();        // TriggerInterface
$msg->getId();             // string
$msg->getMessage();        // object
```

### Cron Expressions

Standard 5-field cron (`minute hour day month weekday`) plus extensions:

| Macro | Equivalent |
|-------|------------|
| `@yearly` / `@annually` | `0 0 1 1 *` |
| `@monthly` | `0 0 1 * *` |
| `@weekly` | `0 0 * * 0` |
| `@daily` / `@midnight` | `0 0 * * *` |
| `@hourly` | `0 * * * *` |

Hashed expressions for jitter (uses message hash for stable randomization):

```
H 8 * * *      # Some minute past 8 AM
H/10 * * * *   # Every 10 minutes, offset deterministically
```

### Every

```php
RecurringMessage::every('30 seconds', $msg);
RecurringMessage::every('15 minutes', $msg);
RecurringMessage::every('2 hours', $msg);
RecurringMessage::every('1 day', $msg, from: '08:00:00');
```

`from` and `until` accept any `strtotime()`-compatible string. They scope when the recurrence is active.

## Triggers

### Built-in

| Class | Notes |
|-------|-------|
| `CronExpressionTrigger` | Wraps `dragonmantank/cron-expression`; supports macros, hashed expressions |
| `PeriodicalTrigger` | Fixed interval |
| `JitterTrigger` | Decorator: adds random delay (0..N seconds) per fire |
| `ExcludeTimeTrigger` | Decorator: skips a daily window |
| `AlwaysTrigger` | Fires immediately and every tick (test/debug) |
| `CallbackTrigger` | Backed by a closure returning next run |

### TriggerInterface

```php
interface TriggerInterface
{
    public function __toString(): string;
    public function getNextRunDate(\DateTimeImmutable $run): ?\DateTimeImmutable;
}
```

Returning `null` from `getNextRunDate()` halts the trigger permanently.

### Composing Triggers

```php
$base = CronExpressionTrigger::fromSpec('0 * * * *');
$jittered = new JitterTrigger($base, maxSeconds: 60);
$message = RecurringMessage::trigger($jittered, new HourlySync());
```

## Attributes

### #[AsCronTask]

Decorate a service. The service is invoked (`__invoke` by default, or a chosen method) on the cron schedule. Internally a `RecurringMessage` is generated whose message handler resolves to the service.

```php
use Symfony\Component\Scheduler\Attribute\AsCronTask;

#[AsCronTask(
    expression: '0 3 * * *',
    schedule: 'default',
    timezone: 'Europe/Paris',
    method: 'run',
    arguments: ['mode' => 'full'],
    jitter: 30,
)]
final class NightlyCleanup
{
    public function run(string $mode): void { /* ... */ }
}
```

### #[AsPeriodicTask]

```php
use Symfony\Component\Scheduler\Attribute\AsPeriodicTask;

#[AsPeriodicTask(frequency: '5 minutes', schedule: 'default', from: '08:00', until: '20:00')]
final class HealthCheck
{
    public function __invoke(): void { /* ... */ }
}
```

`frequency` accepts seconds (int) or any string accepted by `every()`.

## Transport

Schedules expose themselves as Messenger transports:

```yaml
framework:
    messenger:
        transports:
            scheduler_default: 'schedule://default'
```

The DSN segment after `schedule://` matches the `#[AsSchedule]` name.

Routing the produced messages explicitly (so dispatched messages also enter another queue):

```yaml
framework:
    messenger:
        routing:
            'App\Message\SendDailyReport': [scheduler_default, async]
```

In practice, route the message itself to a real queue (e.g. `async`) and the scheduler dispatches it. To make the scheduler push to the real queue, wrap with `RedispatchMessage`.

## Stateful Scheduler

Without state, restarting the worker may cause missed runs to be skipped or duplicated. `stateful()` persists the last fired timestamp:

```php
use Psr\Cache\CacheItemPoolInterface;

return (new Schedule())
    ->stateful($this->cachePool)
    ->with(/* ... */);
```

Pass any PSR-6 cache. On worker start, the schedule replays missed runs since last persisted timestamp (within a sane window).

## Locking

```php
use Symfony\Component\Lock\LockFactory;

return (new Schedule())
    ->lock($lockFactory->createLock('scheduler.default'))
    ->with(/* ... */);
```

Required when multiple workers consume the same schedule transport. Without a lock, every worker will fire each tick.

## Events

| Event | When |
|-------|------|
| `PreRunEvent` | Before each scheduled message dispatch. Call `$event->shouldCancel(true)` to skip |
| `PostRunEvent` | After successful dispatch |
| `FailureEvent` | On exception during dispatch. Call `$event->shouldIgnore(true)` to swallow |

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Scheduler\Event\PreRunEvent;

#[AsEventListener]
final class MaintenanceWindowListener
{
    public function __invoke(PreRunEvent $event): void
    {
        if ($this->isMaintenanceWindow()) {
            $event->shouldCancel(true);
        }
    }
}
```

## Multiple Schedules

Define each in its own provider with distinct `name`. Run a worker per schedule:

```bash
php bin/console messenger:consume scheduler_reports --time-limit=3600
php bin/console messenger:consume scheduler_cleanup --time-limit=3600
```

Use `--memory-limit`, `--time-limit`, `--limit` to recycle workers; pair with systemd/supervisord for restart.

## Debugging

```bash
php bin/console debug:scheduler              # All schedules
php bin/console debug:scheduler default      # Specific schedule with next 5 runs
php bin/console debug:scheduler default --date='2026-04-29 10:00:00'
```

Output shows trigger description, message FQCN, next N run dates.

## Redispatching

When a scheduled message is heavy, do not run it in the scheduler worker. Wrap it:

```php
use Symfony\Component\Messenger\Message\RedispatchMessage;

RecurringMessage::cron('0 4 * * *', new RedispatchMessage(
    new HeavyJob(),
    transports: 'async_high',
));
```

The scheduler emits a tiny envelope; the actual handler runs on `async_high`.

## Error Handling

If a handler throws while scheduler-driven:

1. `FailureEvent` fires; listeners can `shouldIgnore(true)` to silence.
2. If unhandled, Messenger applies the transport's retry policy (or sends to failure transport).
3. The schedule itself is not affected; the next tick still fires on schedule.

For best practice, route scheduled messages through `RedispatchMessage` to a transport with explicit retry strategy and failure transport:

```yaml
framework:
    messenger:
        failure_transport: failed
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 3
                    delay: 1000
                    multiplier: 2
            failed: 'doctrine://default?queue_name=failed'
```

## CLI Reference

| Command | Purpose |
|---------|---------|
| `debug:scheduler [name]` | List schedules and upcoming runs |
| `messenger:consume scheduler_<name>` | Run scheduler worker |
| `messenger:stats` | Inspect transport (including scheduler) |
