# Symfony JsonPath Component - Complete Reference

## Installation

```bash
composer require symfony/json-path
```

## Classes

### JsonCrawler

The main class for querying JSON documents. Accepts a JSON string in its constructor and provides the `find()` method to execute JSONPath queries.

```php
use Symfony\Component\JsonPath\JsonCrawler;

$crawler = new JsonCrawler($jsonString);
$results = $crawler->find('$.store.book[0].title');
// Returns: ['Sayings of the Century']
```

The `find()` method always returns an array of matched values.

### JsonPath

A fluent builder for constructing JSONPath expressions programmatically with automatic escaping and syntax handling.

```php
use Symfony\Component\JsonPath\JsonPath;

$path = (new JsonPath())
    ->key('store')
    ->key('book')
    ->index(1);

$results = $crawler->find($path);
```

#### Methods

| Method | Purpose | Equivalent Expression |
|--------|---------|----------------------|
| `key(string $name)` | Select a named key (auto-escaped for special chars) | `$.name` or `$["name"]` |
| `deepScan()` | Recursive descent operator | `..` |
| `all()` | Wildcard selector | `[*]` |
| `index(int $num)` | Array index selector | `[n]` |
| `first()` | First element (shortcut for `index(0)`) | `[0]` |
| `last()` | Last element (shortcut for `index(-1)`) | `[-1]` |
| `slice(int $start, ?int $end, int $step = 1)` | Array slice | `[start:end:step]` |
| `filter(string $expr)` | Filter expression | `[?expr]` |

## JSONPath Expression Syntax

### Notation Styles

```php
// Dot notation
$crawler->find('$.store.book[0].title');

// Bracket notation (required for keys with special characters)
$crawler->find('$["store"]["book"][0]["title"]');

// Mixed notation
$crawler->find('$["store"].book[0].title');
```

### Operators

#### Root (`$`)
Every expression starts with `$` representing the root of the JSON document.

#### Child (`.key` or `["key"]`)
Selects a direct child property.

```php
$crawler->find('$.store');
$crawler->find('$.store.bicycle.color');
```

#### Recursive Descent (`..`)
Searches all descendants recursively.

```php
// Find all "author" values anywhere in the document
$crawler->find('$..author');

// Find all "price" values anywhere
$crawler->find('$..price');
```

#### Wildcard (`[*]`)
Matches all elements of an array or all values of an object.

```php
$crawler->find('$.store.book[*]');       // All books
$crawler->find('$.store.book[*].title'); // All book titles
```

#### Array Index (`[n]`)
Selects array element by index (0-based). Negative indices count from the end.

```php
$crawler->find('$.store.book[0]');   // First book
$crawler->find('$.store.book[-1]');  // Last book
```

#### Array Slice (`[start:end:step]`)
Selects a range of array elements.

```php
$crawler->find('$.store.book[0:2]');     // First two books
$crawler->find('$.store.book[0:4:2]');   // Every other book (indices 0 and 2)
```

#### Filter Expressions (`[?()]`)
Filters array elements based on conditions. Use `@` to reference the current element.

```php
// Books cheaper than $10
$crawler->find('$.store.book[?(@.price < 10)]');

// Books in the "fiction" category
$crawler->find("$.store.book[?(@.category == 'fiction')]");

// Regex matching on author name
$crawler->find('$.store.book[?match(@.author, "[A-Z].*el.+")]');
```

## Programmatic Query Building Examples

```php
use Symfony\Component\JsonPath\JsonPath;

// $.store.book[0].title
$path = (new JsonPath())
    ->key('store')
    ->key('book')
    ->first()
    ->key('title');

// $..price
$path = (new JsonPath())
    ->deepScan()
    ->key('price');

// $.store.book[*].author
$path = (new JsonPath())
    ->key('store')
    ->key('book')
    ->all()
    ->key('author');

// $.store.book[0:2]
$path = (new JsonPath())
    ->key('store')
    ->key('book')
    ->slice(0, 2);

// $.store.book[?(@.price < 10)]
$path = (new JsonPath())
    ->key('store')
    ->key('book')
    ->filter('@.price < 10');
```

## PHPUnit Testing Support

### JsonPathAssertionsTrait

Add JSON assertions to PHPUnit test cases:

```php
use PHPUnit\Framework\TestCase;
use Symfony\Component\JsonPath\Test\JsonPathAssertionsTrait;

class MyTest extends TestCase
{
    use JsonPathAssertionsTrait;

    public function testJsonStructure(): void
    {
        $json = '{"books": [{"title": "A"}, {"title": "B"}]}';

        self::assertJsonPathCount(2, '$.books[*]', $json);
        self::assertJsonPathSame('A', '$.books[0].title', $json);
        self::assertJsonPathContains('B', '$.books[*].title', $json);
    }
}
```

### Available Assertions

| Method | Description |
|--------|-------------|
| `assertJsonPathCount(int $expected, string $path, string $json)` | Assert number of matches |
| `assertJsonPathEquals(mixed $expected, string $path, string $json)` | Assert equality (loose `==`) |
| `assertJsonPathNotEquals(mixed $expected, string $path, string $json)` | Assert inequality (loose `!=`) |
| `assertJsonPathSame(mixed $expected, string $path, string $json)` | Assert strict equality (`===`) |
| `assertJsonPathNotSame(mixed $expected, string $path, string $json)` | Assert strict inequality (`!==`) |
| `assertJsonPathContains(mixed $value, string $path, string $json)` | Assert value exists in results |
| `assertJsonPathNotContains(mixed $value, string $path, string $json)` | Assert value not in results |

## Exception Handling

```php
use Symfony\Component\JsonPath\Exception\InvalidArgumentException;
use Symfony\Component\JsonPath\Exception\InvalidJsonStringInputException;
use Symfony\Component\JsonPath\Exception\JsonCrawlerException;
```

| Exception | When Thrown |
|-----------|------------|
| `InvalidArgumentException` | Constructor receives invalid JSON |
| `InvalidJsonStringInputException` | Malformed JSON encountered during `find()` |
| `JsonCrawlerException` | Invalid JSONPath expression (syntax errors, unknown functions) |

```php
try {
    $crawler = new JsonCrawler('invalid json');
} catch (InvalidArgumentException $e) {
    // Handle invalid JSON input
}

try {
    $crawler->find('$.store.book[?unknown(@.price)]');
} catch (JsonCrawlerException $e) {
    // Handle invalid JSONPath expression
}
```

## Complete Example

```php
use Symfony\Component\JsonPath\JsonCrawler;
use Symfony\Component\JsonPath\JsonPath;

$json = <<<'JSON'
{
    "store": {
        "book": [
            {"category": "reference", "author": "Nigel Rees", "title": "Sayings of the Century", "price": 8.95},
            {"category": "fiction", "author": "Evelyn Waugh", "title": "Sword of Honour", "price": 12.99},
            {"category": "fiction", "author": "Herman Melville", "title": "Moby Dick", "price": 8.99},
            {"category": "fiction", "author": "J.R.R. Tolkien", "title": "The Lord of the Rings", "price": 22.99}
        ],
        "bicycle": {"color": "red", "price": 399}
    }
}
JSON;

$crawler = new JsonCrawler($json);

// All authors
$authors = $crawler->find('$..author');
// ['Nigel Rees', 'Evelyn Waugh', 'Herman Melville', 'J.R.R. Tolkien']

// All prices (books + bicycle)
$prices = $crawler->find('$..price');
// [8.95, 12.99, 8.99, 22.99, 399]

// Cheap books (under $10)
$cheap = $crawler->find('$.store.book[?(@.price < 10)]');

// First book title using fluent API
$path = (new JsonPath())->key('store')->key('book')->first()->key('title');
$title = $crawler->find($path);
// ['Sayings of the Century']
```
