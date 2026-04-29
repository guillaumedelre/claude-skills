---
name: "symfony-7-4-http-client"
description: "Symfony 7.4 HttpClient component reference for performing HTTP requests with synchronous and asynchronous APIs. Use when making outbound HTTP calls, configuring scoped clients, retries, mocking, streaming, or PSR-18/PSR-7 interop. Triggers on: HttpClient, HttpClientInterface, HttpClient::create, ScopingHttpClient, RetryableHttpClient, MockHttpClient, MockResponse, ResponseInterface, ChunkInterface, AsyncResponse, NativeHttpClient, CurlHttpClient, AmpHttpClient, EventSourceHttpClient, ThrottlingHttpClient, NoPrivateNetworkHttpClient, TraceableHttpClient, HttpOptions, base_uri, query, headers, json, body, auth_basic, auth_bearer, scoped_clients, http_client.yaml, retry_failed, framework http_client, Server-Sent Events, PSR-18, Psr18Client, HttplugClient, TransportException, ClientException, ServerException, RedirectionException, DecodingException, retry strategy, GenericRetryStrategy."
---

# Symfony 7.4 HttpClient Component

GitHub: https://github.com/symfony/http-client
Docs: https://symfony.com/doc/7.4/http_client.html

## Installation

```bash
composer require symfony/http-client
```

For PSR-18 / PSR-7 / HTTPlug interop:

```bash
composer require nyholm/psr7
```

## Quick Reference

### Basic Request

```php
use Symfony\Contracts\HttpClient\HttpClientInterface;

class GitHubClient
{
    public function __construct(private HttpClientInterface $client) {}

    public function getRepo(string $owner, string $repo): array
    {
        $response = $this->client->request('GET', "https://api.github.com/repos/$owner/$repo");
        return $response->toArray();   // Decodes JSON
    }
}
```

### Methods

```php
$client->request('GET',  $url);
$client->request('POST', $url, ['json' => ['name' => 'Alice']]);
$client->request('PUT',  $url, ['body' => $multipartBody]);
$client->request('DELETE', $url);
```

### Common Options

```php
$response = $client->request('POST', '/api/users', [
    'headers' => ['X-Trace-Id' => $traceId],
    'query'   => ['limit' => 10],
    'json'    => ['name' => 'Alice'],          // Sets Content-Type: application/json
    'body'    => $rawBodyOrIterableOrCallable, // Mutually exclusive with json
    'auth_basic' => ['user', 'pass'],
    'auth_bearer' => $token,
    'timeout' => 5.0,
    'max_duration' => 30.0,
    'verify_peer' => true,
    'verify_host' => true,
    'cafile' => '/etc/ssl/cert.pem',
    'on_progress' => function (int $dlNow, int $dlSize, array $info) { /* ... */ },
    'user_data' => ['request_id' => $id],      // Available later via $response->getInfo('user_data')
]);
```

### Reading Responses

```php
$response = $client->request('GET', $url);

$status   = $response->getStatusCode();   // Throws on transport errors
$headers  = $response->getHeaders();      // Throws on 3xx/4xx/5xx by default
$content  = $response->getContent();      // Full body (string)
$array    = $response->toArray();         // JSON decode
$info     = $response->getInfo();         // Curl/transport info
$canceled = $response->cancel();
```

By default, `getHeaders()` and `toArray()/getContent()` throw on non-2xx. Pass `false` to suppress:

```php
$response->getHeaders(throw: false);
$response->getContent(throw: false);
```

### Streaming Responses

```php
foreach ($client->stream($response) as $chunk) {
    if ($chunk->isTimeout()) continue;
    if ($chunk->isFirst()) { $status = $response->getStatusCode(); continue; }
    if ($chunk->isLast()) break;
    file_put_contents('/tmp/file', $chunk->getContent(), \FILE_APPEND);
}
```

### Concurrent Requests

```php
$responses = [];
foreach ($urls as $url) {
    $responses[] = $client->request('GET', $url);
}

foreach ($client->stream($responses) as $response => $chunk) {
    if ($chunk->isLast()) {
        echo $response->getContent();
    }
}
```

### Scoped Clients

```yaml
framework:
    http_client:
        scoped_clients:
            github.client:
                base_uri: 'https://api.github.com'
                headers: { Accept: 'application/vnd.github+json' }
                auth_bearer: '%env(GITHUB_TOKEN)%'
```

Inject by autowire alias:

```php
public function __construct(
    private readonly HttpClientInterface $githubClient,   // ← matches scoped_clients key
) {}
```

The variable name matters (camelCase from `github.client` becomes `$githubClient`).

### Retries

```yaml
framework:
    http_client:
        scoped_clients:
            api.client:
                base_uri: 'https://api.example.com'
                retry_failed:
                    max_retries: 3
                    delay: 500
                    multiplier: 2
                    max_delay: 5000
                    jitter: 0.1
                    http_codes:
                        0: ['GET', 'HEAD']         # Connection errors
                        429: true
                        500: ['GET', 'HEAD', 'PUT']
                        503: true
```

Programmatic:

```php
use Symfony\Component\HttpClient\RetryableHttpClient;
use Symfony\Component\HttpClient\Retry\GenericRetryStrategy;

$client = new RetryableHttpClient(
    HttpClient::create(),
    new GenericRetryStrategy([429, 500, 502, 503, 504], delayMs: 500, multiplier: 2.0, maxDelayMs: 5000)
);
```

### Mocking (Testing)

```php
use Symfony\Component\HttpClient\MockHttpClient;
use Symfony\Component\HttpClient\Response\MockResponse;

$mock = new MockHttpClient([
    new MockResponse('{"id":42}', ['response_headers' => ['Content-Type: application/json']]),
    new MockResponse('Created', ['http_code' => 201]),
]);

$response = $mock->request('POST', '/users', ['json' => ['name' => 'Alice']]);
$this->assertSame(['id' => 42], $response->toArray());

$this->assertSame(1, $mock->getRequestsCount());
$this->assertSame('POST', $mock->getRequest(0)->getMethod());
```

`MockResponse` factories:
- `MockResponse::fromFile($path)` for fixtures.
- Pass a callable to `MockHttpClient` for dynamic responses.

### PSR-18 Adapter

```php
use Symfony\Component\HttpClient\Psr18Client;

$psr18 = new Psr18Client();   // Wraps default HttpClient + nyholm/psr7
$psr7Response = $psr18->sendRequest($psr7Request);
```

### Server-Sent Events

```php
use Symfony\Component\HttpClient\EventSourceHttpClient;

$client = new EventSourceHttpClient(HttpClient::create());
$response = $client->connect('https://example.com/events');

foreach ($client->stream($response) as $chunk) {
    if (!$chunk instanceof ServerSentEvent) continue;
    echo $chunk->getType(), ': ', $chunk->getData(), \PHP_EOL;
}
```

### Decorators

| Decorator | Purpose |
|-----------|---------|
| `RetryableHttpClient` | Retry on configured codes |
| `ScopingHttpClient` | Apply options based on URL prefix |
| `ThrottlingHttpClient` | Rate-limit using `RateLimiter` |
| `NoPrivateNetworkHttpClient` | Block RFC1918/loopback (SSRF protection) |
| `TraceableHttpClient` | Captures requests for profiler |
| `LoggableHttpClient` | Wraps any client with PSR-3 logging |
| `EventSourceHttpClient` | SSE support |
| `AsyncDecoratorTrait` | Build custom decorators |

### Exception Hierarchy

```
HttpClient\Exception\TransportExceptionInterface       (network errors)
HttpClient\Exception\HttpExceptionInterface            (HTTP error responses)
  ├── ClientExceptionInterface     (4xx)
  ├── ServerExceptionInterface     (5xx)
  └── RedirectionExceptionInterface (3xx; only if max_redirects=0)
HttpClient\Exception\DecodingExceptionInterface        (toArray on non-JSON)
HttpClient\Exception\TimeoutExceptionInterface
```

```php
try {
    $data = $response->toArray();
} catch (ClientExceptionInterface $e) {
    // 4xx
} catch (ServerExceptionInterface $e) {
    // 5xx
} catch (TransportExceptionInterface $e) {
    // DNS/SSL/connection
}
```

## Full Documentation

For all options exhaustively, custom decorators, framework config, downloading/uploading large files, HTTP/2/3, proxies, and profiler integration: see [references/http-client.md](references/http-client.md).
