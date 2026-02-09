# Symfony PSR-7 Bridge - Complete Documentation

## Overview

The PSR-7 Bridge component provides seamless conversion between Symfony's HttpFoundation objects and PSR-7 standard HTTP message interfaces. This enables interoperability with PSR-7 compliant libraries and middleware.

## What is PSR-7?

PSR-7 (PHP Standard Recommendation 7) defines common interfaces for representing HTTP messages as described in RFC 7230 and RFC 7231. The bridge allows Symfony applications to work with:

- **PSR-7**: HTTP message interfaces
- **PSR-17**: HTTP message factory interfaces

## Installation

### Install the Bridge

```bash
composer require symfony/psr-http-message-bridge
```

### Install a PSR-7/PSR-17 Implementation

The bridge requires a concrete implementation of PSR-7 and PSR-17 interfaces. The recommended implementation is `nyholm/psr7`:

```bash
composer require nyholm/psr7
```

Other compatible implementations include:
- `guzzlehttp/psr7` with `http-interop/http-factory-guzzle`
- `laminas/laminas-diactoros`
- `slim/psr7`

## Core Concepts

### Factory Classes

The bridge provides two main factory classes:

| Factory Class | Interface | Purpose |
|---------------|-----------|---------|
| `PsrHttpFactory` | `HttpMessageFactoryInterface` | Converts HttpFoundation to PSR-7 |
| `HttpFoundationFactory` | `HttpFoundationFactoryInterface` | Converts PSR-7 to HttpFoundation |

### Key Interfaces

```php
namespace Symfony\Bridge\PsrHttpMessage;

interface HttpMessageFactoryInterface
{
    public function createRequest(Request $symfonyRequest): ServerRequestInterface;
    public function createResponse(Response $symfonyResponse): ResponseInterface;
}

interface HttpFoundationFactoryInterface
{
    public function createRequest(ServerRequestInterface $psrRequest, bool $streamed = false): Request;
    public function createResponse(ResponseInterface $psrResponse, bool $streamed = false): Response;
}
```

## Converting HttpFoundation to PSR-7

### Setup PsrHttpFactory

The `PsrHttpFactory` constructor requires four PSR-17 factory instances:

```php
use Nyholm\Psr7\Factory\Psr17Factory;
use Symfony\Bridge\PsrHttpMessage\Factory\PsrHttpFactory;

$psr17Factory = new Psr17Factory();

$psrHttpFactory = new PsrHttpFactory(
    $psr17Factory,  // ServerRequestFactoryInterface
    $psr17Factory,  // StreamFactoryInterface
    $psr17Factory,  // UploadedFileFactoryInterface
    $psr17Factory   // ResponseFactoryInterface
);
```

### Convert Request

```php
use Symfony\Component\HttpFoundation\Request;

$symfonyRequest = new Request(
    [],                              // query parameters
    [],                              // request parameters
    [],                              // attributes
    [],                              // cookies
    [],                              // files
    ['HTTP_HOST' => 'example.com'],  // server parameters
    'Request body content'           // content
);

// Convert to PSR-7 ServerRequestInterface
$psrRequest = $psrHttpFactory->createRequest($symfonyRequest);
```

**Important**: The `HTTP_HOST` server key must be set to avoid errors during conversion.

### Convert Response

```php
use Symfony\Component\HttpFoundation\Response;

$symfonyResponse = new Response(
    'Response content',
    200,
    ['Content-Type' => 'text/html']
);

// Convert to PSR-7 ResponseInterface
$psrResponse = $psrHttpFactory->createResponse($symfonyResponse);
```

### Convert StreamedResponse

```php
use Symfony\Component\HttpFoundation\StreamedResponse;

$symfonyResponse = new StreamedResponse(function () {
    echo 'Streamed content';
});

$psrResponse = $psrHttpFactory->createResponse($symfonyResponse);
```

### Convert BinaryFileResponse

```php
use Symfony\Component\HttpFoundation\BinaryFileResponse;

$symfonyResponse = new BinaryFileResponse('/path/to/file.pdf');

$psrResponse = $psrHttpFactory->createResponse($symfonyResponse);
```

## Converting PSR-7 to HttpFoundation

### Setup HttpFoundationFactory

```php
use Symfony\Bridge\PsrHttpMessage\Factory\HttpFoundationFactory;

$httpFoundationFactory = new HttpFoundationFactory();
```

### Convert Request

```php
// $psrRequest is an instance of Psr\Http\Message\ServerRequestInterface
$symfonyRequest = $httpFoundationFactory->createRequest($psrRequest);

// For streamed requests (preserves stream instead of reading content)
$symfonyRequest = $httpFoundationFactory->createRequest($psrRequest, true);
```

### Convert Response

```php
// $psrResponse is an instance of Psr\Http\Message\ResponseInterface
$symfonyResponse = $httpFoundationFactory->createResponse($psrResponse);

// For streamed responses
$symfonyResponse = $httpFoundationFactory->createResponse($psrResponse, true);
```

## Symfony Framework Integration

### Argument Value Resolver

When using the full Symfony framework, the bridge provides automatic argument resolution for controller actions:

```php
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\RequestInterface;
use Psr\Http\Message\MessageInterface;

class MyController
{
    public function index(ServerRequestInterface $request)
    {
        // $request is automatically converted from HttpFoundation Request
        return new Response('OK');
    }
}
```

### Event Listener

The bridge includes an event listener that can automatically convert responses:

```yaml
# config/services.yaml
services:
    Symfony\Bridge\PsrHttpMessage\EventListener\PsrResponseListener:
        tags:
            - { name: kernel.event_listener, event: kernel.view, priority: -128 }
```

## Common Use Cases

### Integrating PSR-15 Middleware

```php
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;

class SymfonyPsr15Adapter
{
    private PsrHttpFactory $psrHttpFactory;
    private HttpFoundationFactory $httpFoundationFactory;
    private MiddlewareInterface $middleware;

    public function handle(Request $symfonyRequest): Response
    {
        $psrRequest = $this->psrHttpFactory->createRequest($symfonyRequest);
        $psrResponse = $this->middleware->process($psrRequest, $this->handler);

        return $this->httpFoundationFactory->createResponse($psrResponse);
    }
}
```

### Working with Guzzle

```php
use GuzzleHttp\Client;
use GuzzleHttp\Psr7\Request as GuzzleRequest;

// Convert Symfony request to Guzzle request
$psrRequest = $psrHttpFactory->createRequest($symfonyRequest);
$guzzleRequest = new GuzzleRequest(
    $psrRequest->getMethod(),
    $psrRequest->getUri(),
    $psrRequest->getHeaders(),
    $psrRequest->getBody()
);

$client = new Client();
$psrResponse = $client->send($guzzleRequest);

// Convert back to Symfony response
$symfonyResponse = $httpFoundationFactory->createResponse($psrResponse);
```

## Handling File Uploads

### HttpFoundation to PSR-7

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\File\UploadedFile;

$symfonyRequest = new Request(
    [], [], [], [],
    ['file' => new UploadedFile('/tmp/upload.txt', 'original.txt')],
    ['HTTP_HOST' => 'example.com']
);

$psrRequest = $psrHttpFactory->createRequest($symfonyRequest);
$uploadedFiles = $psrRequest->getUploadedFiles();
// Returns array of Psr\Http\Message\UploadedFileInterface
```

### PSR-7 to HttpFoundation

```php
// PSR-7 uploaded files are converted to Symfony UploadedFile instances
$symfonyRequest = $httpFoundationFactory->createRequest($psrRequest);
$files = $symfonyRequest->files->all();
// Returns array of Symfony\Component\HttpFoundation\File\UploadedFile
```

## Error Handling

### Common Errors

**Missing HTTP_HOST**:
```php
// This will cause an error
$request = new Request();
$psrRequest = $psrHttpFactory->createRequest($request); // Error!

// Always set HTTP_HOST
$request = new Request([], [], [], [], [], ['HTTP_HOST' => 'localhost']);
$psrRequest = $psrHttpFactory->createRequest($request); // OK
```

**Invalid Stream**:
```php
// Ensure response body is readable
$psrResponse = $psrResponse->withBody($streamFactory->createStream('content'));
$symfonyResponse = $httpFoundationFactory->createResponse($psrResponse);
```

## Performance Considerations

1. **Avoid Unnecessary Conversions**: Each conversion has overhead. Convert only when necessary.

2. **Use Streamed Mode for Large Bodies**: For large request/response bodies, use the streamed parameter:
   ```php
   $httpFoundationFactory->createRequest($psrRequest, true);  // streamed
   $httpFoundationFactory->createResponse($psrResponse, true); // streamed
   ```

3. **Reuse Factory Instances**: Create factory instances once and reuse them:
   ```php
   // Good: Reuse factory
   private PsrHttpFactory $factory;

   // Bad: Create new factory each time
   $factory = new PsrHttpFactory(...);
   ```

## Related Resources

- [PSR-7 Specification](https://www.php-fig.org/psr/psr-7/)
- [PSR-17 Specification](https://www.php-fig.org/psr/psr-17/)
- [Symfony HttpFoundation](https://symfony.com/doc/7.4/components/http_foundation.html)
- [Nyholm PSR-7 Implementation](https://github.com/Nyholm/psr7)

## Links

- **Documentation**: https://symfony.com/doc/7.4/components/psr7.html
- **GitHub Repository**: https://github.com/symfony/psr-http-message-bridge
- **Latest Release**: v8.0.4 (January 2026)
- **License**: MIT
