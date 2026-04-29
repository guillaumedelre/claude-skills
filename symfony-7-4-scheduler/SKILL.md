---
name: "symfony-7-4-scheduler"
description: "Symfony 7.4 Scheduler component reference for scheduling recurring tasks via Messenger. Use when defining cron-like schedules, periodic tasks, custom triggers, or running messages on a recurring basis. Triggers on: Scheduler, ScheduleProviderInterface, RecurringMessage, AsSchedule, AsCronTask, AsPeriodicTask, Schedule, MessageGenerator, TriggerInterface, CronExpressionTrigger, PeriodicalTrigger, JitterTrigger, scheduler:debug, scheduler transport, messenger scheduler, RedispatchMessage, ScheduledStamp, every, lock, cache, before/after schedule events."
---

# Symfony 7.4 Scheduler Component

GitHub: https://github.com/symfony/scheduler
Docs: https://symfony.com/doc/7.4/scheduler.html

## Installation

```bash
composer require symfony/scheduler
```

The Scheduler relies on the Messenger component for dispatching messages.

## Quick Reference

### Defining a Schedule

```php
namespace App\Scheduler;

use App\Message\SendDailyReportMessage;
use Symfony\Component\Scheduler\Attribute\AsSchedule;
use Symfony\Component\Scheduler\RecurringMessage;
use Symfony\Component\Scheduler\Schedule;
use Symfony\Component\Scheduler\ScheduleProviderInterface;
use Symfony\Contracts\Cache\CacheInterface;

#[AsSchedule('default')]
class MainSchedule implements ScheduleProviderInterface
{
    public function __construct(private CacheInterface $cache) {}

    public function getSchedule(): Schedule
    {
        return (new Schedule())
            ->with(
                RecurringMessage::every('5 minutes', new SendDailyReportMessage()),
                RecurringMessage::cron('0 8 * * *', new SendDailyReportMessage()),
            )
            ->stateful($this->cache)
            ->lock($this->lockFactory->createLock('scheduler-main'));
    }
}
```

### Consuming the Schedule

```bash
# Run the scheduler transport (configured via name in #[AsSchedule])
php bin/console messenger:consume scheduler_default
```

### Configuring the Transport

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            scheduler_default: 'schedule://default'
        routing:
            'App\Message\SendDailyReportMessage': scheduler_default
```

The transport name pattern is `scheduler_<name>` where `<name>` matches the `#[AsSchedule]` argument.

### Recurring Message Triggers

```php
use Symfony\Component\Scheduler\RecurringMessage;

// Cron expression
RecurringMessage::cron('*/5 * * * *', new MyMessage());

// Cron with timezone
RecurringMessage::cron('0 8 * * *', new MyMessage(), new \DateTimeZone('Europe/Paris'));

// Every interval
RecurringMessage::every('10 seconds', new MyMessage());
RecurringMessage::every('1 day', new MyMessage(), from: '08:00:00');
RecurringMessage::every('30 minutes', new MyMessage(), until: '2026-12-31');

// Custom trigger
RecurringMessage::trigger(new CustomTrigger(), new MyMessage());
```

### #[AsCronTask] / #[AsPeriodicTask] Attributes (PHP 8 attributes)

Apply directly to a service class (the service itself becomes the message handler):

```php
use Symfony\Component\Scheduler\Attribute\AsCronTask;
use Symfony\Component\Scheduler\Attribute\AsPeriodicTask;

#[AsCronTask('0 * * * *', schedule: 'default')]
final class HourlyCleanupTask
{
    public function __invoke(): void
    {
        // Executed every hour
    }
}

#[AsPeriodicTask(frequency: 60, schedule: 'default', method: 'run')]
final class HeartbeatTask
{
    public function run(): void
    {
        // Executed every 60 seconds
    }
}
```

Useful options: `schedule`, `method`, `from`, `until`, `timezone`, `arguments`, `jitter`.

### Stateful + Locked Schedules (HA)

```php
return (new Schedule())
    ->with(/* messages */)
    ->stateful($this->cache)         // Survives restarts (uses CacheInterface)
    ->lock($this->lockFactory->createLock('scheduler-default'));
```

`stateful()` persists last execution to avoid re-running missed tasks. `lock()` ensures only one worker runs at a time when consumed concurrently.

### Custom Trigger

```php
use Symfony\Component\Scheduler\Trigger\TriggerInterface;

final class BusinessHoursTrigger implements TriggerInterface
{
    public function __construct(private TriggerInterface $inner) {}

    public function __toString(): string
    {
        return 'business-hours';
    }

    public function getNextRunDate(\DateTimeImmutable $run): ?\DateTimeImmutable
    {
        $next = $this->inner->getNextRunDate($run);
        while ($next && ($next->format('N') > 5 || $next->format('H') < 9 || $next->format('H') >= 18)) {
            $next = $this->inner->getNextRunDate($next);
        }
        return $next;
    }
}
```

### Built-in Trigger Decorators

```php
use Symfony\Component\Scheduler\Trigger\JitterTrigger;
use Symfony\Component\Scheduler\Trigger\ExcludeTimeTrigger;

// Add random delay (0..30s) to spread load
new JitterTrigger($trigger, maxSeconds: 30);

// Skip execution during a window
new ExcludeTimeTrigger($trigger, '02:00', '04:00');
```

### Multiple Schedules

```php
#[AsSchedule('reports')]
class ReportsSchedule implements ScheduleProviderInterface { /* ... */ }

#[AsSchedule('cleanup')]
class CleanupSchedule implements ScheduleProviderInterface { /* ... */ }
```

Run each in its own worker:

```bash
php bin/console messenger:consume scheduler_reports
php bin/console messenger:consume scheduler_cleanup
```

### Debug Schedules

```bash
php bin/console debug:scheduler
php bin/console debug:scheduler default
```

### Schedule Events

```php
use Symfony\Component\Scheduler\Event\PreRunEvent;
use Symfony\Component\Scheduler\Event\PostRunEvent;
use Symfony\Component\Scheduler\Event\FailureEvent;

#[AsEventListener]
final class SchedulerListener
{
    public function onPreRun(PreRunEvent $event): void
    {
        // Skip with $event->shouldCancel(true)
    }
}
```

### Redispatching Messages

Messages produced by the scheduler dispatch to the same `scheduler_*` transport by default. Use `RedispatchMessage` to forward to another transport:

```php
use Symfony\Component\Messenger\Message\RedispatchMessage;

RecurringMessage::every('1 hour', new RedispatchMessage(new MyMessage(), 'async'));
```

## Full Documentation

Detailed reference (custom triggers, full attribute options, transport internals, advanced patterns): see [references/scheduler.md](references/scheduler.md).
