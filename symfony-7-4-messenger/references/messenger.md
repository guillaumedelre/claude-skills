# Symfony 7.4 Messenger - Complete Reference

## Table of Contents

- [Installation](#installation)
- [Core Concepts](#core-concepts)
- [Messages and Handlers](#messages-and-handlers)
- [Message Bus](#message-bus)
- [Transports](#transports)
- [Transport Configuration](#transport-configuration)
- [Consuming Messages](#consuming-messages)
- [Retry and Failure Handling](#retry-and-failure-handling)
- [Envelopes and Stamps](#envelopes-and-stamps)
- [Middleware](#middleware)
- [Multiple Buses](#multiple-buses)
- [Advanced Features](#advanced-features)
- [Production Deployment](#production-deployment)
- [Events](#events)
- [Testing](#testing)

---

## Installation

```bash
composer require symfony/messenger
```

For specific transports:

```bash
composer require symfony/amqp-messenger      # RabbitMQ
composer require symfony/doctrine-messenger  # Doctrine/Database
composer require symfony/redis-messenger     # Redis
composer require symfony/amazon-sqs-messenger # Amazon SQS
composer require symfony/beanstalkd-messenger # Beanstalkd
```

## Core Concepts

### Sender
Responsible for serializing and sending messages to message brokers or third-party APIs.

### Receiver
Responsible for retrieving, deserializing, and forwarding messages to handlers.

### Handler
Responsible for handling messages using applicable business logic. Called by `HandleMessageMiddleware`.

### Middleware
Cross-cutting concerns (logging, validation, transactions) that process messages as they pass through the bus.

### Envelope
Wraps messages within the message bus, allowing metadata to be attached via envelope stamps.

### Envelope Stamps
Metadata pieces attached to messages:
- `DelayStamp` - Delay asynchronous message handling
- `DispatchAfterCurrentBusStamp` - Handle after current bus execution
- `HandledStamp` - Mark message as handled with return value
- `ReceivedStamp` - Mark message as received from transport
- `SentStamp` - Mark message as sent by specific sender
- `SerializerStamp` - Configure serialization groups
- `ValidationStamp` - Configure validation groups
- `TransportNamesStamp` - Override transport at runtime
- `ErrorDetailsStamp` - Internal stamp for handler exceptions
- `ScheduledStamp` - Mark message from scheduler

## Messages and Handlers

### Creating a Message

Messages are simple PHP classes that hold data:

```php
namespace App\Message;

class SmsNotification
{
    public function __construct(
        private string $content,
        private string $phoneNumber,
    ) {}

    public function getContent(): string
    {
        return $this->content;
    }

    public function getPhoneNumber(): string
    {
        return $this->phoneNumber;
    }
}
```

### Creating a Handler

Handlers process messages using the `#[AsMessageHandler]` attribute:

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
        $this->sendSms($message->getPhoneNumber(), $message->getContent());
    }
}
```

### Handler with Multiple Message Types

```php
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

class NotificationHandler
{
    #[AsMessageHandler]
    public function handleSms(SmsNotification $message): void
    {
        // Handle SMS
    }

    #[AsMessageHandler]
    public function handleEmail(EmailNotification $message): void
    {
        // Handle email
    }
}
```

### Handler Options

```php
#[AsMessageHandler(
    bus: 'messenger.bus.command',    // Restrict to specific bus
    fromTransport: 'async',          // Only handle from specific transport
    handles: SmsNotification::class, // Explicit message class
    method: 'handleMessage',         // Custom method name
    priority: 10,                    // Handler priority (higher = earlier)
)]
class MyHandler
{
    public function handleMessage(SmsNotification $message): void {}
}
```

### Handler Interface (Alternative)

```php
use Symfony\Component\Messenger\Handler\MessageHandlerInterface;

class SmsNotificationHandler implements MessageHandlerInterface
{
    public function __invoke(SmsNotification $message): void
    {
        // Handle message
    }
}
```

### Debugging Handlers

```bash
php bin/console debug:messenger
```

## Message Bus

### Dispatching Messages

```php
use Symfony\Component\Messenger\MessageBusInterface;

class MyController
{
    public function __construct(
        private MessageBusInterface $bus,
    ) {}

    public function action(): Response
    {
        $this->bus->dispatch(new SmsNotification('Hello!', '+1234567890'));

        return new Response('Message dispatched');
    }
}
```

### Dispatching with Stamps

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Stamp\DelayStamp;

$envelope = new Envelope(new SmsNotification('Hello!', '+1234567890'), [
    new DelayStamp(5000), // Delay by 5 seconds
]);

$this->bus->dispatch($envelope);

// Or shorthand:
$this->bus->dispatch(new SmsNotification('Hello!', '+1234567890'), [
    new DelayStamp(5000),
]);
```

### Getting Handler Results

```php
use Symfony\Component\Messenger\Stamp\HandledStamp;

$envelope = $this->bus->dispatch(new MyQuery());

/** @var HandledStamp[] $handledStamps */
$handledStamps = $envelope->all(HandledStamp::class);

$result = $handledStamps[0]->getResult();
```

### Manual Bus Setup (Component Only)

```php
use App\Message\MyMessage;
use App\MessageHandler\MyMessageHandler;
use Symfony\Component\Messenger\Handler\HandlersLocator;
use Symfony\Component\Messenger\MessageBus;
use Symfony\Component\Messenger\Middleware\HandleMessageMiddleware;

$handler = new MyMessageHandler();

$bus = new MessageBus([
    new HandleMessageMiddleware(new HandlersLocator([
        MyMessage::class => [$handler],
    ])),
]);

$bus->dispatch(new MyMessage(/* ... */));
```

## Transports

### Environment Variable Setup

```env
# .env
# RabbitMQ (AMQP)
MESSENGER_TRANSPORT_DSN=amqp://guest:guest@localhost:5672/%2f/messages

# Doctrine (database)
MESSENGER_TRANSPORT_DSN=doctrine://default

# Redis
MESSENGER_TRANSPORT_DSN=redis://localhost:6379/messages

# Synchronous (no queue)
MESSENGER_TRANSPORT_DSN=sync://

# In-memory (for testing)
MESSENGER_TRANSPORT_DSN=in-memory://
```

### Basic Configuration

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async: "%env(MESSENGER_TRANSPORT_DSN)%"
            sync: 'sync://'

        routing:
            'App\Message\SmsNotification': async
            'App\Message\EmailNotification': async
```

### Message Routing with Attribute (Symfony 7.2+)

```php
use Symfony\Component\Messenger\Attribute\AsMessage;

#[AsMessage('async')]
class SmsNotification
{
    // Message will be routed to 'async' transport
}

// Multiple transports:
#[AsMessage(['async', 'audit'])]
class CriticalMessage {}
```

### Routing with Wildcards

```yaml
framework:
    messenger:
        routing:
            'App\Message\*': async           # All messages in namespace
            '*': sync                         # Default for unmatched
            'App\Message\AsyncInterface': async  # Interface-based routing
```

## Transport Configuration

### AMQP Transport (RabbitMQ)

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: 'amqp://guest:guest@localhost:5672/%2f/messages'
                options:
                    auto_setup: true
                    exchange:
                        name: my_exchange
                        type: direct  # fanout, direct, topic
                    queues:
                        messages:
                            binding_keys: [my_routing_key]
                    delay:
                        exchange_name: delays
                        queue_name_pattern: 'delay_%exchange_name%_%routing_key%_%delay%'
```

**AMQP Options:**

| Option | Description |
|--------|-------------|
| `auto_setup` | Create exchanges/queues automatically |
| `cacert` | TLS/SSL certificate path |
| `heartbeat` | Connection heartbeat (seconds) |
| `channel_max` | Max channel number |
| `frame_max` | Max frame size |
| `confirm_timeout` | Message confirmation timeout |
| `connect_timeout` | Connection timeout |
| `vhost` | Virtual host |
| `exchange[name]` | Exchange name |
| `exchange[type]` | `fanout`, `direct`, `topic` |
| `queues[name][binding_keys]` | Queue binding keys |

**Custom AMQP Stamps:**

```php
use Symfony\Component\Messenger\Bridge\Amqp\Transport\AmqpStamp;

$this->bus->dispatch(new SmsNotification('Hello!', '+123'), [
    new AmqpStamp('custom-routing-key', AMQP_NOPARAM, [
        'headers' => ['x-custom-header' => 'value'],
    ]),
]);
```

### Doctrine Transport

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: 'doctrine://default'
                options:
                    table_name: messenger_messages
                    queue_name: default
                    redeliver_timeout: 3600
                    auto_setup: true
```

**Doctrine Options:**

| Option | Description |
|--------|-------------|
| `table_name` | Custom table name |
| `queue_name` | Queue identifier for multiple transports |
| `redeliver_timeout` | Timeout for redelivering abandoned messages (seconds) |
| `auto_setup` | Create table automatically |
| `use_notify` | Use PostgreSQL LISTEN/NOTIFY |
| `check_delayed_interval` | Check for delayed messages (ms) |
| `get_notify_timeout` | LISTEN timeout (ms) |

### Redis Transport

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: 'redis://localhost:6379/messages'
                options:
                    stream: messages
                    group: my_group
                    consumer: consumer_name
                    auto_setup: true
                    delete_after_ack: false
                    lazy: false
                    sentinel_master: null
```

### Amazon SQS Transport

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: 'sqs://ACCESS_KEY:SECRET_KEY@us-east-1/my-queue'
                options:
                    auto_setup: true
                    wait_time: 20
                    poll_timeout: 0.1
```

### Beanstalkd Transport

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: 'beanstalkd://localhost:11300'
                options:
                    tube_name: 'my_tube'
                    timeout: 0
                    ttr: 90
```

### In-Memory Transport (Testing)

```yaml
# config/packages/test/messenger.yaml
framework:
    messenger:
        transports:
            async: 'in-memory://'
```

## Consuming Messages

### Basic Consumption

```bash
# Consume from a single transport
php bin/console messenger:consume async

# Consume from multiple transports (priority order)
php bin/console messenger:consume async_priority_high async_priority_low

# Consume from all transports
php bin/console messenger:consume --all

# Exclude specific receivers
php bin/console messenger:consume --all --exclude-receivers=async_priority_low

# Verbose output
php bin/console messenger:consume async -vv
```

### Worker Options

| Option | Description |
|--------|-------------|
| `--limit=N` | Exit after processing N messages |
| `--time-limit=N` | Exit after N seconds |
| `--memory-limit=N` | Exit if memory exceeds N (e.g., 128M) |
| `--keepalive=N` | Send keepalive every N seconds |
| `--failure-limit=N` | Exit after N failures |
| `--no-reset` | Don't reset container between messages |
| `--queues=name` | Consume from specific queues (AMQP) |
| `--no-sleep` | Busy wait on empty queue |
| `--bus=name` | Specify message bus |

### Checking Message Counts

```bash
# Display stats for all transports
php bin/console messenger:stats

# Display stats for specific transports
php bin/console messenger:stats async failed

# JSON output
php bin/console messenger:stats --format=json
```

### Stop Workers

```bash
php bin/console messenger:stop-workers
```

## Retry and Failure Handling

### Retry Configuration

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 3
                    delay: 1000           # milliseconds
                    multiplier: 2         # exponential backoff
                    max_delay: 60000      # max delay in milliseconds
                    jitter: 0.1           # 10% randomness
```

### Failure Transport (Dead Letter Queue)

```yaml
framework:
    messenger:
        failure_transport: failed

        transports:
            async: "%env(MESSENGER_TRANSPORT_DSN)%"
            failed: 'doctrine://default?queue_name=failed'
```

### Multiple Failure Transports

```yaml
framework:
    messenger:
        failure_transport: failed_default

        transports:
            async_high:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                failure_transport: failed_high_priority

            async_low:
                dsn: 'doctrine://default?queue_name=async_low'
                # Uses global failure_transport

            failed_default: 'doctrine://default?queue_name=failed_default'
            failed_high_priority: 'doctrine://default?queue_name=failed_high'
```

### Failed Message Commands

```bash
# View failed messages
php bin/console messenger:failed:show

# View with limit
php bin/console messenger:failed:show --max=10

# Filter by message class
php bin/console messenger:failed:show --class-filter='App\Message\MyMessage'

# Show stats
php bin/console messenger:failed:show --stats

# Show details for specific message
php bin/console messenger:failed:show 20 -vv

# Retry failed messages (interactive)
php bin/console messenger:failed:retry -vv

# Force retry specific messages
php bin/console messenger:failed:retry 20 30 --force

# Remove message
php bin/console messenger:failed:remove 20

# Remove with preview
php bin/console messenger:failed:remove 20 30 --show-messages

# Remove all
php bin/console messenger:failed:remove --all

# Remove by class
php bin/console messenger:failed:remove --class-filter='App\Message\MyMessage'

# Specific failure transport
php bin/console messenger:failed:show --transport=failed_high_priority
```

### Custom Retry Exceptions

```php
use Symfony\Component\Messenger\Exception\UnrecoverableMessageHandlingException;
use Symfony\Component\Messenger\Exception\RecoverableMessageHandlingException;

class MyHandler
{
    public function __invoke(MyMessage $message): void
    {
        try {
            // Handle message
        } catch (PermanentException $e) {
            // Don't retry - send directly to failure transport
            throw new UnrecoverableMessageHandlingException(
                'Permanent failure',
                0,
                $e
            );
        } catch (TemporaryException $e) {
            // Always retry (ignores max_retries)
            throw new RecoverableMessageHandlingException(
                'Temporary failure',
                0,
                $e,
                5000  // Custom delay in milliseconds (Symfony 7.2+)
            );
        }
    }
}
```

## Envelopes and Stamps

### Common Stamps

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Stamp\DelayStamp;
use Symfony\Component\Messenger\Stamp\TransportNamesStamp;
use Symfony\Component\Messenger\Stamp\SerializerStamp;
use Symfony\Component\Messenger\Stamp\DispatchAfterCurrentBusStamp;
use Symfony\Component\Messenger\Stamp\ValidationStamp;

// Delay message
$bus->dispatch(new MyMessage(), [new DelayStamp(5000)]);

// Override transport at runtime
$bus->dispatch(new MyMessage(), [new TransportNamesStamp(['sync'])]);

// Configure serialization groups
$bus->dispatch(new MyMessage(), [new SerializerStamp([
    'groups' => ['my_serialization_groups'],
])]);

// Dispatch after current bus execution completes
$bus->dispatch(new FollowUpMessage(), [new DispatchAfterCurrentBusStamp()]);

// Configure validation groups
$bus->dispatch(new MyMessage(), [new ValidationStamp(['my_group'])]);
```

### Working with Envelopes

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Stamp\ReceivedStamp;

$envelope = new Envelope($message, [new DelayStamp(1000)]);

// Add more stamps
$envelope = $envelope->with(new AnotherStamp());

// Get all stamps
$allStamps = $envelope->all();

// Get stamps by type
$receivedStamps = $envelope->all(ReceivedStamp::class);

// Get last stamp of type
$lastStamp = $envelope->last(ReceivedStamp::class);

// Get the message
$message = $envelope->getMessage();
```

### Custom Stamps

```php
use Symfony\Component\Messenger\Stamp\StampInterface;

class AuditStamp implements StampInterface
{
    public function __construct(
        private string $userId,
        private \DateTimeImmutable $timestamp,
    ) {}

    public function getUserId(): string
    {
        return $this->userId;
    }

    public function getTimestamp(): \DateTimeImmutable
    {
        return $this->timestamp;
    }
}

// Usage
$bus->dispatch(new MyMessage(), [
    new AuditStamp($currentUserId, new \DateTimeImmutable()),
]);
```

## Middleware

### Custom Middleware

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Middleware\MiddlewareInterface;
use Symfony\Component\Messenger\Middleware\StackInterface;
use Symfony\Component\Messenger\Stamp\ReceivedStamp;

class LoggingMiddleware implements MiddlewareInterface
{
    public function __construct(
        private LoggerInterface $logger,
    ) {}

    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        $message = $envelope->getMessage();
        $this->logger->info('Handling message', [
            'class' => get_class($message),
        ]);

        // Call next middleware
        $envelope = $stack->next()->handle($envelope, $stack);

        $this->logger->info('Message handled', [
            'class' => get_class($message),
        ]);

        return $envelope;
    }
}
```

### Registering Middleware

```yaml
framework:
    messenger:
        buses:
            messenger.bus.default:
                middleware:
                    # Add custom middleware
                    - App\Messenger\LoggingMiddleware

                    # Add doctrine transaction middleware
                    - doctrine_transaction

                    # Add validation middleware
                    - validation
```

### Conditional Middleware (Checking Stamps)

```php
class MyMiddleware implements MiddlewareInterface
{
    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        if (null !== $envelope->last(ReceivedStamp::class)) {
            // Message was received from transport (async)
            $envelope = $envelope->with(new ProcessedAsyncStamp());
        } else {
            // Message was originally dispatched (sync or about to be sent)
        }

        return $stack->next()->handle($envelope, $stack);
    }
}
```

### Built-in Middleware

| Middleware | Description |
|------------|-------------|
| `SendMessageMiddleware` | Sends messages to transports |
| `HandleMessageMiddleware` | Calls registered handlers |
| `AddBusNameStampMiddleware` | Adds bus name stamp |
| `DispatchAfterCurrentBusMiddleware` | Handles `DispatchAfterCurrentBusStamp` |
| `FailedMessageProcessingMiddleware` | Handles failed message retry |
| `RejectRedeliveredMessageMiddleware` | Rejects already redelivered messages |
| `doctrine_transaction` | Wraps handler in Doctrine transaction |
| `validation` | Validates messages before handling |

## Multiple Buses

### Configuration

```yaml
framework:
    messenger:
        default_bus: messenger.bus.default

        buses:
            messenger.bus.command:
                middleware:
                    - doctrine_transaction
                    - validation

            messenger.bus.query:
                middleware:
                    - validation

            messenger.bus.event:
                default_middleware: allow_no_handlers
```

### Injecting Specific Buses

```php
use Symfony\Component\DependencyInjection\Attribute\Target;
use Symfony\Component\Messenger\MessageBusInterface;

class MyController
{
    public function __construct(
        #[Target('messenger.bus.command')]
        private MessageBusInterface $commandBus,

        #[Target('messenger.bus.query')]
        private MessageBusInterface $queryBus,
    ) {}

    public function action(): Response
    {
        $this->commandBus->dispatch(new CreateUserCommand('john'));

        $envelope = $this->queryBus->dispatch(new GetUserQuery(1));
        // Get result from envelope...

        return new Response('Done');
    }
}
```

### Restricting Handlers to Specific Buses

```php
#[AsMessageHandler(bus: 'messenger.bus.command')]
class CreateUserHandler
{
    public function __invoke(CreateUserCommand $command): void
    {
        // Only handles messages from messenger.bus.command
    }
}
```

## Advanced Features

### Priority-Based Routing

```yaml
framework:
    messenger:
        transports:
            async_priority_high:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queue_name: high

            async_priority_low:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queue_name: low

        routing:
            'App\Message\CriticalMessage': async_priority_high
            'App\Message\RegularMessage': async_priority_low
```

Consume with priority:

```bash
php bin/console messenger:consume async_priority_high async_priority_low
```

### Doctrine Entities in Messages

**Do not pass entity objects in messages.** Pass IDs instead:

```php
// Wrong - entity may be stale when processed
class NewUserWelcomeEmail
{
    public function __construct(private User $user) {}
}

// Correct - fetch fresh data in handler
class NewUserWelcomeEmail
{
    public function __construct(private int $userId) {}

    public function getUserId(): int
    {
        return $this->userId;
    }
}

class NewUserWelcomeEmailHandler
{
    public function __construct(private UserRepository $userRepository) {}

    public function __invoke(NewUserWelcomeEmail $message): void
    {
        $user = $this->userRepository->find($message->getUserId());
        if (!$user) {
            return; // User was deleted
        }
        // Send email...
    }
}
```

### Transactional Messages

Dispatch messages after the current bus execution completes:

```php
use Symfony\Component\Messenger\Stamp\DispatchAfterCurrentBusStamp;

class OrderCreatedHandler
{
    public function __construct(private MessageBusInterface $bus) {}

    public function __invoke(OrderCreatedEvent $event): void
    {
        // Process order...

        // This message won't be dispatched until current handler completes
        // If current handler throws, this message is NOT dispatched
        $this->bus->dispatch(
            new SendOrderConfirmationEmail($event->orderId),
            [new DispatchAfterCurrentBusStamp()]
        );
    }
}
```

### Rate Limited Transport

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                rate_limiter: my_limiter

    rate_limiter:
        my_limiter:
            policy: 'fixed_window'
            limit: 100
            interval: '60 seconds'
```

**Warning:** Rate limiting blocks the entire worker. Use dedicated workers for rate-limited transports.

### Stateless Workers & Service Reset

Services implementing `ResetInterface` are reset after each message:

```php
use Symfony\Contracts\Service\ResetInterface;

class StatefulService implements ResetInterface
{
    private array $cache = [];

    public function reset(): void
    {
        $this->cache = [];
    }
}
```

Disable reset:

```bash
php bin/console messenger:consume async --no-reset
```

### Custom Sender

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Transport\Sender\SenderInterface;

class ImportantActionToEmailSender implements SenderInterface
{
    public function __construct(
        private MailerInterface $mailer,
        private string $toEmail,
    ) {}

    public function send(Envelope $envelope): Envelope
    {
        $message = $envelope->getMessage();

        if (!$message instanceof ImportantAction) {
            throw new \InvalidArgumentException(
                sprintf('This transport only supports "%s" messages.',
                ImportantAction::class)
            );
        }

        $this->mailer->send(
            (new Email())
                ->to($this->toEmail)
                ->subject('Important action made')
                ->html('<h1>Important action</h1><p>Made by '
                    .$message->getUsername().'</p>')
        );

        return $envelope;
    }
}
```

### Custom Receiver

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Transport\Receiver\ReceiverInterface;

class NewOrdersFromCsvFileReceiver implements ReceiverInterface
{
    public function get(): iterable
    {
        $data = $this->readFromCsv();
        if (null === $data) {
            return [];
        }

        $envelope = new Envelope(new NewOrderMessage($data));
        return [$envelope->with(new CustomStamp($data['id']))];
    }

    public function ack(Envelope $envelope): void
    {
        // Mark as processed
    }

    public function reject(Envelope $envelope): void
    {
        // Handle rejection
    }
}
```

### Custom Serializer

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Transport\Serialization\SerializerInterface;

class CustomSerializer implements SerializerInterface
{
    public function encode(Envelope $envelope): array
    {
        $message = $envelope->getMessage();

        return [
            'body' => json_encode([
                'class' => get_class($message),
                'data' => $message->toArray(),
            ]),
            'headers' => [
                'Content-Type' => 'application/json',
            ],
        ];
    }

    public function decode(array $encoded): Envelope
    {
        $data = json_decode($encoded['body'], true);
        $class = $data['class'];
        $message = $class::fromArray($data['data']);

        return new Envelope($message);
    }
}
```

Configure:

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                serializer: App\Messenger\CustomSerializer
```

### Message Signing (Symfony 7.1+)

```yaml
framework:
    messenger:
        signing_secret: '%env(MESSENGER_SIGNING_SECRET)%'
```

Or per-bus:

```yaml
framework:
    messenger:
        buses:
            messenger.bus.default:
                options:
                    signing_secret: '%env(MESSENGER_SIGNING_SECRET)%'
```

## Production Deployment

### Using Supervisor

```ini
; /etc/supervisor/conf.d/messenger-worker.conf
[program:messenger-consume]
command=php /path/to/app/bin/console messenger:consume async --time-limit=3600
user=www-data
numprocs=2
startsecs=0
autostart=true
autorestart=true
startretries=10
process_name=%(program_name)s_%(process_num)02d
stdout_logfile=/var/log/supervisor/messenger-consume.log
stderr_logfile=/var/log/supervisor/messenger-consume-error.log
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start messenger-consume:*
```

### Using Systemd

```ini
# /etc/systemd/system/messenger-worker@.service
[Unit]
Description=Symfony messenger-consume %i

[Service]
ExecStart=/usr/bin/php /path/to/app/bin/console messenger:consume async --time-limit=3600
User=www-data
Environment="MESSENGER_CONSUMER_NAME=symfony-%n-%i"
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
```

```bash
# Single instance
sudo systemctl enable messenger-worker@1.service
sudo systemctl start messenger-worker@1.service

# Multiple instances
sudo systemctl enable messenger-worker@{1..4}.service
sudo systemctl start messenger-worker@{1..4}.service
```

### Graceful Shutdown

```yaml
framework:
    messenger:
        stop_worker_on_signals:
            - SIGTERM
            - SIGINT
            - SIGUSR1
```

Supervisor configuration:

```ini
[program:messenger-consume]
stopwaitsecs=20
stopsignal=SIGTERM
```

### Restart on Deploy

```bash
php bin/console messenger:stop-workers
# Process manager will restart workers with new code
```

### Cache Configuration

Use consistent cache prefix between deploys:

```yaml
framework:
    cache:
        prefix_seed: 'my_app'
```

## Events

### Messenger Events

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Messenger\Event\WorkerStartedEvent;
use Symfony\Component\Messenger\Event\WorkerMessageHandledEvent;
use Symfony\Component\Messenger\Event\WorkerMessageFailedEvent;
use Symfony\Component\Messenger\Event\WorkerMessageRetriedEvent;
use Symfony\Component\Messenger\Event\WorkerRunningEvent;
use Symfony\Component\Messenger\Event\WorkerStoppedEvent;
use Symfony\Component\Messenger\Event\SendMessageToTransportsEvent;

class MessengerEventListener
{
    #[AsEventListener]
    public function onWorkerStarted(WorkerStartedEvent $event): void
    {
        // Worker just started
    }

    #[AsEventListener]
    public function onMessageHandled(WorkerMessageHandledEvent $event): void
    {
        $envelope = $event->getEnvelope();
        // Message successfully handled
    }

    #[AsEventListener]
    public function onMessageFailed(WorkerMessageFailedEvent $event): void
    {
        $envelope = $event->getEnvelope();
        $throwable = $event->getThrowable();
        // Message handling failed
    }

    #[AsEventListener]
    public function onMessageRetried(WorkerMessageRetriedEvent $event): void
    {
        // Message will be retried
    }

    #[AsEventListener]
    public function onWorkerRunning(WorkerRunningEvent $event): void
    {
        // Worker is running (called between messages)
        if ($event->isWorkerIdle()) {
            // No messages to process
        }
    }

    #[AsEventListener]
    public function onWorkerStopped(WorkerStoppedEvent $event): void
    {
        // Worker stopped
    }

    #[AsEventListener]
    public function onSendToTransports(SendMessageToTransportsEvent $event): void
    {
        $envelope = $event->getEnvelope();
        // Message about to be sent to transports
    }
}
```

## Testing

### Testing with In-Memory Transport

```yaml
# config/packages/test/messenger.yaml
framework:
    messenger:
        transports:
            async: 'in-memory://'
```

### Testing Handler Directly

```php
use PHPUnit\Framework\TestCase;

class SmsNotificationHandlerTest extends TestCase
{
    public function testInvoke(): void
    {
        $smsSender = $this->createMock(SmsSenderInterface::class);
        $smsSender->expects($this->once())
            ->method('send')
            ->with('+1234567890', 'Hello!');

        $handler = new SmsNotificationHandler($smsSender);
        $handler(new SmsNotification('Hello!', '+1234567890'));
    }
}
```

### Testing Message Dispatch

```php
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;
use Symfony\Component\Messenger\Transport\InMemory\InMemoryTransport;

class OrderServiceTest extends KernelTestCase
{
    public function testOrderCreationDispatchesMessage(): void
    {
        self::bootKernel();

        $orderService = self::getContainer()->get(OrderService::class);
        $orderService->createOrder(['item' => 'Test']);

        /** @var InMemoryTransport $transport */
        $transport = self::getContainer()->get('messenger.transport.async');

        $this->assertCount(1, $transport->getSent());
        $this->assertInstanceOf(
            OrderCreatedEvent::class,
            $transport->getSent()[0]->getMessage()
        );
    }
}
```

### Testing Async Processing

```php
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;
use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Messenger\Transport\InMemory\InMemoryTransport;
use Symfony\Component\Messenger\Worker;

class AsyncProcessingTest extends KernelTestCase
{
    public function testAsyncMessageProcessing(): void
    {
        self::bootKernel();

        $bus = self::getContainer()->get(MessageBusInterface::class);
        $bus->dispatch(new ProcessDataMessage(['data' => 'test']));

        /** @var InMemoryTransport $transport */
        $transport = self::getContainer()->get('messenger.transport.async');

        // Process the queued message
        $worker = new Worker(
            ['async' => $transport],
            $bus
        );

        $worker->run(['sleep' => 0]);

        // Assert message was processed
        $this->assertCount(1, $transport->getAcknowledged());
    }
}
```
