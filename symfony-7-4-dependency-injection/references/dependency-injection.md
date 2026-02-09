# The DependencyInjection Component - Symfony 7.4

Source: https://symfony.com/doc/7.4/components/dependency_injection.html
GitHub: https://github.com/symfony/dependency-injection

The DependencyInjection component implements a PSR-11 compatible service container that standardizes and centralizes how objects are constructed in your application.

## Installation

```bash
composer require symfony/dependency-injection
```

## Basic Usage

### Creating a Simple Service

Define a class:

```php
class Mailer
{
    public function __construct(
        private string $transport,
    ) {
    }
}
```

Register it in the container:

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;

$container = new ContainerBuilder();
$container
    ->register('mailer', 'Mailer')
    ->addArgument('sendmail');
```

### Using Parameters

Extract configuration into parameters:

```php
$container = new ContainerBuilder();
$container->setParameter('mailer.transport', 'sendmail');
$container
    ->register('mailer', 'Mailer')
    ->addArgument('%mailer.transport%');
```

### Injecting Dependencies

Class with dependency:

```php
class NewsletterManager
{
    public function __construct(
        private \Mailer $mailer,
    ) {
    }
}
```

Register with service injection:

```php
use Symfony\Component\DependencyInjection\Reference;

$container
    ->register('newsletter_manager', 'NewsletterManager')
    ->addArgument(new Reference('mailer'));
```

### Setter Injection

For optional dependencies:

```php
class NewsletterManager
{
    private \Mailer $mailer;

    public function setMailer(\Mailer $mailer): void
    {
        $this->mailer = $mailer;
    }
}
```

Register with method call:

```php
$container
    ->register('newsletter_manager', 'NewsletterManager')
    ->addMethodCall('setMailer', [new Reference('mailer')]);
```

### Retrieving Services

```php
$newsletterManager = $container->get('newsletter_manager');
```

## Handling Missing Services

Control behavior when services don't exist:

```php
use Symfony\Component\DependencyInjection\ContainerInterface;

$newsletterManager = $containerBuilder->get(
    'newsletter_manager',
    ContainerInterface::EXCEPTION_ON_INVALID_REFERENCE
);
```

Available behaviors:
- `EXCEPTION_ON_INVALID_REFERENCE` - throws exception at compile time (default)
- `RUNTIME_EXCEPTION_ON_INVALID_REFERENCE` - throws exception at runtime
- `NULL_ON_INVALID_REFERENCE` - returns `null`
- `IGNORE_ON_INVALID_REFERENCE` - ignores the wrapping command
- `IGNORE_ON_UNINITIALIZED_REFERENCE` - ignores uninitialized services

## Configuration Files

### Loading YAML Configuration

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

$container = new ContainerBuilder();
$loader = new YamlFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.yaml');
```

services.yaml:

```yaml
parameters:
    mailer.transport: sendmail

services:
    mailer:
        class: Mailer
        arguments: ['%mailer.transport%']
    newsletter_manager:
        class: NewsletterManager
        calls:
            - [setMailer, ['@mailer']]
```

### Loading XML Configuration

```php
use Symfony\Component\DependencyInjection\Loader\XmlFileLoader;

$loader = new XmlFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.xml');
```

services.xml:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<container xmlns="http://symfony.com/schema/dic/services"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://symfony.com/schema/dic/services
        https://symfony.com/schema/dic/services/services-1.0.xsd">

    <parameters>
        <parameter key="mailer.transport">sendmail</parameter>
    </parameters>

    <services>
        <service id="mailer" class="Mailer">
            <argument>%mailer.transport%</argument>
        </service>

        <service id="newsletter_manager" class="NewsletterManager">
            <call method="setMailer">
                <argument type="service" id="mailer"/>
            </call>
        </service>
    </services>
</container>
```

### Loading PHP Configuration

```php
use Symfony\Component\DependencyInjection\Loader\PhpFileLoader;

$loader = new PhpFileLoader($container, new FileLocator(__DIR__));
$loader->load('services.php');
```

services.php:

```php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return static function (ContainerConfigurator $container): void {
    $container->parameters()
        ->set('mailer.transport', 'sendmail')
    ;

    $services = $container->services();
    $services->set('mailer', 'Mailer')
        ->args([param('mailer.transport')])
    ;

    $services->set('newsletter_manager', 'NewsletterManager')
        ->call('setMailer', [service('mailer')])
    ;
};
```

## Autowiring

Autowiring allows the container to automatically resolve constructor arguments by their type hints. Enable it in service defaults:

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'
        exclude: '../src/{DependencyInjection,Entity,Kernel.php}'
```

PHP configuration equivalent:

```php
namespace Symfony\Component\DependencyInjection\Loader\Configurator;

return static function (ContainerConfigurator $container): void {
    $services = $container->services();

    $services->defaults()
        ->autowire()
        ->autoconfigure();

    $services->load('App\\', '../src/')
        ->exclude('../src/{DependencyInjection,Entity,Kernel.php}');
};
```

### Autowiring with Interfaces

When multiple implementations exist for an interface, create an alias:

```yaml
services:
    App\Mailer\MailerInterface: '@App\Mailer\SmtpMailer'
```

Or use the `#[AsAlias]` attribute:

```php
use Symfony\Component\DependencyInjection\Attribute\AsAlias;

#[AsAlias(MailerInterface::class)]
class SmtpMailer implements MailerInterface
{
    // ...
}
```

### Named Autowiring with #[Target]

```php
use Symfony\Component\DependencyInjection\Attribute\Target;

class NewsletterManager
{
    public function __construct(
        #[Target('smtp_mailer')] private MailerInterface $mailer,
    ) {
    }
}
```

## Compiler Passes

Compiler passes allow you to manipulate service definitions during container compilation:

```php
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

class CustomPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->has('app.mailer')) {
            return;
        }

        $definition = $container->findDefinition('app.mailer');
        $taggedServices = $container->findTaggedServiceIds('app.mail_transport');

        foreach ($taggedServices as $id => $tags) {
            $definition->addMethodCall('addTransport', [new Reference($id)]);
        }
    }
}
```

Register compiler passes in the Kernel or Bundle:

```php
// src/Kernel.php
protected function build(ContainerBuilder $container): void
{
    $container->addCompilerPass(new CustomPass());
}
```

### Compiler Pass Priorities

```php
use Symfony\Component\DependencyInjection\Compiler\PassConfig;

$container->addCompilerPass(new CustomPass(), PassConfig::TYPE_BEFORE_OPTIMIZATION, 10);
```

Pass types (execution order):
1. `TYPE_BEFORE_OPTIMIZATION`
2. `TYPE_OPTIMIZE`
3. `TYPE_BEFORE_REMOVING`
4. `TYPE_REMOVE`
5. `TYPE_AFTER_REMOVING`

## Service Tags

Tags allow services to be collected and processed together:

```yaml
services:
    App\Mailer\SmtpTransport:
        tags: ['app.mail_transport']

    App\Mailer\SendmailTransport:
        tags:
            - { name: 'app.mail_transport', priority: 10 }
```

### Autoconfigure with Tags

```php
use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;

#[AutoconfigureTag('app.mail_transport')]
class SmtpTransport implements TransportInterface
{
    // ...
}
```

### Tagged Iterator

Inject all tagged services:

```php
use Symfony\Component\DependencyInjection\Attribute\TaggedIterator;

class TransportChain
{
    public function __construct(
        #[TaggedIterator('app.mail_transport')] private iterable $transports,
    ) {
    }
}
```

### Tagged Locator

Lazy-load tagged services:

```php
use Symfony\Component\DependencyInjection\Attribute\TaggedLocator;
use Psr\Container\ContainerInterface;

class TransportChain
{
    public function __construct(
        #[TaggedLocator('app.mail_transport')] private ContainerInterface $transports,
    ) {
    }

    public function getTransport(string $name): TransportInterface
    {
        return $this->transports->get($name);
    }
}
```

## Service Decoration

Replace or wrap an existing service:

```yaml
services:
    App\Mailer\DecoratingMailer:
        decorates: App\Mailer\Mailer
        arguments: ['@.inner']
```

PHP attribute:

```php
use Symfony\Component\DependencyInjection\Attribute\AsDecorator;
use Symfony\Component\DependencyInjection\Attribute\AutowireDecorated;

#[AsDecorator(decorates: Mailer::class)]
class DecoratingMailer implements MailerInterface
{
    public function __construct(
        #[AutowireDecorated] private MailerInterface $inner,
    ) {
    }
}
```

### Decoration Priority

```php
#[AsDecorator(decorates: Mailer::class, priority: 5)]
class HighPriorityDecorator implements MailerInterface { }

#[AsDecorator(decorates: Mailer::class, priority: 1)]
class LowPriorityDecorator implements MailerInterface { }
```

## Lazy Services

Services that are expensive to instantiate can be made lazy:

```yaml
services:
    App\Mailer\Mailer:
        lazy: true
```

Or with the attribute:

```php
use Symfony\Component\DependencyInjection\Attribute\Autoconfigure;

#[Autoconfigure(lazy: true)]
class Mailer
{
    // ...
}
```

## Factory Services

Create services using factory methods:

```yaml
services:
    App\Email\NewsletterManager:
        factory: ['@App\Email\NewsletterManagerFactory', 'createNewsletterManager']
```

Static factory:

```yaml
services:
    App\Email\NewsletterManager:
        factory: ['App\Email\NewsletterManagerStaticFactory', 'createNewsletterManager']
```

## Service Aliases

```yaml
services:
    App\Mailer\MailerInterface:
        alias: App\Mailer\SmtpMailer
```

Public aliases (accessible from the container directly):

```yaml
services:
    app.mailer:
        alias: App\Mailer\SmtpMailer
        public: true
```

## Parent Services

Share configuration between similar service definitions:

```yaml
services:
    _defaults:
        autowire: true

    app.base_mailer:
        abstract: true
        arguments:
            $sender: 'noreply@example.com'

    App\Mailer\SmtpMailer:
        parent: app.base_mailer
        arguments:
            $host: 'smtp.example.com'

    App\Mailer\SendmailMailer:
        parent: app.base_mailer
```

## Immutable Setters (Wither Pattern)

```php
class Mailer
{
    private LoggerInterface $logger;

    #[Required]
    public function withLogger(LoggerInterface $logger): static
    {
        $new = clone $this;
        $new->logger = $logger;
        return $new;
    }
}
```

## #[Autowire] Attribute

Fine-grained control over autowired arguments:

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;

class SomeService
{
    public function __construct(
        #[Autowire(service: 'some.specific.service')] private SomeInterface $service,
        #[Autowire('%kernel.project_dir%')] private string $projectDir,
        #[Autowire(env: 'DATABASE_URL')] private string $databaseUrl,
        #[Autowire(expression: "service('router').generate('home')")] private string $homeUrl,
    ) {
    }
}
```

## #[When] Attribute

Conditionally register services per environment:

```php
use Symfony\Component\DependencyInjection\Attribute\When;

#[When(env: 'dev')]
class DebugMailer implements MailerInterface
{
    // Only registered in the "dev" environment
}
```

## Best Practices

1. **Minimize container dependency** - Inject services into classes rather than having them retrieve from the container.
2. **Use the container at entry points only** - Retrieve services only at the application's entry point.
3. **Use configuration files** - Organize service definitions in dedicated configuration files for larger applications.
4. **Prefer autowiring** - Let the container resolve dependencies automatically via type hints.
5. **Use autoconfigure** - Automatically apply tags based on implemented interfaces.
6. **Keep compiler passes simple** - Each pass should do one thing well.

## Additional Topics

For more advanced topics, see the Symfony documentation:
- [Compiling the Container](https://symfony.com/doc/7.4/components/dependency_injection/compilation.html)
- [Container Building Workflow](https://symfony.com/doc/7.4/components/dependency_injection/workflow.html)
- [Service Subscribers & Locators](https://symfony.com/doc/7.4/service_container/service_subscribers_locators.html)
- [Service Closures](https://symfony.com/doc/7.4/service_container/service_closures.html)
- [Performance](https://symfony.com/doc/7.4/components/dependency_injection/compilation.html)
