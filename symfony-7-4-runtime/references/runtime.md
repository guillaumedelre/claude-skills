# Symfony Runtime Component - Full Documentation

## Overview

The Runtime Component decouples bootstrapping logic from global state, enabling applications to run with alternative PHP runtimes like PHP-PM, ReactPHP, Swoole, RoadRunner, and FrankenPHP without code changes.

**GitHub Repository**: https://github.com/symfony/runtime
**Official Documentation**: https://symfony.com/doc/7.4/components/runtime.html

## Installation

```bash
composer require symfony/runtime
```

**Note**: If installing outside a Symfony application, require `vendor/autoload.php` for Composer autoloading.

## Basic Usage

The Runtime component enables a minimal front-controller structure:

```php
// public/index.php
<?php

use App\Kernel;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return function (array $context): Kernel {
    return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
};
```

### How It Works

1. Instantiates a `RuntimeInterface` implementation
2. Includes the front-controller script
3. Resolves callable arguments via the Runtime
4. Invokes the callable to get the application
5. Uses the Runtime to execute the application

## Runtime Lifecycle (6 Steps)

1. **Entry Point**: Main entry point returns a callable (the "app")
2. **Resolver**: App callable passed to `RuntimeInterface::getResolver()` which returns `ResolverInterface` with app and arguments
3. **Invocation**: App callable invoked with resolved arguments, returns application object
4. **Runner**: Application object passed to `RuntimeInterface::getRunner()` which returns `RunnerInterface`
5. **Execution**: `RunnerInterface::run(object $application)` called, returns exit status code
6. **Termination**: PHP engine terminates with status code

## Available Runtimes

### SymfonyRuntime (Default)

Optimized for traditional PHP-FPM servers (Nginx, Apache). Supports HTTP requests, console commands, and Symfony-specific features.

Set via `APP_RUNTIME` environment variable or `extra.runtime.class` in `composer.json`.

### GenericRuntime

Uses PHP superglobals (`$_SERVER`, `$_POST`, `$_GET`, `$_FILES`, `$_SESSION`). Platform-agnostic alternative to SymfonyRuntime.

### Configuring Runtime

Define in `composer.json`:

```json
{
    "extra": {
        "runtime": {
            "class": "Symfony\\Component\\Runtime\\GenericRuntime"
        }
    }
}
```

## Resolvable Arguments

The closure can accept these arguments (both `SymfonyRuntime` and `GenericRuntime`):

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Application;

return function (
    Request $request,
    InputInterface $input,
    OutputInterface $output,
    Application $application,
    array $context,
    array $argv,
    array $request
): Application {
    // ...
};
```

### Supported Arguments Table

| Argument | Description |
|----------|-------------|
| `Request` | HTTP request created from globals |
| `InputInterface` | CLI input for options/arguments |
| `OutputInterface` | Styled CLI output |
| `Application` | Console application instance |
| `Command` | Single CLI command |
| `array $context` | `$_SERVER` + `$_ENV` combined |
| `array $argv` | Command arguments |
| `array $request` | Keys: `query`, `body`, `files`, `session` |

## Resolvable Applications

### HTTP Applications (Kernel)

```php
use App\Kernel;

return static function (): Kernel {
    return new Kernel('prod', false);
};
```

### Response Objects

```php
use Symfony\Component\HttpFoundation\Response;

return static function (): Response {
    return new Response('Hello world');
};
```

### Console Commands

```php
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

return static function (Command $command): Command {
    $command->setCode(static function (InputInterface $input, OutputInterface $output): void {
        $output->write('Hello World');
    });
    return $command;
};
```

### Console Applications

```php
use Symfony\Component\Console\Application;
use Symfony\Component\Console\Command\Command;

return static function (array $context): Application {
    $app = new Application();
    $command = new Command('hello');
    $command->setCode(/* ... */);
    $app->add($command);
    return $app;
};
```

### RunnerInterface Implementation

```php
use Symfony\Component\Runtime\RunnerInterface;

return static function (): RunnerInterface {
    return new class implements RunnerInterface {
        public function run(): int {
            echo 'Hello World';
            return 0;
        }
    };
};
```

### Callable Applications

```php
return static function (): callable {
    return static function(): int {
        echo 'Hello World';
        return 0;
    };
};
```

### Void Return

```php
return function (): void {
    echo 'Hello world';
};
```

## Console Application Example

```php
// bin/console
<?php

use App\Kernel;
use Symfony\Bundle\FrameworkBundle\Console\Application;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return function (array $context): Application {
    $kernel = new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
    return new Application($kernel);
};
```

## Runtime Options

### Via Environment Variables

```php
$_SERVER['APP_RUNTIME_OPTIONS'] = [
    'project_dir' => '/var/task',
];

// or JSON format (Symfony 7.4+)
$_SERVER['APP_RUNTIME_OPTIONS'] = '{"project_dir":"/var/task"}';
```

### Via Composer Configuration

```json
{
    "extra": {
        "runtime": {
            "project_dir": "/var/task"
        }
    }
}
```

## Configuration Options

### SymfonyRuntime Options

| Option | Default | Description |
|--------|---------|-------------|
| `env` | `APP_ENV` or `"dev"` | Environment name |
| `disable_dotenv` | `false` | Disable `.env` file loading |
| `dotenv_path` | `.env` | Path to dot-env files |
| `dotenv_overload` | `false` | Override with `.env.local` |
| `use_putenv` | - | Use `putenv()` for env vars |
| `prod_envs` | `["prod"]` | Production environment names |
| `test_envs` | `["test"]` | Test environment names |

### Generic/Symfony Runtime Options

| Option | Default | Description |
|--------|---------|-------------|
| `debug` | `APP_DEBUG` or `true` | Enable debug mode |
| `runtimes` | - | Map application types to runners |
| `error_handler` | `BasicErrorHandler` or `SymfonyErrorHandler` | Error handling class |
| `env_var_name` | `"APP_ENV"` | Environment variable name |
| `debug_var_name` | `"APP_DEBUG"` | Debug variable name |

## Creating Custom Runtimes

### Basic Structure

A custom runtime extends the resolution and execution process:

```php
<?php

namespace App\Runtime;

use Symfony\Component\Runtime\GenericRuntime;
use Symfony\Component\Runtime\RunnerInterface;
use Psr\Http\Server\RequestHandlerInterface;

class CustomRuntime extends GenericRuntime
{
    private int $port;

    public function __construct(array $options)
    {
        $this->port = $options['port'] ?? 8080;
        parent::__construct($options);
    }

    public function getRunner(?object $application): RunnerInterface
    {
        if ($application instanceof RequestHandlerInterface) {
            return new CustomRunner($application, $this->port);
        }
        return parent::getRunner($application);
    }
}
```

### Custom Runner Implementation

```php
<?php

namespace App\Runtime;

use Symfony\Component\Runtime\RunnerInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Psr\Http\Message\ServerRequestInterface;

class CustomRunner implements RunnerInterface
{
    public function __construct(
        private RequestHandlerInterface $application,
        private int $port,
    ) {
    }

    public function run(): int
    {
        // Custom application execution logic
        // Setup server, event loop, etc.
        return 0;
    }
}
```

### Using Custom Runtime

Configure in `composer.json`:

```json
{
    "extra": {
        "runtime": {
            "class": "App\\Runtime\\CustomRuntime",
            "port": 8080
        }
    }
}
```

## Key Interfaces

### RuntimeInterface

Manages resolver and runner instantiation. Primary methods:

- `getResolver(callable $callable, ?ReflectionFunction $reflector = null): ResolverInterface`
- `getRunner(?object $application): RunnerInterface`

### ResolverInterface

Resolves callable arguments. Primary method:

- `resolve(): array` - Returns `[callable, array $arguments]`

### RunnerInterface

Executes the application and returns exit code. Primary method:

- `run(): int` - Executes the application and returns exit status code

## Third-Party Runtimes

The Runtime component enables integration with various PHP runtimes:

- **Swoole**: High-performance async networking framework
- **RoadRunner**: High-performance PHP application server
- **FrankenPHP**: Modern PHP app server built in Go
- **ReactPHP**: Event-driven, non-blocking I/O
- **PHP-PM**: Process manager for PHP applications

Each runtime typically provides its own Runtime class that extends `GenericRuntime` or `SymfonyRuntime`.

## Important Notes

- Do not use Composer with `--no-plugins` or `--no-scripts` options (except Composer >= 2.1.3 with `--no-scripts`)
- Front-controller scripts should not produce side effects as they are included twice
- Run `composer dump-autoload` after modifying runtime configuration in `composer.json`
- The Runtime component is automatically configured when using Symfony Flex
