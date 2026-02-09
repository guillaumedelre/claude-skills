---
name: "symfony-7-4-dom-crawler"
description: "Symfony 7.4 DomCrawler component reference for HTML and XML document navigation. Use when parsing HTML/XML, traversing DOM trees, selecting elements with CSS selectors or XPath, extracting text/attributes, working with links/images/forms, or web scraping. Triggers on: DomCrawler, Crawler, HTML parsing, XML parsing, DOM traversal, CSS selectors, XPath queries, form handling, selectButton, selectLink, filterXPath, filter(), web scraping, UriResolver."
---

# Symfony 7.4 DomCrawler Component

GitHub: https://github.com/symfony/dom-crawler
Docs: https://symfony.com/doc/7.4/components/dom_crawler.html

## Quick Reference

### Installation

```bash
composer require symfony/dom-crawler
# For CSS selector support:
composer require symfony/css-selector
```

### Creating a Crawler

```php
use Symfony\Component\DomCrawler\Crawler;

$crawler = new Crawler($html);
// Or add content later:
$crawler = new Crawler();
$crawler->addHtmlContent('<html><body><p>Hello</p></body></html>');
$crawler->addXmlContent('<root><node/></root>');
```

### Filtering with CSS Selectors & XPath

```php
// CSS selectors (requires symfony/css-selector)
$crawler->filter('body > p');
$crawler->filter('.message');
$crawler->filter('#main-content a.link');

// XPath
$crawler->filterXPath('descendant-or-self::body/p');

// Reduce with callback
$crawler->filter('p')->reduce(function (Crawler $node, $i): bool {
    return ($i % 2) === 0;
});
```

### Traversing Nodes

```php
$crawler->filter('p')->eq(0);        // By position
$crawler->filter('p')->first();       // First node
$crawler->filter('p')->last();        // Last node
$crawler->filter('p')->siblings();    // Siblings
$crawler->filter('p')->nextAll();     // Next siblings
$crawler->filter('p')->previousAll(); // Previous siblings
$crawler->filter('body')->children(); // Children
$crawler->filter('p')->ancestors();   // Ancestors
$crawler->filter('p')->closest('div');// Closest matching parent
```

### Extracting Values

```php
$crawler->filter('p')->text();                    // Text content
$crawler->filter('p')->text('Default');           // With default
$crawler->filter('p')->innerText();               // Direct text only
$crawler->filter('p')->attr('class');             // Attribute
$crawler->filter('p')->attr('class', 'default');  // Attr with default
$crawler->filter('p')->nodeName();                // Tag name
$crawler->filter('p')->html();                    // Inner HTML
$crawler->filter('p')->outerHtml();               // Outer HTML
$crawler->filter('p')->extract(['_name', '_text', 'class']); // Multiple
$crawler->filter('p')->each(function (Crawler $node, $i) {
    return $node->text();
});
```

### Working with Forms

```php
$form = $crawler->selectButton('Submit')->form();
$form = $crawler->selectButton('Submit')->form([
    'name' => 'Ryan',
]);

$form['registration[username]']->setValue('symfonyfan');
$form['registration[terms]']->tick();
$form['registration[birthday][year]']->select(1984);
$form['registration[photo]']->upload('/path/to/file.jpg');

$uri = $form->getUri();
$method = $form->getMethod();
$values = $form->getPhpValues();
$files = $form->getPhpFiles();
```

### Working with Links & Images

```php
$link = $crawler->selectLink('Log in')->link();
$uri = $link->getUri();

$image = $crawler->selectImage('Kitten')->image();
$uri = $image->getUri();
```

### URI Resolution

```php
use Symfony\Component\DomCrawler\UriResolver;

UriResolver::resolve('/foo', 'http://localhost/bar/foo/');
// http://localhost/foo
```

## Full Documentation

For complete details including namespace handling, expression evaluation, adding content from DOM objects, multi-dimensional form fields, form validation disabling, and integration with HttpBrowser, see [references/dom-crawler.md](references/dom-crawler.md).
