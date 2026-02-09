---
name: "symfony-7-4-event-dispatcher"
description: "Symfony 7.4 EventDispatcher component reference for event-driven communication between application components. Use when creating, dispatching, or listening to events, implementing event subscribers, configuring listeners, or working with kernel events. Triggers on: EventDispatcher, events, listeners, subscribers, dispatch, #[AsEventListener], kernel events, event propagation, stopPropagation, EventSubscriberInterface, KernelEvents, kernel.request, kernel.response, kernel.exception, kernel.controller."
---

# Symfony 7.4 EventDispatcher Component

GitHub: https://github.com/symfony/event-dispatcher
Docs: https://symfony.com/doc/7.4/components/event_dispatcher.html

## Quick Reference

### Creating a Custom Event

```php
use Symfony\Contracts\EventDispatcher\Event;

final class OrderPlacedEvent extends Event
{
    public function __construct(private Order $order) {}

    public function getOrder(): Order
    {
        return $this->order;
    }
}
```

### Dispatching an Event

```php
$event = new OrderPlacedEvent($order);
$dispatcher->dispatch($event);
```

### Event Listener with #[AsEventListener] (Recommended)

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
final class OrderListener
{
    public function __invoke(OrderPlacedEvent $event): void
    {
        // handle event
    }
}
```

### Multiple Listeners on One Class

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

final class MyMultiListener
{
    #[AsEventListener]
    public function onOrderPlaced(OrderPlacedEvent $event): void { /* ... */ }

    #[AsEventListener(event: 'foo', priority: 42)]
    public function onFoo(): void { /* ... */ }
}
```

### Event Subscriber

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\KernelEvents;

class StoreSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::RESPONSE => [
                ['onKernelResponsePre', 10],
                ['onKernelResponsePost', -10],
            ],
            OrderPlacedEvent::class => 'onPlacedOrder',
        ];
    }

    // ... handler methods
}
```

### Stopping Event Propagation

```php
public function onPlacedOrder(OrderPlacedEvent $event): void
{
    $event->stopPropagation();
}
```

### Kernel Events

- `kernel.request` (KernelEvents::REQUEST)
- `kernel.controller` (KernelEvents::CONTROLLER)
- `kernel.response` (KernelEvents::RESPONSE)
- `kernel.exception` (KernelEvents::EXCEPTION)

### Debugging

```bash
php bin/console debug:event-dispatcher
php bin/console debug:event-dispatcher kernel.exception
```

## Full Documentation

For complete details including service container integration, event aliases, before/after filters, listener vs subscriber comparison, all kernel event types, priority system, and advanced patterns, see [references/event-dispatcher.md](references/event-dispatcher.md).
