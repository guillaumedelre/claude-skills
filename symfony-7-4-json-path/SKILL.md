---
name: "symfony-7-4-json-path"
description: "Provides guidance on using the Symfony JsonPath component for querying and extracting data from JSON structures using RFC 9535 JSONPath expressions. Triggers on: JsonPath, JSON querying, JSON data extraction, JSONPath expressions, JsonCrawler."
---

# Symfony 7.4 JsonPath Component

## Overview

The Symfony JsonPath component allows querying and extracting data from JSON structures using the RFC 9535 JSONPath standard. It provides `JsonCrawler` for executing queries and `JsonPath` for building queries programmatically.

## Quick Reference

### Installation

```bash
composer require symfony/json-path
```

### Core Usage

```php
use Symfony\Component\JsonPath\JsonCrawler;

$crawler = new JsonCrawler($jsonString);
$results = $crawler->find('$.store.book[0].title');
```

### Programmatic Query Building

```php
use Symfony\Component\JsonPath\JsonPath;

$path = (new JsonPath())
    ->key('store')
    ->key('book')
    ->index(0)
    ->key('title');

$results = $crawler->find($path);
```

### JsonPath Builder Methods

| Method | Description |
|--------|-------------|
| `key($name)` | Select a named key (auto-escaped) |
| `deepScan()` | Recursive descent (`..`) |
| `all()` | Wildcard (`[*]`) |
| `index($n)` | Array index |
| `first()` | Shortcut for `index(0)` |
| `last()` | Shortcut for `index(-1)` |
| `slice($start, $end, $step)` | Array slice |
| `filter($expr)` | Filter expression |

### Common Query Patterns

| Pattern | Expression |
|---------|------------|
| Root property | `$.store` |
| Nested property | `$.store.book[0].title` |
| All items | `$.store.book[*]` |
| Recursive descent | `$..author` |
| Filter by value | `$.store.book[?(@.price < 10)]` |
| Bracket notation | `$["store"]["book"][0]` |

### PHPUnit Assertions

```php
use Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait;

self::assertJsonPathCount(2, '$.books[*]', $json);
self::assertJsonPathSame('expected', '$.path', $json);
self::assertJsonPathContains('value', '$.path', $json);
```

## Full Documentation
- [Symfony JsonPath Documentation](https://symfony.com/doc/7.4/components/json_path.html)
- [GitHub: symfony/json-path](https://github.com/symfony/json-path)
- [RFC 9535 - JSONPath](https://www.rfc-editor.org/rfc/rfc9535)

## References

- [references/json-path.md](references/json-path.md) - Complete API reference and detailed usage guide
