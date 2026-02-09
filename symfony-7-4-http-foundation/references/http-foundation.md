# Symfony 7.4 HttpFoundation - Complete Reference

## Installation

```bash
composer require symfony/http-foundation
```

---

## Request

### Creating a Request

```php
use Symfony\Component\HttpFoundation\Request;

// From PHP globals
$request = Request::createFromGlobals();

// Manual construction
$request = new Request($_GET, $_POST, [], $_COOKIE, $_FILES, $_SERVER);

// Simulated request
$request = Request::create('/hello', 'GET', ['name' => 'Fabien']);
```

### Public Properties (ParameterBag Instances)

| Property     | Equivalent  | Class          |
|-------------|-------------|----------------|
| `query`     | `$_GET`     | `InputBag`     |
| `request`   | `$_POST`    | `InputBag`     |
| `cookies`   | `$_COOKIE`  | `InputBag`     |
| `attributes`| Custom data | `ParameterBag` |
| `files`     | `$_FILES`   | `FileBag`      |
| `server`    | `$_SERVER`  | `ServerBag`    |
| `headers`   | HTTP headers| `HeaderBag`    |

### ParameterBag Methods

```php
$bag->get('key');                          // Get value
$bag->get('key', 'default');               // Get with default
$bag->all();                               // Get all
$bag->all('key');                          // Get nested array for key
$bag->has('key');                          // Check existence
$bag->set('key', 'value');                 // Set value
$bag->remove('key');                       // Remove
$bag->keys();                              // Get all keys
$bag->replace(['new' => 'values']);        // Replace all
$bag->add(['extra' => 'values']);          // Add parameters
$bag->count();                             // Count parameters
```

### InputBag Filter Methods

```php
$bag->getAlpha('key');                     // Alphabetic chars only
$bag->getAlnum('key');                     // Alphanumeric only
$bag->getDigits('key');                    // Digits only
$bag->getInt('key');                       // Integer
$bag->getBoolean('key');                   // Boolean
$bag->getString('key');                    // String
$bag->getEnum('key', StatusEnum::class);   // PHP enum
$bag->filter('key', null, FILTER_VALIDATE_EMAIL); // PHP filter_var
```

### Request Body

```php
$request->getContent();                    // Raw body string
$request->toArray();                       // JSON-decoded body
$request->getPayload();                    // InputBag from POST or JSON
$request->getPayload()->get('key');
```

### Request Identification

```php
$request->getMethod();                     // HTTP method
$request->getPathInfo();                   // URI path
$request->getUri();                        // Full URI
$request->getScheme();                     // http or https
$request->getHost();                       // Hostname
$request->getPort();                       // Port number
$request->getClientIp();                   // Client IP
$request->getLocale();                     // Locale
$request->getDefaultLocale();              // Default locale
$request->isSecure();                      // HTTPS?
$request->isXmlHttpRequest();              // AJAX?
```

### Accept Headers

```php
$request->getAcceptableContentTypes();     // Ordered by quality
$request->getLanguages();
$request->getCharsets();
$request->getEncodings();
```

### AcceptHeader Parser

```php
use Symfony\Component\HttpFoundation\AcceptHeader;

$accept = AcceptHeader::fromString($request->headers->get('Accept'));
if ($accept->has('text/html')) {
    $item = $accept->get('text/html');
    $quality = $item->getQuality();
    $charset = $item->getAttribute('charset', 'utf-8');
}
$all = $accept->all(); // Sorted by descending quality
```

### Session

```php
$session = $request->getSession();
$request->hasPreviousSession();            // Session started in previous request?
```

### Custom Request Factory

```php
Request::setFactory(function ($query, $request, $attributes, $cookies, $files, $server, $content) {
    return new CustomRequest($query, $request, $attributes, $cookies, $files, $server, $content);
});
```

### Request Duplication and Globals

```php
$copy = $request->duplicate();
$request->overrideGlobals();
```

---

## Response

### Creating Responses

```php
use Symfony\Component\HttpFoundation\Response;

$response = new Response('Content', Response::HTTP_OK, ['Content-Type' => 'text/html']);
$response->setContent('Hello World');
$response->headers->set('Content-Type', 'text/plain');
$response->setStatusCode(Response::HTTP_NOT_FOUND);
$response->setCharset('UTF-8');
```

### HTTP Status Code Constants

```php
Response::HTTP_OK                    // 200
Response::HTTP_CREATED               // 201
Response::HTTP_ACCEPTED              // 202
Response::HTTP_NO_CONTENT            // 204
Response::HTTP_MOVED_PERMANENTLY     // 301
Response::HTTP_FOUND                 // 302
Response::HTTP_NOT_MODIFIED          // 304
Response::HTTP_BAD_REQUEST           // 400
Response::HTTP_UNAUTHORIZED          // 401
Response::HTTP_FORBIDDEN             // 403
Response::HTTP_NOT_FOUND             // 404
Response::HTTP_METHOD_NOT_ALLOWED    // 405
Response::HTTP_CONFLICT              // 409
Response::HTTP_GONE                  // 410
Response::HTTP_UNPROCESSABLE_ENTITY  // 422
Response::HTTP_TOO_MANY_REQUESTS     // 429
Response::HTTP_INTERNAL_SERVER_ERROR // 500
Response::HTTP_SERVICE_UNAVAILABLE   // 503
```

### Sending

```php
$response->prepare($request);   // Fix HTTP spec issues
$response->send();              // Send to client
$response->send(flush: false);  // Without flushing
```

---

## Cookie

```php
use Symfony\Component\HttpFoundation\Cookie;

// Fluent creation
$cookie = Cookie::create('name')
    ->withValue('value')
    ->withExpires(strtotime('+1 day'))
    ->withDomain('.example.com')
    ->withPath('/')
    ->withSecure(true)
    ->withHttpOnly(true)
    ->withSameSite(Cookie::SAMESITE_LAX)     // LAX, STRICT, NONE
    ->withPartitioned();                      // CHIPS support

// From header string
$cookie = Cookie::fromString('foo=bar; Path=/; Domain=.example.com');

// Set on response
$response->headers->setCookie($cookie);
$response->headers->clearCookie('name');
```

---

## Cache Management

```php
$response->setPublic();
$response->setPrivate();
$response->setMaxAge(600);                     // Client cache seconds
$response->setSharedMaxAge(600);               // Shared/proxy cache (s-maxage)
$response->setTtl(600);                        // TTL in seconds
$response->setClientTtl(300);
$response->expire();                           // Expire immediately
$response->setExpires(new \DateTime('+1 day'));
$response->setETag('version-hash');
$response->setLastModified(new \DateTime());
$response->setVary(['Accept-Encoding', 'Accept-Language']);
$response->setStaleWhileRevalidate(60);
$response->setStaleIfError(86400);
$response->setImmutable(true);

// Bulk set
$response->setCache([
    'public'              => true,
    'max_age'             => 600,
    's_maxage'            => 600,
    'etag'                => 'abc',
    'last_modified'       => new \DateTime(),
    'immutable'           => true,
    'must_revalidate'     => false,
    'no_cache'            => false,
    'no_store'            => false,
    'no_transform'        => false,
    'proxy_revalidate'    => false,
    'stale_if_error'      => 86400,
    'stale_while_revalidate' => 60,
]);

// Conditional response (304 Not Modified)
if ($response->isNotModified($request)) {
    $response->send();
}
```

---

## JsonResponse

```php
use Symfony\Component\HttpFoundation\JsonResponse;

$response = new JsonResponse(['data' => 123]);
$response = new JsonResponse();
$response->setData(['data' => 123]);

// From raw JSON string
$response = JsonResponse::fromJsonString('{"data": 123}');

// Encoding options
$response->setEncodingOptions(JsonResponse::DEFAULT_ENCODING_OPTIONS | \JSON_PRESERVE_ZERO_FRACTION);

// JSONP
$response->setCallback('handleResponse');
```

**Security:** Always return an object (not an array) as the outermost JSON structure to prevent XSSI attacks.

---

## RedirectResponse

```php
use Symfony\Component\HttpFoundation\RedirectResponse;

$response = new RedirectResponse('https://example.com/');
$response = new RedirectResponse('/new-path', Response::HTTP_MOVED_PERMANENTLY);
```

---

## StreamedResponse

```php
use Symfony\Component\HttpFoundation\StreamedResponse;

// From callback
$response = new StreamedResponse(function (): void {
    echo 'Hello World';
    flush();
    sleep(2);
    echo 'More data';
    flush();
});

// From iterable chunks
$response = new StreamedResponse();
$response->setChunks(['Hello', ' World']);
$response->send();
```

---

## StreamedJsonResponse

```php
use Symfony\Component\HttpFoundation\StreamedJsonResponse;

function loadItems(): \Generator {
    yield ['id' => 1, 'title' => 'Item 1'];
    yield ['id' => 2, 'title' => 'Item 2'];
}

$response = new StreamedJsonResponse([
    '_embedded' => [
        'items' => loadItems(),
    ],
]);

// Or pass generator directly
$response = new StreamedJsonResponse(loadItems());
```

---

## BinaryFileResponse

```php
use Symfony\Component\HttpFoundation\BinaryFileResponse;
use Symfony\Component\HttpFoundation\ResponseHeaderBag;
use Symfony\Component\HttpFoundation\File\Stream;

$response = new BinaryFileResponse('/path/to/file.pdf');

// Force download
$response->setContentDisposition(
    ResponseHeaderBag::DISPOSITION_ATTACHMENT,
    'download.pdf'
);

// Delete after send
$response->deleteFileAfterSend();

// From SplTempFileObject
$file = new \SplTempFileObject();
$file->fwrite('content');
$file->rewind();
$response = new BinaryFileResponse($file);

// Stream (disables Range requests)
$response = new BinaryFileResponse(new Stream('path/to/stream'));

// X-Sendfile support
BinaryFileResponse::trustXSendfileTypeHeader();
```

---

## EventStreamResponse (Server-Sent Events)

```php
use Symfony\Component\HttpFoundation\EventStreamResponse;
use Symfony\Component\HttpFoundation\ServerEvent;

$response = new EventStreamResponse(function (): iterable {
    yield new ServerEvent('First message');
    sleep(1);
    yield new ServerEvent('Second message');
});

// Advanced event
$event = new ServerEvent(
    data: ['message' => 'hello'],   // String or iterable
    type: 'custom_event',           // Event type
    id: 'event-123',                // Event ID
    retry: 5000,                    // Reconnect interval (ms)
    comment: 'keep-alive'           // Comment
);
```

Automatically sets `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`.

---

## File Download Helper

```php
use Symfony\Component\HttpFoundation\HeaderUtils;

$disposition = HeaderUtils::makeDisposition(
    HeaderUtils::DISPOSITION_ATTACHMENT,
    'filename.pdf'
);
$response->headers->set('Content-Disposition', $disposition);
```

---

## HeaderUtils

```php
use Symfony\Component\HttpFoundation\HeaderUtils;

HeaderUtils::split('da, en-gb;q=0.8', ',;');
// [['da'], ['en-gb', 'q=0.8']]

HeaderUtils::combine([['foo', 'abc'], ['bar']]);
// ['foo' => 'abc', 'bar' => true]

HeaderUtils::toString(['foo' => 'abc', 'bar' => true], ',');
// 'foo=abc, bar'

HeaderUtils::quote('foo "bar"');
// '"foo \"bar\""'

HeaderUtils::unquote('"foo \"bar\""');
// 'foo "bar"'

HeaderUtils::parseQuery('foo[bar.baz]=qux');
// ['foo' => ['bar.baz' => 'qux']]  (preserves dots unlike parse_str)
```

---

## IpUtils

```php
use Symfony\Component\HttpFoundation\IpUtils;

// Anonymize
IpUtils::anonymize('123.234.235.236');         // '123.234.235.0'
IpUtils::anonymize('123.234.235.236', 3);      // '123.0.0.0'

// IPv6
IpUtils::anonymize('2a01:198:603:10:396e:4789:8e99:890f');
// '2a01:198:603::'

// CIDR check
IpUtils::checkIp('192.168.1.56', '192.168.1.0/16'); // true

// Private IP check
IpUtils::isPrivateIp('192.168.1.1'); // true
```

---

## Request Matchers

```php
use Symfony\Component\HttpFoundation\ChainRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\HostRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\PathRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\SchemeRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\MethodRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\IpsRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\HeaderRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\QueryParameterRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\PortRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\AttributesRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\IsJsonRequestMatcher;

// Single
$matcher = new SchemeRequestMatcher('https');
$matcher->matches($request); // bool

// Chain (all must match)
$chain = new ChainRequestMatcher([
    new HostRequestMatcher('example.com'),
    new PathRequestMatcher('/admin'),
    new MethodRequestMatcher(['POST', 'PUT']),
]);
$chain->matches($request); // bool
```

---

## UrlHelper

```php
use Symfony\Component\HttpFoundation\UrlHelper;

$urlHelper->getAbsoluteUrl('/path');   // Full URL with scheme+host
$urlHelper->getRelativePath('/path');  // Relative path
```

---

## Safe Content Preference

```php
if ($request->preferSafeContent()) {
    $response = new Response($safeContent);
    $response->setContentSafe();
}
```

---

## Key Classes Summary

| Class | Purpose |
|-------|---------|
| `Request` | HTTP request abstraction |
| `Response` | HTTP response |
| `JsonResponse` | JSON responses |
| `RedirectResponse` | HTTP redirects |
| `StreamedResponse` | Streamed responses |
| `StreamedJsonResponse` | Streamed JSON |
| `BinaryFileResponse` | File serving |
| `EventStreamResponse` | Server-Sent Events |
| `ParameterBag` | Key-value container |
| `InputBag` | User input (extends ParameterBag) |
| `HeaderBag` | HTTP headers |
| `FileBag` | Uploaded files |
| `ServerBag` | Server variables |
| `Cookie` | HTTP cookie |
| `IpUtils` | IP address utilities |
| `HeaderUtils` | Header parsing utilities |
| `AcceptHeader` | Accept header parsing |
| `UrlHelper` | URL generation |
| `ChainRequestMatcher` | Composite request matching |

## Links

- Documentation: https://symfony.com/doc/7.4/components/http_foundation.html
- GitHub: https://github.com/symfony/http-foundation
