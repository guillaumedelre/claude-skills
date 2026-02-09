---
name: "symfony-7-4-browser-kit"
description: "Symfony 7.4 BrowserKit component reference for simulating browser behavior in PHP. Use when writing functional tests, simulating HTTP client requests, crawling pages, clicking links, submitting forms, or working with cookies and history programmatically. Triggers on: BrowserKit, AbstractBrowser, HttpBrowser, functional testing, HTTP client simulation, crawling, CookieJar, browser history, submitForm, clickLink, jsonRequest, xmlHttpRequest."
---

# Symfony 7.4 BrowserKit Component

GitHub: https://github.com/symfony/browser-kit
Docs: https://symfony.com/doc/7.4/components/browser_kit.html

## Quick Reference

### Installation

```bash
composer require symfony/browser-kit
```

### Making Requests

```php
use Symfony\Component\BrowserKit\HttpBrowser;
use Symfony\Component\HttpClient\HttpClient;

$browser = new HttpBrowser(HttpClient::create());
$crawler = $browser->request('GET', 'https://example.com');

// JSON request
$crawler = $browser->jsonRequest('GET', '/api/endpoint', ['key' => 'value']);

// AJAX request
$crawler = $browser->xmlHttpRequest('GET', '/api/endpoint');
```

### Clicking Links

```php
$crawler = $browser->clickLink('Go elsewhere...');

// With custom headers
$crawler = $browser->clickLink('Go elsewhere...', ['X-Custom-Header' => 'value']);
```

### Submitting Forms

```php
$browser->submitForm('Log in', [
    'login' => 'my_user',
    'password' => 'my_pass',
]);

// With HTTP method and server params override
$browser->submitForm('Log in', ['login' => 'user'], 'PUT', ['HTTP_ACCEPT_LANGUAGE' => 'es']);
```

### Cookies

```php
use Symfony\Component\BrowserKit\Cookie;
use Symfony\Component\BrowserKit\CookieJar;

$cookieJar = $browser->getCookieJar();
$cookie = $cookieJar->get('name_of_the_cookie');

// Set cookies before request
$cookie = new Cookie('flavor', 'chocolate', strtotime('+1 day'));
$cookieJar = new CookieJar();
$cookieJar->set($cookie);
```

### History Navigation

```php
$crawler = $browser->back();
$crawler = $browser->forward();
$browser->restart(); // clears history and cookies
```

### Custom AbstractBrowser

```php
use Symfony\Component\BrowserKit\AbstractBrowser;
use Symfony\Component\BrowserKit\Response;

class TestClient extends AbstractBrowser
{
    protected function doRequest($request): Response
    {
        return new Response($content, $status, $headers);
    }
}
```

## Full Documentation

For complete details including cookie management, history API, response handling, custom header handling, and external HTTP requests, see [references/browser-kit.md](references/browser-kit.md).
