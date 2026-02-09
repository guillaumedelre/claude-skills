---
name: "symfony-7-4-finder"
description: "Symfony 7.4 Finder component reference for finding files and directories with an intuitive fluent interface. Use when searching for files or directories, filtering by name, content, size, date, depth, path, glob patterns, or sorting results. Triggers on: Finder, file search, directory search, glob, name filtering, content filtering, sorting, date filtering, size filtering, depth filtering, path filtering, SplFileInfo, ignoreVCS, followLinks."
---

# Symfony 7.4 Finder Component

GitHub: https://github.com/symfony/finder
Docs: https://symfony.com/doc/7.4/components/finder.html

## Quick Reference

### Basic Usage

```php
use Symfony\Component\Finder\Finder;

$finder = new Finder();
$finder->files()->in(__DIR__);

if ($finder->hasResults()) {
    foreach ($finder as $file) {
        $file->getRealPath();           // Absolute path
        $file->getRelativePathname();   // Relative path with filename
        $file->getContents();           // File contents
    }
}
```

### Common Patterns

```php
// Find PHP files, exclude vendor
$finder = new Finder();
$finder->files()->in(__DIR__)->name('*.php')->exclude('vendor');

// Find by content
$finder->files()->in(__DIR__)->contains('/lorem\s+ipsum$/i');

// Filter by size and date
$finder->files()->in(__DIR__)->size('>= 1K')->size('<= 1M')->date('since yesterday');

// Control depth
$finder->files()->in(__DIR__)->depth('== 0');  // Direct children only

// Sorting
$finder->files()->in(__DIR__)->sortByName();

// Multiple names (glob or regex)
$finder->files()->in(__DIR__)->name(['*.php', '*.twig']);

// Directories only
$finder->directories()->in(__DIR__)->depth('< 2');

// Custom filter
$finder->files()->in(__DIR__)->filter(function (\SplFileInfo $file) {
    return strlen($file->getFilename()) < 20;
});
```

### Important: Finder is Stateful

Clone the Finder before reusing with different criteria:

```php
foreach ((clone $finder)->name('*.php') as $file) { /* ... */ }
foreach ((clone $finder)->name('*.twig') as $file) { /* ... */ }
```

## Full Documentation

For complete details including all filtering methods, size/date operators, VCS handling, symbolic links, custom sorting, stream wrappers, directory pruning, and result transformation, see [references/finder.md](references/finder.md).
