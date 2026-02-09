# Symfony 7.4 Finder Component - Complete Reference

GitHub: https://github.com/symfony/finder
Docs: https://symfony.com/doc/7.4/components/finder.html

## Installation

```bash
composer require symfony/finder
```

## Core Classes

- `Symfony\Component\Finder\Finder` - Main class, implements `\IteratorAggregate` and `\Countable`
- `Symfony\Component\Finder\SplFileInfo` - Extends PHP's `SplFileInfo` with `getRelativePath()`, `getRelativePathname()`, and `getContents()`
- `Symfony\Component\Finder\Glob` - Converts glob patterns to regex
- `Symfony\Component\Finder\Gitignore` - Parses `.gitignore` rules

## Basic Usage

```php
use Symfony\Component\Finder\Finder;

$finder = new Finder();
$finder->files()->in(__DIR__);

if ($finder->hasResults()) {
    // ...
}

foreach ($finder as $file) {
    $file->getRealPath();           // Absolute file path
    $file->getRelativePathname();   // Relative path with filename
    $file->getRelativePath();       // Relative directory path (no filename)
    $file->getContents();           // Read file contents
    $file->getFilename();           // Filename with extension
    $file->getExtension();          // File extension
}
```

### Stateful Behavior

The Finder is stateful. Clone before reusing with different criteria:

```php
$finder = new Finder();
$finder->files()->in('./templates');

foreach ((clone $finder)->name('partial_*') as $file) { /* ... */ }
foreach ((clone $finder)->name('plugin_*') as $file) { /* ... */ }
```

## Location (Mandatory)

```php
// Single directory
$finder->in(__DIR__);

// Multiple directories
$finder->in([__DIR__, '/elsewhere']);
$finder->in(__DIR__)->in('/elsewhere');

// Wildcard patterns (each must resolve to at least one directory)
$finder->in('src/Symfony/*/*/Resources');

// Exclude directories by name
$finder->in(__DIR__)->exclude('ruby');
$finder->in(__DIR__)->exclude(['ruby', 'python']);

// Ignore unreadable directories (instead of throwing exception)
$finder->ignoreUnreadableDirs()->in(__DIR__);
```

### Stream Wrappers

```php
// FTP
$finder->in('ftp://example.com/pub/');

// S3 (after registering stream wrapper)
$s3Client->registerStreamWrapper();
$finder->in('s3://bucket-name');
```

## Files or Directories

```php
$finder->files();         // Find files only
$finder->directories();   // Find directories only
```

### Symbolic Links

```php
$finder->files()->followLinks();
```

- Without `followLinks()`: returns direct files/links; does not traverse linked directories
- With `followLinks()`: follows directory links and includes their contents

## Version Control Files

```php
// Include VCS files (normally ignored by default)
$finder->ignoreVCS(false);

// Use .gitignore rules to exclude files/directories
$finder->ignoreVCSIgnored(true);
```

Note: `.gitignore` rules in subdirectories override parent rules. Finder starts from the search directory, not the repository root.

## Filtering by File Name

```php
// Glob pattern
$finder->files()->name('*.php');

// Regex pattern
$finder->files()->name('/\.php$/');

// Multiple names (OR logic)
$finder->files()->name('*.php')->name('*.twig');
$finder->files()->name(['*.php', '*.twig']);

// Exclude by name
$finder->files()->notName('*.rb');
$finder->files()->notName(['*.rb', '*.py']);
```

## Filtering by File Contents

```php
// String match
$finder->files()->contains('lorem ipsum');

// Regex match
$finder->files()->contains('/lorem\s+ipsum$/i');

// Exclude by content
$finder->files()->notContains('dolor sit amet');
```

## Filtering by Path

```php
// String match (matches anywhere in path)
$finder->path('data');        // Matches data/*.xml, data.xml, etc.
$finder->path('foo/bar');

// Regex match
$finder->path('/^foo\/bar/');

// Multiple paths (OR logic)
$finder->path('data')->path('foo/bar');
$finder->path(['data', 'foo/bar']);

// Exclude by path
$finder->notPath('other/dir');
$finder->notPath(['first/dir', 'other/dir']);
```

String to regex conversion: `dirname` becomes `/dirname/`, `a/b/c` becomes `/a\/b\/c/`.

## Filtering by File Size

```php
$finder->files()->size('< 1.5K');

// Size range
$finder->files()->size('>= 1K')->size('<= 2K');
$finder->files()->size(['>= 1K', '<= 2K']);
```

### Size Operators

`>`, `>=`, `<`, `<=`, `==`, `!=`

### Size Units

| Suffix | Meaning |
|--------|---------|
| `k` | Kilobytes (10^3) |
| `ki` | Kibibytes (2^10) |
| `m` | Megabytes (10^6) |
| `mi` | Mebibytes (2^20) |
| `g` | Gigabytes (10^9) |
| `gi` | Gibibytes (2^30) |

## Filtering by File Date

```php
$finder->date('since yesterday');

// Date range
$finder->date('>= 2018-01-01')->date('<= 2018-12-31');
$finder->date(['>= 2018-01-01', '<= 2018-12-31']);
```

### Date Operators

- `>`, `>=`, `<`, `<=`, `==`
- Aliases: `since` and `after` for `>`, `until` and `before` for `<`
- Values: any string supported by PHP's `strtotime()`

## Filtering by Directory Depth

```php
$finder->depth('== 0');    // Direct children only
$finder->depth('< 3');     // Max 3 levels deep

// Depth range
$finder->depth('> 2')->depth('< 5');
$finder->depth(['> 2', '< 5']);
```

## Custom Filtering

```php
// Simple filter
$finder->files()->filter(function (\SplFileInfo $file) {
    if (strlen($file) > 10) {
        return false;  // Exclude this file
    }
});
```

### Directory Pruning

When the filter callback accepts a second argument set to `true`, returning `false` for a directory prunes the entire subtree, improving performance:

```php
$finder->files()->filter(function (\SplFileInfo $file) {
    return true;  // Include file/directory
});
```

## Sorting

### Built-in Sort Methods

```php
$finder->sortByName();                     // strcmp (case-sensitive)
$finder->sortByName(true);                 // Natural sort order
$finder->sortByCaseInsensitiveName();      // strcasecmp
$finder->sortByCaseInsensitiveName(true);  // Case-insensitive natural sort
$finder->sortByExtension();
$finder->sortBySize();
$finder->sortByType();                     // Directories first, then files
$finder->sortByAccessedTime();
$finder->sortByChangedTime();
$finder->sortByModifiedTime();
```

### Custom Sorting

```php
$finder->sort(function (\SplFileInfo $a, \SplFileInfo $b): int {
    return strcmp($a->getRealPath(), $b->getRealPath());
});
```

### Reverse Sorting

```php
$finder->sortByName()->reverseSorting();
```

**Warning:** Sort methods load all matching elements into memory. This can be slow for very large result sets.

## Transforming Results

```php
// Convert to array
$files = iterator_to_array($finder);

// When using multiple in() calls, pass false to avoid key duplication
$files = iterator_to_array($finder, false);

// Count results
$count = iterator_count($finder);
```

## Complete Example

```php
use Symfony\Component\Finder\Finder;

$finder = new Finder();
$finder
    ->files()
    ->in(['/path/to/dir1', '/path/to/dir2'])
    ->exclude('node_modules')
    ->name('*.php')
    ->notName('test_*.php')
    ->path('src')
    ->notPath('vendor')
    ->size('>= 100K')
    ->size('<= 1M')
    ->date('>= 2018-01-01')
    ->depth('< 3')
    ->ignoreVCS(true)
    ->ignoreVCSIgnored(true)
    ->sortByName();

foreach ($finder as $file) {
    echo $file->getRelativePathname() . "\n";
}
```

## Finder Method Summary

| Method | Description |
|--------|-------------|
| `files()` | Restrict to files only |
| `directories()` | Restrict to directories only |
| `in($dirs)` | Set search directories (mandatory) |
| `exclude($dirs)` | Exclude directories by name |
| `name($pattern)` | Filter by filename (glob or regex) |
| `notName($pattern)` | Exclude by filename |
| `contains($pattern)` | Filter by file content |
| `notContains($pattern)` | Exclude by file content |
| `path($pattern)` | Filter by path |
| `notPath($pattern)` | Exclude by path |
| `size($expr)` | Filter by file size |
| `date($expr)` | Filter by modification date |
| `depth($expr)` | Filter by directory depth |
| `filter($closure)` | Custom filter callback |
| `followLinks()` | Follow symbolic links |
| `ignoreVCS($ignore)` | Ignore/include VCS files |
| `ignoreVCSIgnored($ignore)` | Use .gitignore rules |
| `ignoreUnreadableDirs($ignore)` | Skip unreadable directories |
| `sortByName($natural)` | Sort by name |
| `sortByCaseInsensitiveName($natural)` | Case-insensitive name sort |
| `sortByExtension()` | Sort by extension |
| `sortBySize()` | Sort by size |
| `sortByType()` | Sort by type (dirs first) |
| `sortByAccessedTime()` | Sort by access time |
| `sortByChangedTime()` | Sort by change time |
| `sortByModifiedTime()` | Sort by modification time |
| `sort($closure)` | Custom sort callback |
| `reverseSorting()` | Reverse sort order |
| `hasResults()` | Check if any results exist |
| `count()` | Count results |
| `getIterator()` | Get iterator |
| `append($iterator)` | Append results from another source |
