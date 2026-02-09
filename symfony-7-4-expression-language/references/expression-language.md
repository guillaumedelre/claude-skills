# Symfony 7.4 ExpressionLanguage Component - Full Documentation

GitHub: https://github.com/symfony/expression-language
Docs: https://symfony.com/doc/7.4/components/expression_language.html
License: MIT

## Overview

The ExpressionLanguage component provides an engine that can compile and evaluate expressions. An expression is a one-liner that returns a value (mostly Booleans). It allows users to use expressions inside configuration for more complex logic without introducing security problems.

## Installation

```bash
composer require symfony/expression-language
```

## Use Cases

- Security rules in the Symfony Framework
- Validation rules
- Route matching
- Creating a business rule engine

Example expressions:

```
# Get the special price if
user.getGroup() in ['good_customers', 'collaborator']

# Promote article to the homepage when
article.commentCount > 100 and article.category not in ["misc"]

# Send an alert when
product.stock < 15
```

## Basic Usage

### Evaluation and Compilation

The component provides two ways to work with expressions:

- **Evaluation**: the expression is evaluated without being compiled to PHP
- **Compilation**: the expression is compiled to PHP for caching

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$expressionLanguage = new ExpressionLanguage();

// Evaluation: returns the result directly
var_dump($expressionLanguage->evaluate('1 + 2')); // displays 3

// Compilation: returns PHP code as a string
var_dump($expressionLanguage->compile('1 + 2')); // displays '(1 + 2)'
```

### Parsing and Linting Expressions

The `parse()` method returns a `ParsedExpression` instance for inspection:

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$expressionLanguage = new ExpressionLanguage();

var_dump($expressionLanguage->parse('1 + 2', []));
// displays the AST nodes of the expression

$expressionLanguage->lint('1 + 2', []); // doesn't throw anything

$expressionLanguage->lint('1 + a', []);
// throws a SyntaxError exception:
// "Variable "a" is not valid around position 5 for expression `1 + a`."
```

#### Parser Flags

Two flags are available:

- `IGNORE_UNKNOWN_VARIABLES`: don't throw an exception if a variable is not defined
- `IGNORE_UNKNOWN_FUNCTIONS`: don't throw an exception if a function is not defined

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;
use Symfony\Component\ExpressionLanguage\Parser;

$expressionLanguage = new ExpressionLanguage();

// does not throw a SyntaxError
$expressionLanguage->lint(
    'unknown_var + unknown_function()',
    [],
    Parser::IGNORE_UNKNOWN_VARIABLES | Parser::IGNORE_UNKNOWN_FUNCTIONS
);
```

## Passing Variables

You can pass variables of any valid PHP type (including objects) to expressions:

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$expressionLanguage = new ExpressionLanguage();

class Apple
{
    public string $variety;
}

$apple = new Apple();
$apple->variety = 'Honeycrisp';

var_dump($expressionLanguage->evaluate(
    'fruit.variety',
    [
        'fruit' => $apple,
    ]
)); // displays "Honeycrisp"
```

When compiling, pass variable names (not values):

```php
$expressionLanguage->compile(
    'fruit.variety',
    ['fruit']
);
```

## Caching

The ExpressionLanguage component caches parsed expressions internally to improve performance. Both `evaluate()` and `compile()` use the `parse()` method, which returns a `ParsedExpression` that is cached.

### Custom Cache Pool

You can customize caching by injecting a PSR-6 `CacheItemPoolInterface`:

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$cache = new RedisAdapter(/* ... */);
$expressionLanguage = new ExpressionLanguage($cache);
```

### Using Parsed and Serialized Expressions

Both `evaluate()` and `compile()` can handle `ParsedExpression` and `SerializedParsedExpression`:

```php
// Using ParsedExpression
$expression = $expressionLanguage->parse('1 + 4', []);
var_dump($expressionLanguage->evaluate($expression)); // prints 5
```

```php
use Symfony\Component\ExpressionLanguage\SerializedParsedExpression;

$expression = new SerializedParsedExpression(
    '1 + 4',
    serialize($expressionLanguage->parse('1 + 4', [])->getNodes())
);

var_dump($expressionLanguage->evaluate($expression)); // prints 5
```

## AST Dumping and Editing

### Dumping the AST

Call the `getNodes()` method after parsing to get the Abstract Syntax Tree:

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$ast = (new ExpressionLanguage())
    ->parse('1 + 2', [])
    ->getNodes()
;

// dump the AST nodes for inspection
var_dump($ast);

// dump the AST nodes as a string representation
$astAsString = $ast->dump();
```

### Manipulating the AST

Call the `toArray()` method to turn the AST into an array for manipulation:

```php
$astAsArray = (new ExpressionLanguage())
    ->parse('1 + 2', [])
    ->getNodes()
    ->toArray()
;
```

## Extending the ExpressionLanguage

### Registering Custom Functions

Use the `register()` method to add custom functions. It takes 3 arguments:

- **name** - The function name in an expression
- **compiler** - A function executed when compiling the expression
- **evaluator** - A function executed when evaluating the expression

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$expressionLanguage = new ExpressionLanguage();
$expressionLanguage->register('lowercase', function ($str): string {
    return sprintf('(is_string(%1$s) ? strtolower(%1$s) : %1$s)', $str);
}, function ($arguments, $str): string {
    if (!is_string($str)) {
        return $str;
    }

    return strtolower($str);
});

var_dump($expressionLanguage->evaluate('lowercase("HELLO")'));
// this will print: hello
```

### Using Expression Providers

Create a class implementing `ExpressionFunctionProviderInterface` to register multiple functions:

```php
use Symfony\Component\ExpressionLanguage\ExpressionFunction;
use Symfony\Component\ExpressionLanguage\ExpressionFunctionProviderInterface;

class StringExpressionLanguageProvider implements ExpressionFunctionProviderInterface
{
    public function getFunctions(): array
    {
        return [
            new ExpressionFunction('lowercase', function ($str): string {
                return sprintf('(is_string(%1$s) ? strtolower(%1$s) : %1$s)', $str);
            }, function ($arguments, $str): string {
                if (!is_string($str)) {
                    return $str;
                }

                return strtolower($str);
            }),
        ];
    }
}
```

#### Creating Functions from PHP Functions

Use the `fromPhp()` static method:

```php
ExpressionFunction::fromPhp('strtoupper');
```

For namespaced functions, provide a second argument:

```php
ExpressionFunction::fromPhp('My\strtoupper', 'my_strtoupper');
```

#### Registering Providers

Register providers via constructor or `registerProvider()`:

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

// using the constructor
$expressionLanguage = new ExpressionLanguage(null, [
    new StringExpressionLanguageProvider(),
]);

// using registerProvider()
$expressionLanguage->registerProvider(new StringExpressionLanguageProvider());
```

#### Creating a Custom ExpressionLanguage Class

Recommended approach - extend the class and prepend default providers:

```php
use Psr\Cache\CacheItemPoolInterface;
use Symfony\Component\ExpressionLanguage\ExpressionLanguage as BaseExpressionLanguage;

class ExpressionLanguage extends BaseExpressionLanguage
{
    public function __construct(?CacheItemPoolInterface $cache = null, array $providers = [])
    {
        // prepends the default provider to let users override it
        array_unshift($providers, new StringExpressionLanguageProvider());

        parent::__construct($cache, $providers);
    }
}
```

## Key Classes

| Class | Purpose |
|-------|---------|
| `ExpressionLanguage` | Main entry point for evaluating, compiling, and parsing expressions |
| `Expression` | Holds an expression string |
| `ParsedExpression` | A parsed expression containing the AST node tree |
| `SerializedParsedExpression` | A serialized version of a parsed expression for storage |
| `ExpressionFunction` | Represents a registered function with compiler and evaluator callables |
| `ExpressionFunctionProviderInterface` | Interface for classes that provide expression functions |
| `Parser` | Parses token streams into AST nodes; provides parser flag constants |
| `Lexer` | Tokenizes expression strings |
| `Compiler` | Compiles AST nodes into PHP code |
| `Node\Node` | Base class for AST nodes |
