# Symfony 7.4 HttpClient - Complete Reference

## Table of Contents

- [Engine Selection](#engine-selection)
- [Options Reference](#options-reference)
- [Response Interface](#response-interface)
- [Streaming](#streaming)
- [Uploads](#uploads)
- [Downloads](#downloads)
- [Scoped Clients](#scoped-clients)
- [Retries](#retries)
- [Throttling](#throttling)
- [Mocking](#mocking)
- [PSR Interop](#psr-interop)
- [Server-Sent Events](#server-sent-events)
- [Profiler](#profiler)
- [SSRF Protection](#ssrf-protection)
- [HTTP/2 and HTTP/3](#http2-and-http3)
- [Proxies](#proxies)
- [Custom Decorators](#custom-decorators)

---

## Engine Selection

`HttpClient::create()` picks an engine in this order:

1. `CurlHttpClient` if `ext-curl` is loaded (HTTP/2 if libcurl ≥ 7.36).
2. `NativeHttpClient` (PHP streams) otherwise.

`AmpHttpClient` (requires `amphp/http-client`) is fully async and supports HTTP/2 multiplexing on shared sockets.

Force engine:

```php
use Symfony\Component\HttpClient\CurlHttpClient;
use Symfony\Component\HttpClient\NativeHttpClient;
use Symfony\Component\HttpClient\AmpHttpClient;

$client = new CurlHttpClient(['headers' => ['User-Agent' => 'X']], maxHostConnections: 6, maxPendingPushes: 50);
```

Framework config:

```yaml
framework:
    http_client:
        max_host_connections: 12
        default_options:
            timeout: 4
            max_duration: 30
            headers: { Accept: 'application/json' }
```

## Options Reference

### Auth

| Option | Type |
|--------|------|
| `auth_basic` | `string` `'user:pass'` or `[user, pass]` |
| `auth_bearer` | `string` |
| `auth_ntlm` | `string|array` (curl only) |

### Body

| Option | Type | Notes |
|--------|------|-------|
| `body` | `string|iterable|resource|Closure(int): string` | Mutually exclusive with `json`. Use closure for streaming uploads |
| `json` | `mixed` | Auto-encoded; sets `Content-Type: application/json` |
| `query` | `array` | URL query parameters |

### Network

| Option | Type | Default |
|--------|------|---------|
| `timeout` | `float` | `default_socket_timeout` ini value (60) |
| `max_duration` | `float` | `0.0` (no limit) |
| `bindto` | `string` | Local IP/interface to bind |
| `proxy` | `string` | `http://user:pass@proxy:3128` |
| `no_proxy` | `string` | Comma-separated host list |
| `resolve` | `array` | `['host' => '127.0.0.1']` overrides DNS |
| `local_cert` | `string` | TLS client cert |
| `local_pk` | `string` | TLS private key |
| `passphrase` | `string` | Cert passphrase |
| `cafile` | `string` | CA bundle path |
| `capath` | `string` | Directory of CA certs |
| `ciphers` | `string` | OpenSSL cipher list |
| `verify_peer` | `bool` | `true` |
| `verify_host` | `bool` | `true` |
| `peer_fingerprint` | `array` | `['md5' => '...', 'sha1' => '...']` |

### HTTP Behavior

| Option | Notes |
|--------|-------|
| `headers` | `array` of headers |
| `base_uri` | Resolved against relative URLs |
| `max_redirects` | Default 20; set 0 to surface 3xx as `RedirectionException` |
| `http_version` | `'1.0'`, `'1.1'`, `'2.0'`, `'3.0'` |
| `crypto_method` | `\STREAM_CRYPTO_METHOD_TLSv1_3_CLIENT` etc. |
| `buffer` | `bool` or callable returning bool from headers; controls in-memory buffering |
| `extra` | Custom array passed through to engine; check `getInfo('extra')` |

### User Land

| Option | Notes |
|--------|-------|
| `user_data` | Arbitrary data; retrieve via `$response->getInfo('user_data')` |
| `on_progress` | `function(int $dlNow, int $dlSize, array $info): void` |

## Response Interface

```php
interface ResponseInterface
{
    public function getStatusCode(): int;
    public function getHeaders(bool $throw = true): array;
    public function getContent(bool $throw = true): string;
    public function toArray(bool $throw = true): array;
    public function cancel(): void;
    public function getInfo(?string $type = null): mixed;
}
```

`getInfo()` keys (subset):
- `'http_code'`, `'http_method'`, `'response_headers'`, `'redirect_count'`, `'redirect_url'`.
- `'start_time'`, `'connect_time'`, `'pretransfer_time'`, `'starttransfer_time'`, `'total_time'`.
- `'size_download'`, `'size_upload'`, `'primary_ip'`, `'primary_port'`.
- `'user_data'`.
- `'previous_info'` (array of redirected response infos).
- `'debug'` (string when curl verbose enabled).

## Streaming

### Concurrent Pattern

```php
$responses = array_map(
    fn ($url) => $client->request('GET', $url),
    $urls,
);

foreach ($client->stream($responses, timeout: 1.0) as $response => $chunk) {
    try {
        if ($chunk->isTimeout()) continue;
        if ($chunk->isFirst()) {
            // Headers received
            continue;
        }
        if ($chunk->isLast()) {
            $body = $response->getContent();
            // Process complete
        } else {
            // Partial chunk
            echo $chunk->getContent();
        }
    } catch (\Throwable $e) {
        $response->cancel();
    }
}
```

### Cancel Pending

`$response->cancel()` aborts the in-flight request.

## Uploads

### File via stream

```php
$client->request('POST', $url, [
    'body' => fopen('/path/to/large.bin', 'r'),
    'headers' => ['Content-Type' => 'application/octet-stream'],
]);
```

### Generator (chunked)

```php
$client->request('POST', $url, [
    'body' => static function (int $size): string {
        static $fh;
        $fh ??= fopen('/path/file', 'r');
        return fread($fh, $size) ?: '';
    },
]);
```

### Multipart / form-data

Symfony HttpClient does not generate multipart bodies natively; use `symfony/mime`:

```php
use Symfony\Component\Mime\Part\Multipart\FormDataPart;
use Symfony\Component\Mime\Part\DataPart;

$formData = new FormDataPart([
    'name' => 'Alice',
    'avatar' => DataPart::fromPath('/tmp/me.jpg'),
]);

$client->request('POST', $url, [
    'headers' => $formData->getPreparedHeaders()->toArray(),
    'body' => $formData->bodyToIterable(),
]);
```

## Downloads

```php
$response = $client->request('GET', 'https://example.com/file.zip');
$fileHandle = fopen('/tmp/file.zip', 'w');

foreach ($client->stream($response) as $chunk) {
    fwrite($fileHandle, $chunk->getContent());
}
fclose($fileHandle);
```

## Scoped Clients

Programmatic:

```php
use Symfony\Component\HttpClient\ScopingHttpClient;

$client = ScopingHttpClient::forBaseUri(HttpClient::create(), 'https://api.example.com', [
    'headers' => ['Accept' => 'application/json'],
    'auth_bearer' => $token,
]);
```

Multi-scope:

```php
$client = new ScopingHttpClient(HttpClient::create(), [
    'github\.' => ['headers' => ['User-Agent' => 'mybot']],
    'api\.example\.com' => ['auth_bearer' => '%env(API_TOKEN)%'],
]);
```

Framework config: see SKILL.md. Variable name `$githubClient` ↔ scope name `github.client`.

## Retries

`GenericRetryStrategy` (default) retries on:
- `0` (transport errors).
- `423`, `425`, `429`, `500`, `502`, `503`, `504`, `507`, `510`.

Custom strategy implements `RetryStrategyInterface`:

```php
use Symfony\Contracts\HttpClient\Exception\ExceptionInterface;
use Symfony\Component\HttpClient\Retry\RetryStrategyInterface;
use Symfony\Component\HttpClient\Response\AsyncContext;

final class IdempotentRetryStrategy implements RetryStrategyInterface
{
    public function shouldRetry(AsyncContext $context, ?string $responseContent, ?ExceptionInterface $exception): ?bool
    {
        if ($exception) return true;
        return $context->getStatusCode() >= 500;
    }

    public function getDelay(AsyncContext $context, ?string $responseContent, ?ExceptionInterface $exception): int
    {
        return 1000 * (2 ** $context->getInfo('retry_count'));   // ms
    }
}
```

## Throttling

```php
use Symfony\Component\HttpClient\ThrottlingHttpClient;
use Symfony\Component\RateLimiter\RateLimiterFactory;

$throttled = new ThrottlingHttpClient(
    HttpClient::create(),
    $rateLimiterFactory->create('api')
);
```

Rate limiter config:

```yaml
framework:
    rate_limiter:
        api:
            policy: token_bucket
            limit: 10
            rate: { interval: '1 second' }
```

## Mocking

```php
$mock = new MockHttpClient(static function (string $method, string $url, array $options): MockResponse {
    return match (true) {
        str_starts_with($url, 'https://api.github.com/repos') =>
            new MockResponse('{"name":"test"}', ['http_code' => 200, 'response_headers' => ['Content-Type: application/json']]),
        default => new MockResponse('Not Found', ['http_code' => 404]),
    };
});
```

Inject `MockHttpClient` into services in tests via `self::getContainer()->set('http_client', $mock)` or `KernelTestCase::loadFromService`.

`MockResponse::fromFile()` reads:
- A file with `\r\n\r\n` separator (headers + body).
- Or a request file `<name>.client.body` + response files `<name>.txt`, `<name>.headers`, `<name>.body`.

## PSR Interop

```php
// PSR-18
use Symfony\Component\HttpClient\Psr18Client;
$psr18 = new Psr18Client(HttpClient::create());

// HTTPlug
use Symfony\Component\HttpClient\HttplugClient;
$httplug = new HttplugClient(HttpClient::create());
```

Both adapters require `nyholm/psr7` (or any PSR-17/PSR-7 lib).

## Server-Sent Events

```php
use Symfony\Component\HttpClient\EventSourceHttpClient;

$client = new EventSourceHttpClient(HttpClient::create(), reconnectionTime: 10);
$response = $client->connect('https://example.com/sse');

foreach ($client->stream($response) as $chunk) {
    if ($chunk instanceof ServerSentEvent) {
        echo "[{$chunk->getType()}] {$chunk->getData()}\n";
    }
}
```

## Profiler

`TraceableHttpClient` is auto-installed in dev/test environments. View captured calls via the Symfony Profiler (`/_profiler` toolbar → HTTP Client section).

## SSRF Protection

```php
use Symfony\Component\HttpClient\NoPrivateNetworkHttpClient;

$safeClient = new NoPrivateNetworkHttpClient(HttpClient::create(), subnets: [
    '169.254.0.0/16',     // Allow link-local for cloud metadata if needed
]);
```

Blocks RFC1918, loopback, link-local, and IPv6 private ranges.

## HTTP/2 and HTTP/3

```php
$client->request('GET', $url, ['http_version' => '2.0']);
$client->request('GET', $url, ['http_version' => '3.0']);
```

CurlHttpClient supports HTTP/2 with libcurl ≥ 7.36 (cleartext) or 7.43 (TLS). HTTP/3 requires libcurl built with QUIC support (e.g. nghttp3+ngtcp2). NativeHttpClient is HTTP/1.1 only.

## Proxies

Auto-detect via env: `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`.

Per-request:

```php
$client->request('GET', $url, [
    'proxy' => 'http://user:pass@proxy.internal:3128',
    'no_proxy' => '.example.com,127.0.0.1',
]);
```

## Custom Decorators

```php
use Symfony\Component\HttpClient\AsyncDecoratorTrait;
use Symfony\Component\HttpClient\Response\AsyncContext;
use Symfony\Contracts\HttpClient\HttpClientInterface;

final class LoggingClient implements HttpClientInterface
{
    use AsyncDecoratorTrait;

    public function request(string $method, string $url, array $options = []): ResponseInterface
    {
        $passthru = function (ChunkInterface $chunk, AsyncContext $context): \Generator {
            if ($chunk->isFirst()) {
                $this->logger->info('First chunk', ['status' => $context->getStatusCode()]);
            }
            yield $chunk;
        };

        return new AsyncResponse($this->client, $method, $url, $options, $passthru);
    }
}
```
