---
name: "symfony-7-4-config"
description: "Symfony 7.4 Config component reference for configuration loading, validation, and processing. Use when defining configuration trees, validating configuration values, loading resources from files, caching configuration, or working with TreeBuilder, FileLocator, or resource loaders. Triggers on: Config, configuration loading, configuration validation, TreeBuilder, resource loading, FileLocator, Definition, ConfigurationInterface, Processor, LoaderResolver, DelegatingLoader, ConfigCache, FileResource."
---

# Symfony 7.4 Config Component

GitHub: https://github.com/symfony/config
Docs: https://symfony.com/doc/7.4/components/config.html

## Quick Reference

### Defining Configuration with TreeBuilder

```php
use Symfony\Component\Config\Definition\Builder\TreeBuilder;
use Symfony\Component\Config\Definition\ConfigurationInterface;

class DatabaseConfiguration implements ConfigurationInterface
{
    public function getConfigTreeBuilder(): TreeBuilder
    {
        $treeBuilder = new TreeBuilder('database');

        $treeBuilder->getRootNode()
            ->children()
                ->scalarNode('driver')
                    ->isRequired()
                    ->cannotBeEmpty()
                ->end()
                ->scalarNode('host')
                    ->defaultValue('localhost')
                ->end()
                ->integerNode('port')
                    ->min(1)->max(65535)
                ->end()
                ->booleanNode('memory')
                    ->defaultFalse()
                ->end()
                ->enumNode('charset')
                    ->values(['utf8', 'utf8mb4', 'latin1'])
                ->end()
            ->end()
        ;

        return $treeBuilder;
    }
}
```

### Processing Configuration

```php
use Symfony\Component\Config\Definition\Processor;

$processor = new Processor();
$configuration = new DatabaseConfiguration();
$processedConfig = $processor->processConfiguration($configuration, [$config1, $config2]);
```

### Locating and Loading Resources

```php
use Symfony\Component\Config\FileLocator;
use Symfony\Component\Config\Loader\DelegatingLoader;
use Symfony\Component\Config\Loader\LoaderResolver;

$fileLocator = new FileLocator([__DIR__.'/config']);
$loaderResolver = new LoaderResolver([new YamlUserLoader($fileLocator)]);
$delegatingLoader = new DelegatingLoader($loaderResolver);
$delegatingLoader->load(__DIR__.'/config/users.yaml');
```

### Caching Configuration

```php
use Symfony\Component\Config\ConfigCache;
use Symfony\Component\Config\Resource\FileResource;

$cache = new ConfigCache($cachePath, $isDebug);

if (!$cache->isFresh()) {
    $resources = [new FileResource('config.yaml')];
    $cache->write($generatedCode, $resources);
}

require $cachePath;
```

### Validation Rules

```php
$rootNode
    ->children()
        ->scalarNode('driver')
            ->validate()
                ->ifNotInArray(['mysql', 'sqlite', 'mssql'])
                ->thenInvalid('Invalid database driver %s')
            ->end()
        ->end()
    ->end()
;
```

## Full Documentation

For complete details including all node types, array prototypes, normalization, deprecation, optional sections (canBeEnabled/canBeDisabled), custom normalization, merging strategies, appending sections, FileLoader, LoaderResolver, and ConfigCache with resource tracking, see [references/config.md](references/config.md).
