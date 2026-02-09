# Symfony 7.4 EventDispatcher - Complete Reference

## Table of Contents

- [Installation](#installation)
- [Core Concepts](#core-concepts)
- [Custom Events](#custom-events)
- [The Dispatcher](#the-dispatcher)
- [Event Listeners](#event-listeners)
- [AsEventListener Attribute](#aseventlistener-attribute)
- [Event Subscribers](#event-subscribers)
- [Event Propagation](#event-propagation)
- [Kernel Events](#kernel-events)
- [Service Container Integration](#service-container-integration)
- [Event Aliases](#event-aliases)
- [Listeners vs Subscribers](#listeners-vs-subscribers)
- [Debugging](#debugging)
- [API Reference](#api-reference)

---

## Installation

```bash
composer require symfony/event-dispatcher
```

## Core Concepts

The EventDispatcher component implements the Mediator and Observer design patterns. Components communicate by dispatching events and listening to them. An event is identified by a unique name (e.g., `kernel.response`) or by its class FQCN.

## Custom Events

### Creating a Custom Event

```php
namespace App\Event;

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

### Base Event Class

```php
use Symfony\Contracts\EventDispatcher\Event;

class Event
{
    public function stopPropagation(): void;
    public function isPropagationStopped(): bool;
}
```

### Generic Events (No Custom Class)

```php
namespace App\Event;

class StoreEvents
{
    public const ORDER_PLACED = 'order.placed';
}

// Dispatch with base Event class
$this->eventDispatcher->dispatch(new Event(), StoreEvents::ORDER_PLACED);
```

## The Dispatcher

```php
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();

// Dispatch by event class (recommended)
$event = new OrderPlacedEvent($order);
$dispatcher->dispatch($event);

// Dispatch with explicit event name
$dispatcher->dispatch($event, 'order.placed');
```

## Event Listeners

### Adding Listeners Programmatically

```php
$dispatcher->addListener('acme.foo.action', [$listener, 'onFooAction']);

// With priority (higher = earlier execution)
$dispatcher->addListener('acme.foo.action', [$listener, 'onFooAction'], 10);

// Closure listener
$dispatcher->addListener('acme.foo.action', function (Event $event): void {
    // handle event
});
```

### Listener Class

```php
use Symfony\Contracts\EventDispatcher\Event;

class AcmeListener
{
    public function onFooAction(Event $event): void
    {
        // handle event
    }
}
```

### Listener Receives Three Arguments

```php
public function myEventListener(
    Event $event,
    string $eventName,
    EventDispatcherInterface $dispatcher
): void {
    // ...
}
```

### YAML Configuration

```yaml
# config/services.yaml
services:
    App\EventListener\ExceptionListener:
        tags: [kernel.event_listener]
```

### PHP Configuration

```php
// config/services.php
use App\EventListener\ExceptionListener;

return function(ContainerConfigurator $container): void {
    $services = $container->services();
    $services->set(ExceptionListener::class)
        ->tag('kernel.event_listener');
};
```

### Tag Attributes

- **`event`** - Which event to listen to (auto-detected from type-hint if omitted)
- **`method`** - Which method to call (defaults to `__invoke()` or `on[EventName]()`)
- **`priority`** - Execution order; higher = earlier (default: 0)

### Method Resolution (without `event` attribute)

1. If `method` is defined, call that method
2. Try `__invoke()`
3. Throw exception

### Method Resolution (with `event` attribute)

1. If `method` is defined, call that method
2. Try `on` + PascalCased event name (e.g., `onKernelException()`)
3. Try `__invoke()`
4. Throw exception

## AsEventListener Attribute

The recommended way to define listeners in Symfony 7.4.

### Basic Usage (Invokable)

```php
namespace App\EventListener;

use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener]
final class MyListener
{
    public function __invoke(CustomEvent $event): void
    {
        // event type auto-detected from type-hint
    }
}
```

### Class-Level with Multiple Events

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener(event: CustomEvent::class, method: 'onCustomEvent')]
#[AsEventListener(event: 'foo', priority: 42)]
#[AsEventListener(event: 'bar', method: 'onBarEvent')]
final class MyMultiListener
{
    public function onCustomEvent(CustomEvent $event): void { /* ... */ }
    public function onFoo(): void { /* ... */ }
    public function onBarEvent(): void { /* ... */ }
}
```

### Method-Level Attributes

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

final class MyMultiListener
{
    #[AsEventListener]
    public function onCustomEvent(CustomEvent $event): void { /* ... */ }

    #[AsEventListener]
    public function onMultipleCustomEvent(CustomEvent|AnotherCustomEvent $event): void
    {
        // Union types supported as of Symfony 7.4
    }

    #[AsEventListener(event: 'foo', priority: 42)]
    public function onFoo(): void { /* ... */ }
}
```

## Event Subscribers

### EventSubscriberInterface

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class StoreSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            // Single method
            OrderPlacedEvent::class => 'onPlacedOrder',

            // Single method with priority
            OrderPlacedEvent::class => ['onPlacedOrder', 10],

            // Multiple methods with priorities
            KernelEvents::RESPONSE => [
                ['onKernelResponsePre', 10],
                ['onKernelResponsePost', -10],
            ],
        ];
    }

    public function onPlacedOrder(OrderPlacedEvent $event): void { /* ... */ }
    public function onKernelResponsePre(ResponseEvent $event): void { /* ... */ }
    public function onKernelResponsePost(ResponseEvent $event): void { /* ... */ }
}
```

### Registering Subscribers

```php
$dispatcher->addSubscriber(new StoreSubscriber());
```

Subscribers tagged `kernel.event_subscriber` are auto-registered when using the service container.

## Event Propagation

### Stopping Propagation

```php
public function onPlacedOrder(OrderPlacedEvent $event): void
{
    // Prevent subsequent listeners from being called
    $event->stopPropagation();
}
```

### Checking Propagation Status

```php
$dispatcher->dispatch($event);

if ($event->isPropagationStopped()) {
    // A listener stopped propagation
}
```

### Priority System

- Higher priority numbers execute first
- Default priority is 0
- Same priority: execution order matches registration order
- Example: priority 10 runs before priority 0, which runs before priority -10

## Kernel Events

Symfony dispatches these events during HTTP request processing:

### kernel.request (KernelEvents::REQUEST)

Dispatched early in request processing. Used to add information to the Request or return a Response early.

```php
use Symfony\Component\HttpKernel\Event\RequestEvent;

public function onKernelRequest(RequestEvent $event): void
{
    if (!$event->isMainRequest()) {
        return; // skip sub-requests
    }
    // ...
}
```

### kernel.controller (KernelEvents::CONTROLLER)

Dispatched after the controller is resolved but before it executes. Used for before filters.

```php
use Symfony\Component\HttpKernel\Event\ControllerEvent;

public function onKernelController(ControllerEvent $event): void
{
    $controller = $event->getController();
    if (is_array($controller)) {
        $controller = $controller[0];
    }

    if ($controller instanceof TokenAuthenticatedController) {
        $token = $event->getRequest()->query->get('token');
        if (!in_array($token, $this->tokens)) {
            throw new AccessDeniedHttpException('Invalid token!');
        }
    }
}
```

### kernel.response (KernelEvents::RESPONSE)

Dispatched after the controller returns a Response. Used for after filters.

```php
use Symfony\Component\HttpKernel\Event\ResponseEvent;

public function onKernelResponse(ResponseEvent $event): void
{
    $response = $event->getResponse();
    $response->headers->set('X-Custom-Header', 'value');
}
```

### kernel.exception (KernelEvents::EXCEPTION)

Dispatched when an uncaught exception is thrown.

```php
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpFoundation\Response;

public function onKernelException(ExceptionEvent $event): void
{
    $exception = $event->getThrowable();
    $response = new Response();
    $response->setContent($exception->getMessage());

    if ($exception instanceof HttpExceptionInterface) {
        $response->setStatusCode($exception->getStatusCode());
    } else {
        $response->setStatusCode(Response::HTTP_INTERNAL_SERVER_ERROR);
    }

    $event->setResponse($response);
}
```

### Main vs Sub-Requests

Always check `$event->isMainRequest()` in listeners that should only run for the main request.

## Service Container Integration

### RegisterListenersPass

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\EventDispatcher\DependencyInjection\RegisterListenersPass;
use Symfony\Component\EventDispatcher\EventDispatcher;

$container = new ContainerBuilder();
$container->addCompilerPass(new RegisterListenersPass());
$container->register('event_dispatcher', EventDispatcher::class);

// Register listener via tag
$container->register('listener_id', AcmeListener::class)
    ->addTag('kernel.event_listener', [
        'event' => 'acme.foo.action',
        'method' => 'onFooAction',
    ]);

// Register subscriber via tag
$container->register('subscriber_id', AcmeSubscriber::class)
    ->addTag('kernel.event_subscriber');
```

## Event Aliases

### Using FQCN as Event Names

```php
use Symfony\Component\HttpKernel\Event\RequestEvent;

class RequestSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            RequestEvent::class => 'onKernelRequest',
        ];
    }
}
```

### Custom Event Aliases

```php
// src/Kernel.php
use Symfony\Component\EventDispatcher\DependencyInjection\AddEventAliasesPass;

class Kernel extends BaseKernel
{
    protected function build(ContainerBuilder $container): void
    {
        $container->addCompilerPass(new AddEventAliasesPass([
            MyCustomEvent::class => 'my_custom_event',
        ]));
    }
}
```

## Listeners vs Subscribers

| Aspect | Listeners | Subscribers |
|--------|-----------|-------------|
| Reusability | Less flexible | Easier to reuse |
| Configuration | Event info in service config or attribute | Kept in class via `getSubscribedEvents()` |
| Flexibility | More flexible (external config) | Less flexible |
| Recommended for | Simple, single-event handlers | Multiple related events |

## Debugging

```bash
# Show all events and listeners
php bin/console debug:event-dispatcher

# Show listeners for a specific event
php bin/console debug:event-dispatcher kernel.exception

# Filter by partial name
php bin/console debug:event-dispatcher kernel
php bin/console debug:event-dispatcher Security

# Specific dispatcher
php bin/console debug:event-dispatcher --dispatcher=security.event_dispatcher.main
```

## API Reference

### EventDispatcher Methods

```php
public function addListener(string $eventName, callable $listener, int $priority = 0): void
public function addSubscriber(EventSubscriberInterface $subscriber): void
public function dispatch(object $event, ?string $eventName = null): object
public function getListeners(?string $eventName = null): array
public function getListenerPriority(string $eventName, callable $listener): ?int
public function removeListener(string $eventName, callable $listener): void
public function removeSubscriber(EventSubscriberInterface $subscriber): void
public function hasListeners(?string $eventName = null): bool
```

### Event Methods

```php
public function stopPropagation(): void
public function isPropagationStopped(): bool
```

### Other Dispatchers

- **ImmutableEventDispatcher** - Wraps a dispatcher, preventing modification (no addListener/removeListener)
- **TraceableEventDispatcher** - Wraps a dispatcher for debugging/profiling event flow

### GenericEvent

```php
use Symfony\Component\EventDispatcher\GenericEvent;

$event = new GenericEvent($subject, ['key' => 'value']);
$event->getSubject();          // returns $subject
$event->getArgument('key');    // returns 'value'
$event->getArguments();        // returns ['key' => 'value']
$event->setArgument('key', 'new');
$event->hasArgument('key');    // true
```
