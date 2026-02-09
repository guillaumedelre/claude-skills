# Symfony 7.4 DomCrawler Component - Complete Reference

Source: https://symfony.com/doc/7.4/components/dom_crawler.html

The DomCrawler component eases DOM navigation for HTML and XML documents. It is **not designed for DOM manipulation or re-dumping HTML/XML**.

## Installation

```bash
composer require symfony/dom-crawler
```

For CSS selector support (used by the `filter()` method):

```bash
composer require symfony/css-selector
```

## Core Usage

The `Crawler` class provides methods to query and manipulate HTML and XML documents, representing a set of `DOMElement` objects:

```php
use Symfony\Component\DomCrawler\Crawler;

$html = <<<'HTML'
<!DOCTYPE html>
<html>
    <body>
        <p class="message">Hello World!</p>
        <p>Hello Crawler!</p>
    </body>
</html>
HTML;

$crawler = new Crawler($html);

foreach ($crawler as $domElement) {
    var_dump($domElement->nodeName);
}
```

## Node Filtering

### XPath Queries

```php
$crawler = $crawler->filterXPath('descendant-or-self::body/p');
```

### CSS Selectors (requires CssSelector component)

```php
$crawler = $crawler->filter('body > p');
```

### Complex Filtering with Callbacks

```php
$crawler = $crawler
    ->filter('body > p')
    ->reduce(function (Crawler $node, $i): bool {
        return ($i % 2) === 0; // Filter every other node
    });
```

### Namespace Support

```php
// Auto-discovered namespaces
$crawler = $crawler->filterXPath('//default:entry/media:group//yt:aspectRatio');
$crawler = $crawler->filter('default|entry media|group yt|aspectRatio');

// Explicit namespace registration
$crawler->registerNamespace('m', 'http://search.yahoo.com/mrss/');
$crawler = $crawler->filterXPath('//m:group//yt:aspectRatio');
```

### Match Selector Verification

```php
$crawler->matches('p.lorem'); // Returns bool
```

## Node Traversing

```php
// Access by position
$crawler->filter('body > p')->eq(0);

// First/Last nodes
$crawler->filter('body > p')->first();
$crawler->filter('body > p')->last();

// Siblings
$crawler->filter('body > p')->siblings();
$crawler->filter('body > p')->nextAll();
$crawler->filter('body > p')->previousAll();

// Children/Ancestors
$crawler->filter('body')->children();
$crawler->filter('body > p')->ancestors();

// Children matching selector
$crawler->filter('body')->children('p.lorem');

// Closest parent matching selector
$crawler->filter('p')->closest('div.container');
```

## Accessing Node Values

### Node Name (Tag)

```php
$tag = $crawler->filterXPath('//body/*')->nodeName();
```

### Text Content

```php
// Get text (throws InvalidArgumentException if node not found)
$message = $crawler->filterXPath('//body/p')->text();

// With default value (returns default if node not found)
$message = $crawler->filterXPath('//body/p')->text('Default text content');

// Text without trimming whitespace
$crawler->filterXPath('//body/p')->text('Default text content', false);

// Inner text only (direct descendant text, not nested elements)
$text = $crawler->filterXPath('//body/p')->innerText();
$text = $crawler->filterXPath('//body/p')->innerText(false); // Without normalize
```

### Attributes

```php
$class = $crawler->filterXPath('//body/p')->attr('class');

// With default value
$class = $crawler->filterXPath('//body/p')->attr('class', 'my-default-class');
```

### Extract Multiple Values

```php
$attributes = $crawler
    ->filterXpath('//body/p')
    ->extract(['_name', '_text', 'class']);
// Special attributes: _text (node value), _name (element name)
```

### Iterate with Each

```php
$nodeValues = $crawler->filter('p')->each(function (Crawler $node, $i): string {
    return $node->text();
});
```

### HTML Output

```php
// Inner HTML
$html = $crawler->html();
$html = $crawler->html('Default <strong>HTML</strong> content');

// Outer HTML
$html = $crawler->outerHtml();

// Manual iteration
$html = '';
foreach ($crawler as $domElement) {
    $html .= $domElement->ownerDocument->saveHTML($domElement);
}
```

## Adding Content

```php
$crawler = new Crawler('<html><body/></html>');

$crawler->addHtmlContent('<html><body/></html>');
$crawler->addXmlContent('<root><node/></root>');

$crawler->addContent('<html><body/></html>');
$crawler->addContent('<root><node/></root>', 'text/xml');

$crawler->add('<html><body/></html>');
$crawler->add('<root><node/></root>');

// Working with DOM objects
$domDocument = new \DOMDocument();
$domDocument->loadXml('<root><node/><node/></root>');
$nodeList = $domDocument->getElementsByTagName('node');
$node = $domDocument->getElementsByTagName('node')->item(0);

$crawler->addDocument($domDocument);
$crawler->addNodeList($nodeList);
$crawler->addNodes([$node]);
$crawler->addNode($node);
```

## Expression Evaluation (XPath)

```php
use Symfony\Component\DomCrawler\Crawler;

$html = '<html>
<body>
    <span id="article-100" class="article">Article 1</span>
    <span id="article-101" class="article">Article 2</span>
    <span id="article-102" class="article">Article 3</span>
</body>
</html>';

$crawler = new Crawler();
$crawler->addHtmlContent($html);

// Scalar evaluation returns array
$crawler->filterXPath('//span[contains(@id, "article-")]')->evaluate('substring-after(@id, "-")');
// Result: [0 => '100', 1 => '101', 2 => '102']

// DOM evaluation returns Crawler
$crawler->evaluate('//span[1]'); // Returns Crawler instance
```

## Working with Links

```php
// Select link by id, class, or content
$linkCrawler = $crawler->filter('#sign-up');
$linkCrawler = $crawler->filter('.user-profile');
$linkCrawler = $crawler->selectLink('Log in');

// Get Link object
$link = $linkCrawler->link();
$link = $crawler->selectLink('Log in')->link();

// Get URI (absolute, cleaned)
$uri = $link->getUri();
```

## Working with Images

```php
// Select by alt attribute
$imagesCrawler = $crawler->selectImage('Kitten');
$image = $imagesCrawler->image();

// Or combined
$image = $crawler->selectImage('Kitten')->image();

// Get URI
$uri = $image->getUri();
```

## Working with Forms

### Selecting Forms and Buttons

```php
// Select button by label or id
$form = $crawler->selectButton('My super button')->form();
$form = $crawler->selectButton('my-super-button')->form();

// Filter by form class
$crawler->filter('.form-vertical')->form();

// Fill form while selecting button
$form = $crawler->selectButton('my-super-button')->form([
    'name' => 'Ryan',
]);
```

### Form Information

```php
$uri = $form->getUri();
$method = $form->getMethod();
$name = $form->getName();
```

### Setting and Getting Form Values

```php
// Set values
$form->setValues([
    'registration[username]' => 'symfonyfan',
    'registration[terms]'    => 1,
]);

// Get flat array values
$values = $form->getValues();

// Get PHP-style nested array
$values = $form->getPhpValues();
```

### Working with Multi-dimensional Fields

```php
// HTML:
// <form>
//     <input name="multi[]">
//     <input name="multi[]">
//     <input name="multi[dimensional]">
//     <input name="multi[dimensional][]" value="1">
//     <input name="multi[dimensional][]" value="2">
//     <input name="multi[dimensional][]" value="3">
// </form>

// Set values
$form->setValues(['multi' => ['value']]);

$form->setValues(['multi' => [
    1             => 'value',
    'dimensional' => 'an other value',
]]);

// Tick multiple checkboxes
$form->setValues(['multi' => [
    'dimensional' => [1, 3],
]]);
```

### Interacting with Form Fields

```php
// Set text field
$form['registration[username]']->setValue('symfonyfan');

// Tick/untick checkbox
$form['registration[terms]']->tick();
$form['registration[terms]']->untick();

// Select option
$form['registration[birthday][year]']->select(1984);

// Select multiple options
$form['registration[interests]']->select(['symfony', 'cookies']);

// Upload file
$form['registration[photo]']->upload('/path/to/lucas.jpg');

// Disable validation for specific field
$form['country']->disableValidation()->select('Invalid value');

// Disable validation for whole form
$form->disableValidation();
$form['country']->select('Invalid value');
```

### Using Form Data

```php
// Get PHP values and files
$values = $form->getPhpValues();
$files = $form->getPhpFiles();

// Get data for HTTP request
$uri = $form->getUri();
$method = $form->getMethod();
$values = $form->getValues();
$files = $form->getFiles();
```

### Using with HttpBrowser

```php
use Symfony\Component\BrowserKit\HttpBrowser;
use Symfony\Component\HttpClient\HttpClient;

$browser = new HttpBrowser(HttpClient::create());
$crawler = $browser->request('GET', 'https://github.com/login');

// Select and fill form
$form = $crawler->selectButton('Sign in')->form();
$form['login'] = 'symfonyfan';
$form['password'] = 'anypass';

// Submit form
$crawler = $browser->submit($form);
```

## Resolving URIs

```php
use Symfony\Component\DomCrawler\UriResolver;

UriResolver::resolve('/foo', 'http://localhost/bar/foo/');
// Result: http://localhost/foo

UriResolver::resolve('?a=b', 'http://localhost/bar#foo');
// Result: http://localhost/bar?a=b

UriResolver::resolve('../../', 'http://localhost/');
// Result: http://localhost/
```

## Important Notes

- All filter and traversal methods return a **new** `Crawler` instance.
- Check if filter found results: `$crawler->count() > 0`.
- DomCrawler auto-fixes HTML to match the HTML5 spec.
- `formaction` and `formmethod` button attributes are supported.
- XPath queries in nested crawlers are evaluated in context of that crawler.
- The component is read-only: it is not designed for DOM manipulation or re-dumping HTML/XML.
