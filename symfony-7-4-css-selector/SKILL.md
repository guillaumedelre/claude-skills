---
name: "symfony-7-4-css-selector"
description: "Symfony 7.4 CssSelector component reference for converting CSS selectors to XPath expressions. Use when working with CSS to XPath conversion, HTML/XML querying with CSS selectors, DOMXPath, SimpleXMLElement, or DomCrawler integration. Triggers on: CssSelector, CssSelectorConverter, CSS to XPath, toXPath, HTML/XML querying, DOM querying with CSS selectors."
---

# Symfony 7.4 CssSelector Component

GitHub: https://github.com/symfony/css-selector
Docs: https://symfony.com/doc/7.4/components/css_selector.html

## Quick Reference

### Installation

```bash
composer require symfony/css-selector
```

### Basic Usage - Convert CSS to XPath

```php
use Symfony\Component\CssSelector\CssSelectorConverter;

$converter = new CssSelectorConverter();

// Simple element selector
$xpath = $converter->toXPath('div.item > h4 > a');
// Result: descendant-or-self::div[@class and contains(concat(' ',normalize-space(@class), ' '), ' item ')]/h4/a
```

### Using with DOMXPath

```php
$document = new \DOMDocument();
$document->loadHTML($htmlContent);
$xpath = new \DOMXPath($document);

$converter = new CssSelectorConverter();
$expression = $converter->toXPath('div.content > p.intro');
$nodes = $xpath->query($expression);

foreach ($nodes as $node) {
    echo $node->textContent;
}
```

### Using with SimpleXMLElement

```php
$xml = simplexml_load_string($xmlContent);
$converter = new CssSelectorConverter();
$expression = $converter->toXPath('book > author');
$results = $xml->xpath($expression);
```

### HTML Mode vs XML Mode

```php
// HTML mode (default) - case-insensitive tag/attribute matching
$converter = new CssSelectorConverter();

// XML mode - case-sensitive matching
$converter = new CssSelectorConverter(false);
```

### Supported CSS Selectors

| Selector | Example | Supported |
|---|---|---|
| Element | `div`, `p`, `a` | Yes |
| Class | `.class` | Yes |
| ID | `#id` | Yes |
| Attribute | `[attr]`, `[attr=value]` | Yes |
| Descendant | `div p` | Yes |
| Child | `div > p` | Yes |
| Adjacent sibling | `div + p` | Yes |
| General sibling | `div ~ p` | Yes |
| `:first-child` | `li:first-child` | Yes |
| `:last-child` | `li:last-child` | Yes |
| `:nth-child()` | `li:nth-child(2n+1)` | Yes |
| `:enabled` / `:disabled` | `input:enabled` | Yes |
| `:checked` | `input:checked` | Yes |
| `:is()` / `:where()` | `:is(h1, h2)` | Yes (7.1+) |
| `:first-of-type` | `li:first-of-type` | Element only (not `*`) |
| Pseudo-elements | `::before` | No |
| `:hover` / `:focus` | `a:hover` | No |
| `:link` / `:visited` | `a:link` | No |

## Full Documentation

For complete details including all supported selectors, limitations, and integration patterns, see [references/css-selector.md](references/css-selector.md).
