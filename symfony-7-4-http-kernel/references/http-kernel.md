# Symfony 7.4 HttpKernel -- Complete Reference

## Component Overview

The HttpKernel component converts a `Request` into a `Response` through an event-driven lifecycle. It is the foundation of every Symfony application.

**Install:** `composer require symfony/http-kernel`

---

## Core Interface: HttpKernelInterface

```php
namespace Symfony\Component\HttpKernel;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

interface HttpKernelInterface
{
    const MAIN_REQUEST = 1;
    const SUB_REQUEST = 2;

    public function handle(
        Request $request,
        int $type = self::MAIN_REQUEST,
        bool $catch = true
    ): Response;
}
```

## Core Interface: KernelInterface

Extends `HttpKernelInterface`. Adds:

- `boot(): void` -- Boot the kernel (load bundles, build container).
- `shutdown(): void` -- Shutdown the kernel.
- `getBundles(): array` -- Returns registered bundles.
- `getBundle(string $name): BundleInterface`
- `locateResource(string $name): string` -- Resolve `@BundleName/path` to filesystem path.
- `getEnvironment(): string`
- `isDebug(): bool`
- `getProjectDir(): string`
- `getContainer(): ContainerInterface`
- `getStartTime(): float`
- `getCacheDir(): string`
- `getBuildDir(): string`
- `getLogDir(): string`
- `getCharset(): string`

## Core Interface: TerminableInterface

```php
interface TerminableInterface
{
    public function terminate(Request $request, Response $response): void;
}
```

Called after `$response->send()`. Only effective with PHP-FPM or FrankenPHP where the response is flushed before terminate listeners execute.

## Core Interface: RebootableInterface

```php
interface RebootableInterface
{
    public function reboot(?string $warmupDir): void;
}
```

---

## HttpKernel Class

`Symfony\Component\HttpKernel\HttpKernel` implements `HttpKernelInterface` and `TerminableInterface`.

**Constructor:**

```php
public function __construct(
    EventDispatcherInterface $dispatcher,
    ControllerResolverInterface $resolver,
    RequestStack $requestStack,
    ArgumentResolverInterface $argumentResolver,
    bool $handleAllThrowables = false
)
```

**Key method:** `handle(Request $request, int $type, bool $catch): Response`

Internally calls (in order):
1. Dispatch `KernelEvents::REQUEST`
2. `$this->resolver->getController($request)`
3. Dispatch `KernelEvents::CONTROLLER`
4. `$this->argumentResolver->getArguments($request, $controller)`
5. Dispatch `KernelEvents::CONTROLLER_ARGUMENTS`
6. Call controller with arguments
7. If result is not a Response: dispatch `KernelEvents::VIEW`
8. Dispatch `KernelEvents::RESPONSE`
9. Dispatch `KernelEvents::FINISH_REQUEST`
10. Return Response

On exception (if `$catch` is true): dispatch `KernelEvents::EXCEPTION`.

---

## Kernel Events -- Full Detail

### kernel.request

| Property | Value |
|---|---|
| Constant | `KernelEvents::REQUEST` |
| Event class | `Symfony\Component\HttpKernel\Event\RequestEvent` |
| Priority | Highest priority listeners run first |

**Purpose:** Initialize system, add request info, return early Response (security, redirects, maintenance mode).

**Key methods on RequestEvent:**
- `getRequest(): Request`
- `getRequestType(): int`
- `isMainRequest(): bool`
- `setResponse(Response $response)` -- sets response and skips to `kernel.response`
- `getResponse(): ?Response`
- `hasResponse(): bool`

**Important:** If a listener calls `setResponse()`, all remaining `kernel.request` listeners still execute (unless `stopPropagation()` is called), but the controller is never called. Processing jumps directly to `kernel.response`.

**Typical listeners (Symfony):**
- `RouterListener` (priority 32) -- matches route, sets `_controller` on request attributes
- `SessionListener` -- initializes session
- `LocaleListener` -- sets locale
- `FirewallListener` -- security checks

---

### kernel.controller

| Property | Value |
|---|---|
| Constant | `KernelEvents::CONTROLLER` |
| Event class | `Symfony\Component\HttpKernel\Event\ControllerEvent` |

**Purpose:** Modify or replace the controller after resolution.

**Key methods on ControllerEvent:**
- `getController(): callable`
- `setController(callable $controller)`
- `getRequest(): Request`
- `getAttributes(string $className = null): array` -- PHP 8 attributes on controller
- `isMainRequest(): bool`

---

### kernel.controller_arguments

| Property | Value |
|---|---|
| Constant | `KernelEvents::CONTROLLER_ARGUMENTS` |
| Event class | `Symfony\Component\HttpKernel\Event\ControllerArgumentsEvent` |

**Purpose:** Access or modify controller arguments after resolution.

**Key methods on ControllerArgumentsEvent:**
- `getArguments(): array`
- `setArguments(array $arguments)`
- `getNamedArguments(): array`
- `getController(): callable`
- `setController(callable $controller)`
- `getRequest(): Request`

---

### kernel.view

| Property | Value |
|---|---|
| Constant | `KernelEvents::VIEW` |
| Event class | `Symfony\Component\HttpKernel\Event\ViewEvent` |

**Purpose:** Transform a non-Response controller return value into a Response (template rendering, serialization).

Only dispatched if the controller does NOT return a Response instance.

**Key methods on ViewEvent:**
- `getControllerResult(): mixed` -- the value returned by the controller
- `setResponse(Response $response)`
- `getRequest(): Request`

If no listener sets a Response, an exception is thrown.

---

### kernel.response

| Property | Value |
|---|---|
| Constant | `KernelEvents::RESPONSE` |
| Event class | `Symfony\Component\HttpKernel\Event\ResponseEvent` |

**Purpose:** Modify the Response before it is returned (add headers, modify content, set cache headers).

**Key methods on ResponseEvent:**
- `getResponse(): Response`
- `setResponse(Response $response)`
- `getRequest(): Request`
- `isMainRequest(): bool`

**Typical listeners:**
- `ResponseListener` -- sets Content-Type
- `ProfilerListener` -- injects profiler token
- `WebDebugToolbarListener` -- injects debug toolbar
- `CacheControlListener` -- sets cache headers

---

### kernel.finish_request

| Property | Value |
|---|---|
| Constant | `KernelEvents::FINISH_REQUEST` |
| Event class | `Symfony\Component\HttpKernel\Event\FinishRequestEvent` |

**Purpose:** Reset global/request-scoped state. Dispatched after response is ready but before it is returned from `handle()`. Important for cleaning up after sub-requests.

**Typical listeners:**
- `LocaleListener` -- resets locale to parent request
- `RouterListener` -- resets routing context

---

### kernel.terminate

| Property | Value |
|---|---|
| Constant | `KernelEvents::TERMINATE` |
| Event class | `Symfony\Component\HttpKernel\Event\TerminateEvent` |

**Purpose:** Perform heavy post-response work (send emails, log analytics, process queues).

**Key methods on TerminateEvent:**
- `getRequest(): Request`
- `getResponse(): Response`

**Usage pattern:**

```php
$response = $kernel->handle($request);
$response->send();
$kernel->terminate($request, $response); // dispatches kernel.terminate
```

Only benefits from async execution with PHP-FPM or FrankenPHP.

---

### kernel.exception

| Property | Value |
|---|---|
| Constant | `KernelEvents::EXCEPTION` |
| Event class | `Symfony\Component\HttpKernel\Event\ExceptionEvent` |

**Purpose:** Handle exceptions and create error Responses.

**Key methods on ExceptionEvent:**
- `getThrowable(): \Throwable`
- `setThrowable(\Throwable $exception)` -- replace/wrap the exception
- `setResponse(Response $response)` -- provide an error response
- `getResponse(): ?Response`
- `hasResponse(): bool`
- `isKernelTerminating(): bool` -- (Symfony 7.1+)
- `isAllowingCustomResponseCode(): bool`
- `allowCustomResponseCode()`
- `getRequest(): Request`

If no listener sets a Response, the exception is re-thrown.

**Built-in listeners:**
- `ErrorListener` -- converts exception to `FlattenException`, invokes error controller
- Security `ExceptionListener` -- handles access denied, redirects to login

---

## Controller Resolution

### ControllerResolverInterface

```php
namespace Symfony\Component\HttpKernel\Controller;

interface ControllerResolverInterface
{
    public function getController(Request $request): callable|false;
}
```

Reads `_controller` from `$request->attributes`. Supports:
- `App\Controller\FooController::bar` (class::method)
- `App\Controller\FooController` (invokable `__invoke`)
- Closures / callables
- `service_id:method` (with service container)

### ArgumentResolverInterface

```php
interface ArgumentResolverInterface
{
    public function getArguments(Request $request, callable $controller): array;
}
```

Uses a chain of `ValueResolverInterface` implementations:
- `RequestAttributeValueResolver` -- matches argument name to `$request->attributes`
- `RequestValueResolver` -- injects `Request` for type-hinted arguments
- `SessionValueResolver` -- injects `SessionInterface`
- `DefaultValueResolver` -- uses parameter default values
- `VariadicValueResolver` -- handles variadic parameters

### ValueResolverInterface

```php
namespace Symfony\Component\HttpKernel\Controller;

interface ValueResolverInterface
{
    public function resolve(Request $request, ArgumentMetadata $metadata): iterable;
}
```

---

## Sub-Requests

Sub-requests are internal requests processed by the same kernel. Used for ESI fragments, `{{ render() }}` in Twig, and embedding controllers.

```php
use Symfony\Component\HttpKernel\HttpKernelInterface;

$subRequest = Request::create('/internal/sidebar');
// or manually:
$subRequest = new Request();
$subRequest->attributes->set('_controller', 'App\\Controller\\SidebarController::index');

$response = $kernel->handle($subRequest, HttpKernelInterface::SUB_REQUEST);
```

**Important:** In event listeners, always check `$event->isMainRequest()` to avoid running main-request-only logic during sub-requests.

---

## Key Exception Classes

Located in `Symfony\Component\HttpKernel\Exception\`:

| Class | HTTP Status |
|---|---|
| `BadRequestHttpException` | 400 |
| `UnauthorizedHttpException` | 401 |
| `AccessDeniedHttpException` | 403 |
| `NotFoundHttpException` | 404 |
| `MethodNotAllowedHttpException` | 405 |
| `ConflictHttpException` | 409 |
| `GoneHttpException` | 410 |
| `UnprocessableEntityHttpException` | 422 |
| `TooManyRequestsHttpException` | 429 |
| `ServiceUnavailableHttpException` | 503 |
| `HttpException` | Any (custom status code) |

All implement `HttpExceptionInterface` which provides `getStatusCode()` and `getHeaders()`.

---

## Built-in Event Listeners

| Listener | Event | Purpose |
|---|---|---|
| `RouterListener` | kernel.request | Route matching |
| `SessionListener` | kernel.request / kernel.response | Session management |
| `LocaleListener` | kernel.request / kernel.finish_request | Locale handling |
| `ValidateRequestListener` | kernel.request | Request validation |
| `ErrorListener` | kernel.exception | Exception-to-response conversion |
| `ResponseListener` | kernel.response | Content-Type setting |
| `SurrogateListener` | kernel.response | ESI/SSI support |
| `ProfilerListener` | kernel.response / kernel.terminate | Profiler data collection |
| `DisallowRobotsIndexingListener` | kernel.response | X-Robots-Tag header |
| `CacheAttributeListener` | kernel.controller_arguments / kernel.response | `#[Cache]` attribute |

---

## Full Working Example

```php
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Controller\ArgumentResolver;
use Symfony\Component\HttpKernel\Controller\ControllerResolver;
use Symfony\Component\HttpKernel\EventListener\RouterListener;
use Symfony\Component\HttpKernel\HttpKernel;
use Symfony\Component\Routing\Matcher\UrlMatcher;
use Symfony\Component\Routing\RequestContext;
use Symfony\Component\Routing\Route;
use Symfony\Component\Routing\RouteCollection;

$routes = new RouteCollection();
$routes->add('hello', new Route('/hello/{name}', [
    '_controller' => function (Request $request): Response {
        return new Response(sprintf('Hello %s', $request->attributes->get('name')));
    },
]));

$request = Request::createFromGlobals();
$matcher = new UrlMatcher($routes, new RequestContext());

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new RouterListener($matcher, new RequestStack()));

$kernel = new HttpKernel(
    $dispatcher,
    new ControllerResolver(),
    new RequestStack(),
    new ArgumentResolver()
);

$response = $kernel->handle($request);
$response->send();
$kernel->terminate($request, $response);
```
