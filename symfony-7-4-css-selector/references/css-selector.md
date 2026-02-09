# Symfony 7.4 CssSelector Component - Full Reference

GitHub: https://github.com/symfony/css-selector
Docs: https://symfony.com/doc/7.4/components/css_selector.html

## Overview

The CssSelector component converts CSS selectors to XPath expressions. It provides a developer-friendly way to query HTML/XML documents using familiar CSS syntax instead of complex XPath expressions.

The component originated as a port of the Python `cssselect` library (v0.7.1).

## Installation

```bash
composer require symfony/css-selector
```

If installing outside a Symfony application, require `vendor/autoload.php` to enable Composer autoloading.

## Core API

### CssSelectorConverter

The main class. Converts CSS selector strings to XPath expression strings.

```php
use Symfony\Component\CssSelector\CssSelectorConverter;
```

#### Constructor

```php
// HTML mode (default): case-insensitive tag/attribute names
$converter = new CssSelectorConverter();

// XML mode: case-sensitive tag/attribute names
$converter = new CssSelectorConverter(false);
```

The boolean parameter `$html` (default `true`) controls whether HTML-specific behaviors apply (e.g., case-insensitive matching).

#### toXPath(string $cssExpr, string $prefix = 'descendant-or-self::'): string

Converts a CSS selector expression to an XPath expression.

```php
$converter = new CssSelectorConverter();

$converter->toXPath('div');
// "descendant-or-self::div"

$converter->toXPath('div.item > h4 > a');
// "descendant-or-self::div[@class and contains(concat(' ',normalize-space(@class), ' '), ' item ')]/h4/a"

$converter->toXPath('#main');
// "descendant-or-self::*[@id = 'main']"

$converter->toXPath('input[type="text"]');
// "descendant-or-self::input[@type = 'text']"
```

The `$prefix` parameter allows customizing the XPath prefix (default `descendant-or-self::`).

## Integration Patterns

### With DOMDocument and DOMXPath

```php
$html = '<html><body><div class="content"><p>Hello</p><p class="highlight">World</p></div></body></html>';

$document = new \DOMDocument();
$document->loadHTML($html);
$xpath = new \DOMXPath($document);

$converter = new CssSelectorConverter();

// Find all paragraphs inside .content
$expression = $converter->toXPath('div.content > p');
$nodes = $xpath->query($expression);

foreach ($nodes as $node) {
    echo $node->textContent . "\n";
}
// Output:
// Hello
// World

// Find only highlighted paragraphs
$expression = $converter->toXPath('p.highlight');
$nodes = $xpath->query($expression);
```

### With SimpleXMLElement

```php
$xml = <<<XML
<catalog>
    <book id="1">
        <title>PHP Reference</title>
        <author>John</author>
    </book>
    <book id="2">
        <title>Symfony Guide</title>
        <author>Jane</author>
    </book>
</catalog>
XML;

$catalog = simplexml_load_string($xml);
$converter = new CssSelectorConverter(false); // XML mode

$expression = $converter->toXPath('book > title');
$titles = $catalog->xpath($expression);

foreach ($titles as $title) {
    echo $title . "\n";
}
```

### With Symfony DomCrawler

The DomCrawler component uses CssSelector internally. When CssSelector is installed, `Crawler::filter()` accepts CSS selectors directly:

```php
use Symfony\Component\DomCrawler\Crawler;

$crawler = new Crawler($html);

// Uses CssSelector internally to convert to XPath
$links = $crawler->filter('div.nav > a.active');
$text = $crawler->filter('h1.title')->text();
$items = $crawler->filter('ul.menu > li')->each(function (Crawler $node) {
    return $node->text();
});
```

## Supported CSS Selectors

### Type / Element Selectors

| Selector | Example | Description |
|---|---|---|
| `E` | `div`, `p` | Element of type E |
| `*` | `*` | Any element |

### Attribute Selectors

| Selector | Example | Description |
|---|---|---|
| `E[attr]` | `a[href]` | E with attribute `attr` |
| `E[attr=val]` | `input[type="text"]` | E where `attr` equals `val` |
| `E[attr~=val]` | `div[class~="item"]` | E where `attr` contains word `val` |
| `E[attr|=val]` | `div[lang|="en"]` | E where `attr` starts with `val` or `val-` |
| `E[attr^=val]` | `a[href^="https"]` | E where `attr` starts with `val` |
| `E[attr$=val]` | `a[href$=".pdf"]` | E where `attr` ends with `val` |
| `E[attr*=val]` | `a[href*="example"]` | E where `attr` contains `val` |

### Class and ID Selectors

| Selector | Example | Description |
|---|---|---|
| `E.class` | `div.container` | E with class |
| `E#id` | `div#main` | E with ID |

### Combinators

| Selector | Example | Description |
|---|---|---|
| `E F` | `div p` | F descendant of E |
| `E > F` | `div > p` | F direct child of E |
| `E + F` | `h1 + p` | F immediately after E |
| `E ~ F` | `h1 ~ p` | F sibling after E |

### Pseudo-classes (Supported)

| Pseudo-class | Example | Description |
|---|---|---|
| `:first-child` | `li:first-child` | First child element |
| `:last-child` | `li:last-child` | Last child element |
| `:nth-child(n)` | `tr:nth-child(2n+1)` | Nth child element |
| `:nth-last-child(n)` | `li:nth-last-child(2)` | Nth child from end |
| `:only-child` | `p:only-child` | Only child element |
| `:first-of-type` | `li:first-of-type` | First of its type (element name required) |
| `:last-of-type` | `li:last-of-type` | Last of its type (element name required) |
| `:nth-of-type(n)` | `li:nth-of-type(odd)` | Nth of its type (element name required) |
| `:nth-last-of-type(n)` | `li:nth-last-of-type(1)` | Nth of type from end (element name required) |
| `:only-of-type` | `p:only-of-type` | Only of its type |
| `:empty` | `td:empty` | Element with no children |
| `:root` | `:root` | Document root element |
| `:enabled` | `input:enabled` | Enabled form element |
| `:disabled` | `input:disabled` | Disabled form element |
| `:checked` | `input:checked` | Checked input element |
| `:not(s)` | `div:not(.hidden)` | Element not matching s |
| `:contains(text)` | `p:contains("hello")` | Element containing text |
| `:scope` | `*:scope` | Scoped element |
| `:is(s)` | `:is(h1, h2, h3)` | Matches any selector in list (since 7.1) |
| `:where(s)` | `:where(h1, h2)` | Like `:is()` but zero specificity (since 7.1) |

### Pseudo-classes (NOT Supported)

| Pseudo-class | Reason |
|---|---|
| `:link`, `:visited`, `:target` | Browser link-state dependent |
| `:hover`, `:focus`, `:active` | Browser user-action dependent |
| `:invalid`, `:indeterminate` | Browser UI-state dependent |
| `*:first-of-type` | Only works with element names, not `*` |
| `*:last-of-type` | Only works with element names, not `*` |
| `*:nth-of-type` | Only works with element names, not `*` |
| `*:nth-last-of-type` | Only works with element names, not `*` |

### Pseudo-elements (NOT Supported)

`::before`, `::after`, `::first-line`, `::first-letter` -- these select portions of text, not elements, so they cannot be converted to XPath.

## Why CSS Selectors over XPath

CSS selectors are:
- More readable and concise
- Familiar to web developers (CSS, JavaScript `querySelectorAll`, jQuery)
- Sufficient for most element selection needs

XPath is:
- More powerful (can navigate up the tree, use functions)
- More verbose and harder to read
- Required by PHP DOM APIs (`DOMXPath::query()`, `SimpleXMLElement::xpath()`)

The CssSelector component bridges this gap by converting the simpler CSS syntax to XPath automatically.

## Error Handling

The component throws `Symfony\Component\CssSelector\Exception\SyntaxErrorException` when a CSS selector has invalid syntax:

```php
use Symfony\Component\CssSelector\CssSelectorConverter;
use Symfony\Component\CssSelector\Exception\SyntaxErrorException;

$converter = new CssSelectorConverter();

try {
    $converter->toXPath('div[');
} catch (SyntaxErrorException $e) {
    // Handle invalid selector
}
```

## Component Structure

Key namespaces:
- `Symfony\Component\CssSelector\CssSelectorConverter` - Main converter class
- `Symfony\Component\CssSelector\Exception\` - Exception classes
- `Symfony\Component\CssSelector\Node\` - Internal CSS selector node representations
- `Symfony\Component\CssSelector\Parser\` - CSS selector parser
- `Symfony\Component\CssSelector\XPath\` - XPath translation layer
