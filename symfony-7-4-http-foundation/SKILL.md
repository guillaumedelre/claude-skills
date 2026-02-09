---
name: "symfony-7-4-http-foundation"
description: "Symfony 7.4 HttpFoundation component reference. Triggers on: HttpFoundation, Request, Response, Session, Cookie, File upload, ParameterBag, HeaderBag, JsonResponse, RedirectResponse, StreamedResponse, BinaryFileResponse, InputBag, FileBag, ServerBag, AcceptHeader, IpUtils, HeaderUtils, StreamedJsonResponse, EventStreamResponse, ServerEvent, UrlHelper."
---

# Symfony 7.4 HttpFoundation

## Overview

The HttpFoundation component replaces PHP's native superglobals (`$_GET`, `$_POST`, `$_COOKIE`, `$_FILES`, `$_SERVER`) and functions (`header()`, `setcookie()`, `echo`) with an object-oriented API for HTTP request/response handling.

## Quick Reference

### Request

```php
use Symfony\Component\HttpFoundation\Request;

$request = Request::createFromGlobals();
$request = Request::create('/path', 'POST', ['key' => 'value']);

$request->query->get('page', 1);        // $_GET
$request->request->get('name');          // $_POST
$request->cookies->get('token');         // $_COOKIE
$request->files->get('upload');          // $_FILES
$request->server->get('HTTP_HOST');      // $_SERVER
$request->headers->get('Content-Type');  // HTTP headers
$request->attributes->get('_route');     // Custom/app data

$request->getMethod();       // GET, POST, PUT, DELETE...
$request->getPathInfo();     // /path
$request->getContent();      // Raw body
$request->toArray();         // JSON-decoded body
$request->getPayload();      // InputBag from POST or JSON body
```

### Response

```php
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpFoundation\BinaryFileResponse;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Symfony\Component\HttpFoundation\StreamedJsonResponse;

$response = new Response('Hello', Response::HTTP_OK, ['Content-Type' => 'text/html']);
$response = new JsonResponse(['data' => 123]);
$response = new RedirectResponse('https://example.com/');
$response = new BinaryFileResponse('/path/to/file.pdf');
$response = new StreamedResponse(function () { echo 'chunk'; flush(); });
$response = new StreamedJsonResponse(['items' => $generator]);
```

### Cookie

```php
use Symfony\Component\HttpFoundation\Cookie;

$cookie = Cookie::create('name')
    ->withValue('value')
    ->withExpires(strtotime('+1 day'))
    ->withDomain('.example.com')
    ->withSecure(true)
    ->withHttpOnly(true)
    ->withSameSite(Cookie::SAMESITE_LAX);

$response->headers->setCookie($cookie);
$response->headers->clearCookie('name');
```

### Cache Headers

```php
$response->setPublic();
$response->setMaxAge(3600);
$response->setSharedMaxAge(600);
$response->setETag('abc');
$response->setLastModified(new \DateTime());
$response->setVary(['Accept-Encoding']);
```

## Full Documentation
- Documentation: https://symfony.com/doc/7.4/components/http_foundation.html
- GitHub: https://github.com/symfony/http-foundation

## Full Documentation
See [references/http-foundation.md](references/http-foundation.md) for complete API documentation including ParameterBag methods, HeaderUtils, IpUtils, AcceptHeader, request matchers, SSE, and all response types.
