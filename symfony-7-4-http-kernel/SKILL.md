---
name: "symfony-7-4-http-kernel"
description: "Symfony 7.4 HttpKernel component reference. Use this skill when working with: HttpKernel, request handling, kernel events (kernel.request, kernel.controller, kernel.view, kernel.response, kernel.terminate, kernel.exception), controller resolver, argument resolver, exception handling, sub-requests, KernelInterface, HttpKernelInterface, TerminableInterface, RequestEvent, ControllerEvent, ViewEvent, ResponseEvent, ExceptionEvent, TerminateEvent."
---

# Symfony 7.4 HttpKernel Component

## Overview

The HttpKernel component provides a structured process for converting a `Request` into a `Response` using the EventDispatcher component. It is the core of the Symfony framework, orchestrating controller resolution, argument resolution, and a rich event-driven lifecycle.

## Quick Reference

### Request Lifecycle (in order)

1. `kernel.request` (`KernelEvents::REQUEST` / `RequestEvent`) -- early return possible
2. Controller resolution via `ControllerResolverInterface`
3. `kernel.controller` (`KernelEvents::CONTROLLER` / `ControllerEvent`)
4. Argument resolution via `ArgumentResolverInterface`
5. `kernel.controller_arguments` (`KernelEvents::CONTROLLER_ARGUMENTS` / `ControllerArgumentsEvent`)
6. Controller execution
7. `kernel.view` (`KernelEvents::VIEW` / `ViewEvent`) -- only if controller did not return a Response
8. `kernel.response` (`KernelEvents::RESPONSE` / `ResponseEvent`)
9. `kernel.finish_request` (`KernelEvents::FINISH_REQUEST` / `FinishRequestEvent`)
10. `kernel.terminate` (`KernelEvents::TERMINATE` / `TerminateEvent`) -- after response is sent

Exception at any point triggers `kernel.exception` (`KernelEvents::EXCEPTION` / `ExceptionEvent`).

### Core Interfaces

| Interface | Purpose |
|---|---|
| `HttpKernelInterface` | `handle(Request, type, catch): Response` |
| `KernelInterface` | Boot, bundles, container, environment |
| `TerminableInterface` | `terminate(Request, Response)` |
| `RebootableInterface` | `reboot(?cacheDir)` |
| `ControllerResolverInterface` | `getController(Request): callable\|false` |
| `ArgumentResolverInterface` | `getArguments(Request, callable): array` |
| `ValueResolverInterface` | Custom argument value resolution |

### Request Types

```php
HttpKernelInterface::MAIN_REQUEST  // Main browser request
HttpKernelInterface::SUB_REQUEST   // Internal sub-request (e.g. ESI, fragment)
```

### Sub-Requests

```php
$subRequest = new Request();
$subRequest->attributes->set('_controller', 'App\\Controller\\FooController::bar');
$response = $kernel->handle($subRequest, HttpKernelInterface::SUB_REQUEST);
```

Check request type in listeners: `$event->isMainRequest()`.

### Creating an Event Listener

```php
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

public function onKernelRequest(RequestEvent $event): void
{
    if (!$event->isMainRequest()) {
        return;
    }
    // ...
}
```

## Full Documentation
- Official docs: https://symfony.com/doc/7.4/components/http_kernel.html
- GitHub: https://github.com/symfony/http-kernel
- KernelEvents constants: `Symfony\Component\HttpKernel\KernelEvents`

## Full Documentation
See [references/http-kernel.md](references/http-kernel.md) for complete documentation including all events, classes, interfaces, and usage examples.
