---
name: "symfony-7-4-asset"
description: "Symfony 7.4 Asset component reference for managing web asset URLs and versioning. Use when working with asset versioning, CDN configuration, asset packages, static file URL generation, or cache busting. Triggers on: Asset component, asset versioning, CDN, asset packages, JsonManifestVersionStrategy, static assets, UrlPackage, PathPackage, VersionStrategy, Packages class."
---

# Symfony 7.4 Asset Component

GitHub: https://github.com/symfony/asset
Docs: https://symfony.com/doc/7.4/components/asset.html

## Quick Reference

### Installation

```bash
composer require symfony/asset
```

### Basic Package with Versioning

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\StaticVersionStrategy;

$package = new Package(new StaticVersionStrategy('v1'));

echo $package->getUrl('/image.png');  // /image.png?v1
```

### JSON Manifest (Webpack / Vite)

```php
use Symfony\Component\Asset\Package;
use Symfony\Component\Asset\VersionStrategy\JsonManifestVersionStrategy;

$package = new Package(new JsonManifestVersionStrategy(__DIR__.'/rev-manifest.json'));

echo $package->getUrl('css/app.css');
// build/css/app.b916426ea1d10021f3f17ce8031f93c2.css
```

### Path-Based Package

```php
use Symfony\Component\Asset\PathPackage;
use Symfony\Component\Asset\VersionStrategy\StaticVersionStrategy;

$package = new PathPackage('/static/images', new StaticVersionStrategy('v1'));

echo $package->getUrl('logo.png');  // /static/images/logo.png?v1
```

### CDN / URL Package

```php
use Symfony\Component\Asset\UrlPackage;
use Symfony\Component\Asset\VersionStrategy\StaticVersionStrategy;

$package = new UrlPackage(
    'https://cdn.example.com/assets/',
    new StaticVersionStrategy('v1')
);

echo $package->getUrl('/logo.png');  // https://cdn.example.com/assets/logo.png?v1
```

### Multiple CDNs

```php
$urls = [
    'https://static1.example.com/',
    'https://static2.example.com/',
];
$package = new UrlPackage($urls, new StaticVersionStrategy('v1'));
```

### Named Packages

```php
use Symfony\Component\Asset\Packages;

$packages = new Packages(
    new Package(new StaticVersionStrategy('v1')),
    [
        'img' => new UrlPackage('https://img.example.com/', new StaticVersionStrategy('v1')),
        'doc' => new PathPackage('/documents', new StaticVersionStrategy('v1')),
    ]
);

echo $packages->getUrl('/main.css');             // /main.css?v1
echo $packages->getUrl('/logo.png', 'img');      // https://img.example.com/logo.png?v1
```

## Full Documentation

For complete details including all version strategies, custom strategies, request context integration, remote manifests, strict mode, and protocol support, see [references/asset.md](references/asset.md).
