# The BrowserKit Component (Symfony 7.4)

GitHub: https://github.com/symfony/browser-kit
Docs: https://symfony.com/doc/7.4/components/browser_kit.html

The BrowserKit component simulates the behavior of a web browser, allowing you to make requests, click on links, and submit forms programmatically.

## Installation

```bash
composer require symfony/browser-kit
```

## Basic Usage

### Creating a Client

To create a client, extend the `AbstractBrowser` class and implement the `doRequest()` method:

```php
namespace Acme;

use Symfony\Component\BrowserKit\AbstractBrowser;
use Symfony\Component\BrowserKit\Response;

class Client extends AbstractBrowser
{
    protected function doRequest($request): Response
    {
        // ... convert request into a response

        return new Response($content, $status, $headers);
    }
}
```

For simple HTTP-based implementations, use the `HttpBrowser` provided by this component.

### Making Requests

Use the `request()` method to make HTTP requests:

```php
use Acme\Client;

$client = new Client();
$crawler = $client->request('GET', '/');
```

**Shortcut methods:**

```php
// JSON request
$crawler = $client->jsonRequest('GET', '/', ['some_parameter' => 'some_value']);

// AJAX request
$crawler = $client->xmlHttpRequest('GET', '/');
```

### Wrapping Response Content

Wrap HTML fragments that lack proper context using `wrapContent()`:

```php
$client = new Client();
$client->wrapContent('<table>%s</table>');

$crawler = $client->xmlHttpRequest('GET', '/fragment');
```

The `%s` placeholder will be replaced with the response content. The wrapper is only used for parsing and doesn't modify the actual response.

### Clicking Links

Simulate link clicks by passing the link text:

```php
$client = new Client();
$client->request('GET', '/product/123');

$crawler = $client->clickLink('Go elsewhere...');
```

Or use the Link object for more control:

```php
$crawler = $client->request('GET', '/product/123');
$link = $crawler->selectLink('Go elsewhere...')->link();
$client->click($link);
```

Add custom headers to link clicks:

```php
$crawler = $client->clickLink('Go elsewhere...', ['X-Custom-Header' => 'Some data']);
```

### Submitting Forms

Select and submit forms:

```php
$client = new Client();
$crawler = $client->request('GET', 'https://github.com/login');

// Find the form with the 'Log in' button and submit it
$client->submitForm('Log in');

// Override form field values
$client->submitForm('Log in', [
    'login' => 'my_user',
    'password' => 'my_pass',
    'file' => __FILE__, // for file uploads, use absolute path
]);

// Override HTTP method and server parameters
$client->submitForm(
    'Log in',
    ['login' => 'my_user', 'password' => 'my_pass'],
    'PUT',
    ['HTTP_ACCEPT_LANGUAGE' => 'es']
);
```

Or use the Form object:

```php
$form = $crawler->selectButton('Log in')->form();
$form['login'] = 'symfonyfan';
$form['password'] = 'anypass';

$crawler = $client->submit($form);
```

### Custom Header Handling

Override the `getHeaders()` method to handle special header capitalization:

```php
protected function getHeaders(Request $request): array
{
    $headers = parent::getHeaders($request);
    if (isset($request->getServer()['api_key'])) {
        $headers['api_key'] = $request->getServer()['api_key'];
    }

    return $headers;
}
```

## Cookies

### Retrieving Cookies

```php
$client = new Client();
$crawler = $client->request('GET', '/');

$cookieJar = $client->getCookieJar();

// Get a specific cookie
$cookie = $cookieJar->get('name_of_the_cookie');

// Access cookie data
$name       = $cookie->getName();
$value      = $cookie->getValue();
$rawValue   = $cookie->getRawValue();
$isSecure   = $cookie->isSecure();
$isHttpOnly = $cookie->isHttpOnly();
$isExpired  = $cookie->isExpired();
$expires    = $cookie->getExpiresTime();
$path       = $cookie->getPath();
$domain     = $cookie->getDomain();
$sameSite   = $cookie->getSameSite();
```

### Looping Through Cookies

```php
$cookieJar = $client->getCookieJar();

// Get all cookies
$cookies = $cookieJar->all();
foreach ($cookies as $cookie) {
    // ...
}

// Get all values
$values = $cookieJar->allValues('http://symfony.com');

// Get all raw values
$rawValues = $cookieJar->allRawValues('http://symfony.com');
```

### Setting Cookies

```php
use Symfony\Component\BrowserKit\Cookie;
use Symfony\Component\BrowserKit\CookieJar;

$cookie = new Cookie('flavor', 'chocolate', strtotime('+1 day'));
$cookieJar = new CookieJar();
$cookieJar->set($cookie);

$client = new Client([], null, $cookieJar);
```

### Sending Cookies

```php
$client->request('GET', '/', [], [], [
    'HTTP_COOKIE' => new Cookie('flavor', 'chocolate', strtotime('+1 day')),
    // Or pass as string:
    'HTTP_COOKIE' => 'flavor=chocolate; expires=Sat, 11 Feb 2023 12:18:13 GMT; Max-Age=86400; path=/'
]);
```

## History

Navigate through request history:

```php
$client = new Client();
$client->request('GET', '/');

$link = $crawler->selectLink('Documentation')->link();
$client->click($link);

// Go back to home page
$crawler = $client->back();

// Go forward to documentation page
$crawler = $client->forward();

// Check history position
if (!$client->getHistory()->isFirstPage()) {
    $crawler = $client->back();
}

if (!$client->getHistory()->isLastPage()) {
    $crawler = $client->forward();
}
```

Clear history and cookies:

```php
$client->restart();
```

## Making External HTTP Requests

Use `HttpBrowser` to make external HTTP requests via the Symfony HttpClient component:

```php
use Symfony\Component\BrowserKit\HttpBrowser;
use Symfony\Component\HttpClient\HttpClient;

$browser = new HttpBrowser(HttpClient::create());

$browser->request('GET', 'https://github.com');
$browser->clickLink('Sign in');
$browser->submitForm('Sign in', ['login' => '...', 'password' => '...']);
```

### Dealing with HTTP Responses

```php
$browser = new HttpBrowser(HttpClient::create());

$browser->request('GET', 'https://foo.com');
$response = $browser->getResponse();

// For JSON responses
$browser->request('GET', 'https://api.foo.com');
$response = $browser->getResponse()->toArray();
```

## Key Classes

- **`AbstractBrowser`** - Base class for browser simulation; extend and implement `doRequest()`.
- **`HttpBrowser`** - Concrete implementation using Symfony HttpClient for real HTTP requests.
- **`Response`** - Represents an HTTP response with content, status, and headers.
- **`Cookie`** - Represents an HTTP cookie with name, value, expiration, path, domain, secure, httpOnly, sameSite.
- **`CookieJar`** - Stores and manages cookies across requests.
- **`History`** - Tracks request history for back/forward navigation.

## Related Components

- [DomCrawler](https://symfony.com/doc/7.4/components/dom_crawler.html) - Parse and traverse HTML/XML documents (returned by `request()` calls).
- [CssSelector](https://symfony.com/doc/7.4/components/css_selector.html) - CSS selector to XPath conversion for use with DomCrawler.
- [HttpClient](https://symfony.com/doc/7.4/http_client.html) - Used by `HttpBrowser` to make real HTTP requests.
