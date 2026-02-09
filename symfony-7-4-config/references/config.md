# Symfony 7.4 Config - Complete Reference

## Table of Contents

- [Installation](#installation)
- [Defining Configuration](#defining-configuration)
- [Node Types](#node-types)
- [Node Constraints](#node-constraints)
- [Array Nodes](#array-nodes)
- [Default and Required Values](#default-and-required-values)
- [Optional Sections](#optional-sections)
- [Normalization](#normalization)
- [Validation Rules](#validation-rules)
- [Deprecation and Documentation](#deprecation-and-documentation)
- [Merging and Overwriting](#merging-and-overwriting)
- [Appending Sections](#appending-sections)
- [Processing Configuration](#processing-configuration)
- [Locating Resources](#locating-resources)
- [Resource Loaders](#resource-loaders)
- [Loader Resolution](#loader-resolution)
- [Caching Configuration](#caching-configuration)

---

## Installation

```bash
composer require symfony/config
```

## Defining Configuration

Implement `ConfigurationInterface` to define a configuration tree:

```php
namespace Acme\DatabaseConfiguration;

use Symfony\Component\Config\Definition\Builder\TreeBuilder;
use Symfony\Component\Config\Definition\ConfigurationInterface;

class DatabaseConfiguration implements ConfigurationInterface
{
    public function getConfigTreeBuilder(): TreeBuilder
    {
        $treeBuilder = new TreeBuilder('database');

        $treeBuilder->getRootNode()
            ->children()
                ->scalarNode('driver')->isRequired()->cannotBeEmpty()->end()
                ->scalarNode('host')->defaultValue('localhost')->end()
                ->scalarNode('username')->end()
                ->scalarNode('password')->end()
                ->booleanNode('memory')->defaultFalse()->end()
            ->end()
        ;

        return $treeBuilder;
    }
}
```

## Node Types

| Type | Method | Description |
|------|--------|-------------|
| **scalar** | `scalarNode()` | Generic type (booleans, strings, integers, floats, null) |
| **boolean** | `booleanNode()` | Boolean values |
| **string** | `stringNode()` | String values (Symfony 7.2+) |
| **integer** | `integerNode()` | Integer values |
| **float** | `floatNode()` | Floating-point values |
| **enum** | `enumNode()` | Finite set of allowed values |
| **array** | `arrayNode()` | Complex structures |
| **variable** | `variableNode()` | No type validation |

### Basic Example

```php
$rootNode
    ->children()
        ->booleanNode('auto_connect')->defaultTrue()->end()
        ->scalarNode('default_connection')->defaultValue('mysql')->end()
        ->stringNode('username')->defaultValue('root')->end()
    ->end()
;
```

## Node Constraints

### Numeric Constraints

```php
$rootNode
    ->children()
        ->integerNode('positive_value')->min(0)->end()
        ->floatNode('big_value')->max(5E45)->end()
        ->integerNode('value_inside_a_range')->min(-50)->max(50)->end()
    ->end()
;
```

### Enum Nodes

```php
$rootNode
    ->children()
        ->enumNode('delivery')
            ->values(['standard', 'expedited', 'priority'])
        ->end()
    ->end()
;
```

With PHP enums (Symfony 7.3+):

```php
enum Delivery: string
{
    case Standard = 'standard';
    case Expedited = 'expedited';
    case Priority = 'priority';
}

$rootNode
    ->children()
        ->enumNode('delivery')
            ->enumFqcn(Delivery::class)
        ->end()
    ->end()
;
```

## Array Nodes

### Simple Array with Predefined Children

```php
$rootNode
    ->children()
        ->arrayNode('connection')
            ->children()
                ->scalarNode('driver')->end()
                ->scalarNode('host')->end()
                ->scalarNode('username')->end()
                ->scalarNode('password')->end()
            ->end()
        ->end()
    ->end()
;
```

### Prototype Arrays (Repeating Structures)

```php
$rootNode
    ->children()
        ->arrayNode('connections')
            ->arrayPrototype()
                ->children()
                    ->scalarNode('driver')->end()
                    ->scalarNode('host')->end()
                    ->scalarNode('username')->end()
                    ->scalarNode('password')->end()
                ->end()
            ->end()
        ->end()
    ->end()
;
```

### Array Node Options

```php
$node
    ->children()
        ->arrayNode('drivers', 'driver')
            ->useAttributeAsKey('name')
            ->requiresAtLeastOneElement()
            ->addDefaultsIfNotSet()
            ->normalizeKeys(false)
            ->ignoreExtraKeys()
            ->arrayPrototype()
                ->children()
                    ->scalarNode('value')->end()
                ->end()
            ->end()
        ->end()
    ->end()
;
```

### Prototype Types

- `->arrayPrototype()` - array children
- `->scalarPrototype()` - scalar children
- `->integerPrototype()` - integer children
- `->booleanPrototype()` - boolean children
- `->floatPrototype()` - float children
- `->enumPrototype()` - enum children
- `->variablePrototype()` - variable (untyped) children

## Default and Required Values

```php
$rootNode
    ->children()
        ->scalarNode('driver')
            ->isRequired()
            ->cannotBeEmpty()
        ->end()
        ->scalarNode('host')
            ->defaultValue('localhost')
        ->end()
        ->booleanNode('memory')
            ->defaultFalse()
        ->end()
        ->booleanNode('auto_connect')
            ->defaultTrue()
        ->end()
        ->scalarNode('default_connection')
            ->defaultNull()
        ->end()
    ->end()
;
```

### Replacement Values

```php
->treatNullLike(['enabled' => true])
->treatFalseLike(['enabled' => false])
->treatTrueLike(['enabled' => true])
```

## Optional Sections

### canBeEnabled / canBeDisabled

```php
$arrayNode
    ->canBeEnabled()   // disabled by default, enables with empty array or true
;

// Equivalent to:
$arrayNode
    ->treatFalseLike(['enabled' => false])
    ->treatTrueLike(['enabled' => true])
    ->treatNullLike(['enabled' => true])
    ->children()
        ->booleanNode('enabled')
            ->defaultFalse()
        ->end()
    ->end()
;

$arrayNode->canBeDisabled();  // enabled by default
```

## Normalization

### Key Normalization

Dashes and underscores are normalized automatically:
- YAML: `auto_connect`
- XML: `auto-connect`
- Both normalized to: `auto_connect`

Disable with `->normalizeKeys(false)`.

### Singular/Plural Array Normalization

```php
$rootNode
    ->children()
        ->arrayNode('extensions', 'extension')
            ->scalarPrototype()->end()
        ->end()
    ->end()
;
```

### Custom Normalization

```php
$rootNode
    ->children()
        ->arrayNode('connection')
            ->beforeNormalization()
                ->ifString()
                ->then(function (string $v): array {
                    return ['name' => $v];
                })
            ->end()
            ->children()
                ->scalarNode('name')->isRequired()->end()
            ->end()
        ->end()
    ->end()
;
```

## Validation Rules

### Basic Validation

```php
$rootNode
    ->children()
        ->scalarNode('driver')
            ->isRequired()
            ->validate()
                ->ifNotInArray(['mysql', 'sqlite', 'mssql'])
                ->thenInvalid('Invalid database driver %s')
            ->end()
        ->end()
    ->end()
;
```

### Validation Conditions

| Method | Description |
|--------|-------------|
| `ifTrue()` | Condition is true |
| `ifFalse()` | Condition is false (Symfony 7.3+) |
| `ifString()` | Value is string |
| `ifNull()` | Value is null |
| `ifEmpty()` | Value is empty |
| `ifArray()` | Value is array |
| `ifInArray()` | Value in array |
| `ifNotInArray()` | Value not in array |
| `always()` | Always execute |

### Validation Actions

| Method | Description |
|--------|-------------|
| `then()` | Return modified value |
| `thenEmptyArray()` | Return empty array |
| `thenInvalid()` | Throw InvalidConfigurationException |
| `thenUnset()` | Unset the node |

### Custom Validation Example

```php
->validate()
    ->ifTrue(function ($v) { return !is_array($v); })
    ->then(function ($v) { return [$v]; })
->end()
```

## Deprecation and Documentation

### Deprecating Options

```php
$rootNode
    ->children()
        ->integerNode('old_option')
            ->setDeprecated('acme/package', '1.2')
        ->end()
        ->integerNode('other_old_option')
            ->setDeprecated(
                'acme/package',
                '1.2',
                'The "%node%" option is deprecated. Use "new_config_option" instead.'
            )
        ->end()
    ->end()
;
```

### Documenting Options

```php
$rootNode
    ->children()
        ->integerNode('entries_per_page')
            ->info('This value is only used for the search results page.')
            ->defaultValue(25)
        ->end()
    ->end()
;
```

### Documentation URL (Symfony 7.3+)

```php
$rootNode
    ->docUrl('Full documentation at https://example.com/docs/{version:major}.{version:minor}/reference.html')
    ->children()
        // ...
    ->end()
;
```

Placeholders: `{version:major}`, `{version:minor}`, `{package}`.

## Merging and Overwriting

```php
$rootNode
    ->children()
        ->arrayNode('options')
            ->performNoDeepMerging()      // overwrite instead of merge
        ->end()
        ->scalarNode('key')
            ->cannotBeOverwritten()       // prevent overwriting
        ->end()
    ->end()
;
```

## Appending Sections

```php
public function getConfigTreeBuilder(): TreeBuilder
{
    $treeBuilder = new TreeBuilder('database');

    $treeBuilder->getRootNode()
        ->children()
            ->arrayNode('connection')
                ->children()
                    ->scalarNode('driver')->isRequired()->cannotBeEmpty()->end()
                    ->scalarNode('host')->defaultValue('localhost')->end()
                ->end()
                ->append($this->addParametersNode())
            ->end()
        ->end()
    ;

    return $treeBuilder;
}

public function addParametersNode(): NodeDefinition
{
    $treeBuilder = new TreeBuilder('parameters');

    $node = $treeBuilder->getRootNode()
        ->isRequired()
        ->requiresAtLeastOneElement()
        ->useAttributeAsKey('name')
        ->arrayPrototype()
            ->children()
                ->scalarNode('value')->isRequired()->end()
            ->end()
        ->end()
    ;

    return $node;
}
```

## Node Path Separator

```php
$treeBuilder = new TreeBuilder('database');
// Default path: 'database.connection.driver'

$treeBuilder->setPathSeparator('/');
// Custom path: 'database/connection/driver'
```

## Processing Configuration

```php
use Symfony\Component\Config\Definition\Processor;
use Symfony\Component\Yaml\Yaml;

$config = Yaml::parse(file_get_contents('config.yaml'));
$extraConfig = Yaml::parse(file_get_contents('config_extra.yaml'));

$processor = new Processor();
$databaseConfiguration = new DatabaseConfiguration();
$processedConfiguration = $processor->processConfiguration(
    $databaseConfiguration,
    [$config, $extraConfig]
);
```

The processor merges multiple config arrays and validates them against the tree definition. The top-level key (extension name) must already be stripped from input arrays.

## Locating Resources

### FileLocator

```php
use Symfony\Component\Config\FileLocator;

$configDirectories = [__DIR__.'/config'];
$fileLocator = new FileLocator($configDirectories);

// Find first match
$yamlFile = $fileLocator->locate('users.yaml');

// Find first match, searching current path first
$yamlFile = $fileLocator->locate('users.yaml', __DIR__);

// Find all matches
$yamlFiles = $fileLocator->locate('users.yaml', null, false);
```

**`locate()` parameters:**
1. File name to find
2. Current path (optional) - searched first
3. Return first result only (true, default) or all results (false)

## Resource Loaders

### Creating a Custom Loader

```php
namespace Acme\Config\Loader;

use Symfony\Component\Config\Loader\FileLoader;
use Symfony\Component\Yaml\Yaml;

class YamlUserLoader extends FileLoader
{
    public function load($resource, $type = null): void
    {
        $configValues = Yaml::parse(file_get_contents($resource));

        // ... handle the config values

        // Import other resources:
        // $this->import('extra_users.yaml');
    }

    public function supports($resource, $type = null): bool
    {
        return is_string($resource) && 'yaml' === pathinfo(
            $resource,
            PATHINFO_EXTENSION
        );
    }
}
```

**Key methods:**
- `load($resource, $type)` - Load and process the resource
- `supports($resource, $type)` - Check if loader handles this resource type
- `import($resource)` - Recursively import other resources (from FileLoader)

## Loader Resolution

### LoaderResolver and DelegatingLoader

```php
use Symfony\Component\Config\Loader\DelegatingLoader;
use Symfony\Component\Config\Loader\LoaderResolver;

$loaderResolver = new LoaderResolver([
    new YamlUserLoader($fileLocator),
    new XmlUserLoader($fileLocator),
]);
$delegatingLoader = new DelegatingLoader($loaderResolver);

// Automatically selects YamlUserLoader based on file extension
$delegatingLoader->load(__DIR__.'/users.yaml');
```

The `LoaderResolver` iterates through registered loaders and returns the first one whose `supports()` method returns `true`. The `DelegatingLoader` delegates loading to the resolver.

## Caching Configuration

### ConfigCache

```php
use Symfony\Component\Config\ConfigCache;
use Symfony\Component\Config\Resource\FileResource;

$cachePath = __DIR__.'/cache/appUserMatcher.php';

// Second argument enables debug mode (resource tracking)
$cache = new ConfigCache($cachePath, true);

if (!$cache->isFresh()) {
    $yamlUserFiles = $fileLocator->locate('users.yaml', null, false);

    $resources = [];
    foreach ($yamlUserFiles as $yamlUserFile) {
        $delegatingLoader->load($yamlUserFile);
        $resources[] = new FileResource($yamlUserFile);
    }

    $code = '// generated code...';
    $cache->write($code, $resources);
}

require $cachePath;
```

### Debug vs Production Mode

| Feature | Debug (`true`) | Production (`false`) |
|---------|---------------|---------------------|
| `.meta` file | Created with serialized resources | Not created |
| `isFresh()` | Compares timestamps | Always fresh once created |
| Overhead | Resource tracking | None |

### Custom Meta File Path (Symfony 7.1+)

```php
$cache = new ConfigCache($cachePath, true, '/my/absolute/path/to/cache.meta');
```

### Key Methods

| Method | Description |
|--------|-------------|
| `isFresh()` | Check if cache is still valid based on resource timestamps |
| `write($code, $resources)` | Write generated code and track resources |
| `getPath()` | Get the cache file path |

## Class Reference

| Class | Namespace |
|-------|-----------|
| `TreeBuilder` | `Symfony\Component\Config\Definition\Builder\TreeBuilder` |
| `ConfigurationInterface` | `Symfony\Component\Config\Definition\ConfigurationInterface` |
| `Processor` | `Symfony\Component\Config\Definition\Processor` |
| `FileLocator` | `Symfony\Component\Config\FileLocator` |
| `FileLoader` | `Symfony\Component\Config\Loader\FileLoader` |
| `LoaderInterface` | `Symfony\Component\Config\Loader\LoaderInterface` |
| `LoaderResolver` | `Symfony\Component\Config\Loader\LoaderResolver` |
| `DelegatingLoader` | `Symfony\Component\Config\Loader\DelegatingLoader` |
| `ConfigCache` | `Symfony\Component\Config\ConfigCache` |
| `FileResource` | `Symfony\Component\Config\Resource\FileResource` |
