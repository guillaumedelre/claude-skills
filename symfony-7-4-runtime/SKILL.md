---
name: "symfony-7-4-runtime"
description: "Symfony Runtime component documentation. Triggers on: Runtime, application bootstrapping, runtime resolver, Swoole, RoadRunner, FrankenPHP, GenericRuntime, SymfonyRuntime, RuntimeInterface, ResolverInterface, RunnerInterface, autoload_runtime, front-controller, PHP runtime, async runtime, event loop runtime."
---

# Symfony Runtime Component

## Overview

The Symfony Runtime Component decouples application bootstrapping logic from global state, enabling applications to run with alternative PHP runtimes like PHP-PM, ReactPHP, Swoole, RoadRunner, and FrankenPHP without requiring code changes to the application itself.

## Quick Reference

### Installation

```bash
composer require symfony/runtime
```

### Basic Front-Controller (public/index.php)

```php
<?php

use App\Kernel;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return function (array $context): Kernel {
    return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
};
```

### Console Application (bin/console)

```php
<?php

use App\Kernel;
use Symfony\Bundle\FrameworkBundle\Console\Application;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return function (array $context): Application {
    $kernel = new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
    return new Application($kernel);
};
```

### Available Runtimes

| Runtime | Description |
|---------|-------------|
| `SymfonyRuntime` | Default for Symfony apps, optimized for PHP-FPM |
| `GenericRuntime` | Platform-agnostic using PHP superglobals |
| Custom runtimes | Extend for Swoole, RoadRunner, FrankenPHP, etc. |

### Configuring Runtime (composer.json)

```json
{
    "extra": {
        "runtime": {
            "class": "Symfony\\Component\\Runtime\\GenericRuntime",
            "project_dir": "/var/task"
        }
    }
}
```

### Resolvable Arguments

The closure can type-hint these arguments:

```php
return function (
    \Symfony\Component\HttpFoundation\Request $request,
    \Symfony\Component\Console\Input\InputInterface $input,
    \Symfony\Component\Console\Output\OutputInterface $output,
    array $context,  // $_SERVER + $_ENV combined
    array $argv,     // CLI arguments
): YourApplication {
    // Build and return your application
};
```

### Key Interfaces

- **RuntimeInterface**: Manages resolver and runner instantiation
- **ResolverInterface**: Resolves callable arguments
- **RunnerInterface**: Executes the application and returns exit code

## Full Documentation
- Docs: https://symfony.com/doc/7.4/components/runtime.html
- **GitHub Repository**: https://github.com/symfony/runtime

## Full Documentation

See [references/runtime.md](references/runtime.md) for complete documentation including:

- Runtime lifecycle (6 steps)
- All configuration options
- Creating custom runtimes
- Resolvable application types
- Environment variables
- Custom runner implementation
