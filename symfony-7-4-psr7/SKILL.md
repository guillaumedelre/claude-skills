---
name: "symfony-7-4-psr7"
description: "PSR-7 Bridge component for Symfony 7.4. Triggers on: PSR-7 Bridge, PSR-7 conversion, HttpFoundation to PSR-7, ServerRequestInterface, ResponseInterface, PsrHttpFactory, HttpFoundationFactory, PSR-17, HTTP message interoperability"
---

# Symfony PSR-7 Bridge

## Overview

The PSR-7 Bridge converts HttpFoundation objects from and to objects implementing HTTP message interfaces defined by PSR-7. This enables interoperability between Symfony's HTTP foundation and PSR-7 standard HTTP messages.

## Quick Reference

### Installation

```bash
composer require symfony/psr-http-message-bridge
composer require nyholm/psr7
```

### Converting HttpFoundation Request to PSR-7

```php
use Nyholm\Psr7\Factory\Psr17Factory;
use Symfony\Bridge\PsrHttpMessage\Factory\PsrHttpFactory;
use Symfony\Component\HttpFoundation\Request;

$symfonyRequest = new Request(
    [],
    [],
    [],
    [],
    [],
    ['HTTP_HOST' => 'example.com'],
    'Content'
);

$psr17Factory = new Psr17Factory();
$psrHttpFactory = new PsrHttpFactory(
    $psr17Factory,
    $psr17Factory,
    $psr17Factory,
    $psr17Factory
);
$psrRequest = $psrHttpFactory->createRequest($symfonyRequest);
```

### Converting HttpFoundation Response to PSR-7

```php
use Nyholm\Psr7\Factory\Psr17Factory;
use Symfony\Bridge\PsrHttpMessage\Factory\PsrHttpFactory;
use Symfony\Component\HttpFoundation\Response;

$symfonyResponse = new Response('Content');

$psr17Factory = new Psr17Factory();
$psrHttpFactory = new PsrHttpFactory(
    $psr17Factory,
    $psr17Factory,
    $psr17Factory,
    $psr17Factory
);
$psrResponse = $psrHttpFactory->createResponse($symfonyResponse);
```

### Converting PSR-7 Request to HttpFoundation

```php
use Symfony\Bridge\PsrHttpMessage\Factory\HttpFoundationFactory;

// $psrRequest is an instance of Psr\Http\Message\ServerRequestInterface
$httpFoundationFactory = new HttpFoundationFactory();
$symfonyRequest = $httpFoundationFactory->createRequest($psrRequest);
```

### Converting PSR-7 Response to HttpFoundation

```php
use Symfony\Bridge\PsrHttpMessage\Factory\HttpFoundationFactory;

// $psrResponse is an instance of Psr\Http\Message\ResponseInterface
$httpFoundationFactory = new HttpFoundationFactory();
$symfonyResponse = $httpFoundationFactory->createResponse($psrResponse);
```

## Key Interfaces

| Interface | Purpose |
|-----------|---------|
| `HttpMessageFactoryInterface` | Convert HttpFoundation to PSR-7 |
| `HttpFoundationFactoryInterface` | Convert PSR-7 to HttpFoundation |

## Full Documentation
- Docs: https://symfony.com/doc/7.4/components/psr7.html
- GitHub: https://github.com/symfony/psr-http-message-bridge
- **Full Reference**: See [references/psr7.md](references/psr7.md) for complete documentation
