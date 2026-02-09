---
name: "symfony-7-4-expression-language"
description: "Symfony 7.4 ExpressionLanguage component reference for compiling and evaluating expressions. Use when working with expression evaluation, expression compilation, custom expression functions, expression providers, expression caching, or any ExpressionLanguage-related Symfony code. Triggers on: ExpressionLanguage, expression evaluation, expression compilation, custom functions, ExpressionFunctionProviderInterface, ParsedExpression, expression syntax, evaluate(), compile(), expression linting."
---

# Symfony 7.4 ExpressionLanguage Component

GitHub: https://github.com/symfony/expression-language
Docs: https://symfony.com/doc/7.4/components/expression_language.html

## Quick Reference

### Basic Usage - Evaluate & Compile

```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;

$expressionLanguage = new ExpressionLanguage();

// Evaluation: returns the result directly
$result = $expressionLanguage->evaluate('1 + 2'); // 3

// Compilation: returns PHP code as a string
$code = $expressionLanguage->compile('1 + 2'); // '(1 + 2)'
```

### Passing Variables

```php
$expressionLanguage->evaluate(
    'fruit.variety',
    ['fruit' => $apple]
);

$expressionLanguage->compile(
    'fruit.variety',
    ['fruit']
);
```

### Linting Expressions

```php
use Symfony\Component\ExpressionLanguage\Parser;

$expressionLanguage->lint('1 + 2', []); // valid, no exception

// Ignore unknown variables/functions (Symfony 7.1+)
$expressionLanguage->lint(
    'unknown_var',
    [],
    Parser::IGNORE_UNKNOWN_VARIABLES | Parser::IGNORE_UNKNOWN_FUNCTIONS
);
```

### Registering Custom Functions

```php
$expressionLanguage->register('lowercase',
    function ($str): string {
        return sprintf('(is_string(%1$s) ? strtolower(%1$s) : %1$s)', $str);
    },
    function ($arguments, $str): string {
        if (!is_string($str)) {
            return $str;
        }
        return strtolower($str);
    }
);

$expressionLanguage->evaluate('lowercase("HELLO")'); // 'hello'
```

### Expression Function Provider

```php
use Symfony\Component\ExpressionLanguage\ExpressionFunction;
use Symfony\Component\ExpressionLanguage\ExpressionFunctionProviderInterface;

class StringExpressionLanguageProvider implements ExpressionFunctionProviderInterface
{
    public function getFunctions(): array
    {
        return [
            ExpressionFunction::fromPhp('strtoupper'),
        ];
    }
}

$expressionLanguage = new ExpressionLanguage(null, [
    new StringExpressionLanguageProvider(),
]);
```

### Caching with PSR-6

```php
use Symfony\Component\Cache\Adapter\RedisAdapter;

$cache = new RedisAdapter(/* ... */);
$expressionLanguage = new ExpressionLanguage($cache);
```

## Full Documentation

For complete details including AST dumping/editing, serialized expressions, all parser flags, and advanced extension patterns, see [references/expression-language.md](references/expression-language.md).
