---
name: "symfony-7-4-messenger"
description: "Symfony 7.4 Messenger component reference for async message processing and queuing. Use when creating, configuring, or debugging message handlers, message buses, transports, middleware, or any async/queue-related Symfony code. Triggers on: Messenger, message bus, handlers, transports, AMQP, RabbitMQ, async processing, middleware, envelope, stamps, #[AsMessageHandler], MessageHandlerInterface, MessageBusInterface, RoutableMessageBus, messenger:consume, queues, workers, retry strategy, failure transport, DelayStamp, TransportNamesStamp."
---

# Symfony 7.4 Messenger Component

GitHub: https://github.com/symfony/messenger
Docs: https://symfony.com/doc/7.4/messenger.html

## Quick Reference

### Creating a Message

```php
namespace App\Message;

class SmsNotification
{
    public function __construct(
        private string $content,
    ) {}

    public function getContent(): string
    {
        return $this->content;
    }
}
```

### Creating a Handler

```php
namespace App\MessageHandler;

use App\Message\SmsNotification;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
class SmsNotificationHandler
{
    public function __invoke(SmsNotification $message): void
    {
        // Process the message
    }
}
```

### Dispatching Messages

```php
use Symfony\Component\Messenger\MessageBusInterface;

class MyController
{
    public function __construct(private MessageBusInterface $bus) {}

    public function action(): Response
    {
        $this->bus->dispatch(new SmsNotification('Hello!'));
        return new Response('Message sent');
    }
}
```

### Basic Configuration (config/packages/messenger.yaml)

```yaml
framework:
    messenger:
        transports:
            async: "%env(MESSENGER_TRANSPORT_DSN)%"
            failed: 'doctrine://default?queue_name=failed'

        failure_transport: failed

        routing:
            'App\Message\SmsNotification': async
```

### Transport DSN Examples

```env
# RabbitMQ (AMQP)
MESSENGER_TRANSPORT_DSN=amqp://guest:guest@localhost:5672/%2f/messages

# Doctrine (database)
MESSENGER_TRANSPORT_DSN=doctrine://default

# Redis
MESSENGER_TRANSPORT_DSN=redis://localhost:6379/messages

# Synchronous
MESSENGER_TRANSPORT_DSN=sync://
```

### Consuming Messages

```bash
# Basic consumption
php bin/console messenger:consume async

# With limits
php bin/console messenger:consume async --limit=10 --time-limit=3600 --memory-limit=128M

# Multiple transports (priority order)
php bin/console messenger:consume async_high async_low

# Verbose output
php bin/console messenger:consume async -vv
```

### Using Stamps

```php
use Symfony\Component\Messenger\Stamp\DelayStamp;
use Symfony\Component\Messenger\Stamp\TransportNamesStamp;

// Delay message by 5 seconds
$bus->dispatch(new SmsNotification('Hello'), [new DelayStamp(5000)]);

// Override transport at runtime
$bus->dispatch(new SmsNotification('Hello'), [new TransportNamesStamp(['sync'])]);
```

### Retry Configuration

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 3
                    delay: 1000
                    multiplier: 2
                    max_delay: 60000
```

### Failed Messages

```bash
php bin/console messenger:failed:show          # View failed messages
php bin/console messenger:failed:retry 20 --force  # Retry specific message
php bin/console messenger:failed:remove 20     # Remove failed message
```

### Handler with Bus Restriction

```php
#[AsMessageHandler(bus: 'messenger.bus.command')]
class CreateUserHandler
{
    public function __invoke(CreateUserCommand $message): void { }
}
```

### Message Routing with Attribute (Symfony 7.2+)

```php
use Symfony\Component\Messenger\Attribute\AsMessage;

#[AsMessage('async')]
class SmsNotification { }
```

## Full Documentation

For complete details including all transport options (AMQP, Doctrine, Redis, SQS, Beanstalkd), middleware configuration, envelope stamps, failure handling, multiple buses, worker deployment with Supervisor/systemd, transactional messages, custom serializers, and message signing, see [references/messenger.md](references/messenger.md).
