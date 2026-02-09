---
name: "symfony-7-4-filesystem"
description: "Symfony 7.4 Filesystem component reference for file and directory operations. Use when working with filesystem utilities, file operations, directory creation, copy, remove, chmod, chown, symlink, mirror, tempnam, dumpFile, appendToFile, readFile, or Path manipulation (canonicalize, join, makeAbsolute, makeRelative). Triggers on: Filesystem, file operations, directory creation, copy, remove, chmod, symlink, Path, dumpFile, appendToFile, readFile, mirror, makePathRelative."
---

# Symfony 7.4 Filesystem Component

GitHub: https://github.com/symfony/filesystem
Docs: https://symfony.com/doc/7.4/components/filesystem.html

## Quick Reference

### Installation

```bash
composer require symfony/filesystem
```

### Filesystem Operations

```php
use Symfony\Component\Filesystem\Filesystem;

$filesystem = new Filesystem();

// Create directory
$filesystem->mkdir('/tmp/photos', 0700);

// Check existence
$filesystem->exists('/tmp/photos');

// Copy file (use mirror() for directories)
$filesystem->copy('source.jpg', 'dest.jpg', true); // true = force override

// Remove files/directories/symlinks
$filesystem->remove(['symlink', '/path/to/dir', 'file.log']);

// Rename
$filesystem->rename('/tmp/old.txt', '/path/new.txt', true); // true = overwrite

// Symlink
$filesystem->symlink('/path/source', '/path/dest');

// Permissions & ownership
$filesystem->chmod('file.txt', 0600);
$filesystem->chown('file.txt', 'www-data', true); // recursive
$filesystem->chgrp('file.txt', 'nginx', true);

// Read/write files
$contents = $filesystem->readFile('/path/to/file.txt');
$filesystem->dumpFile('file.txt', 'Hello World'); // atomic write
$filesystem->appendToFile('logs.txt', 'New log entry');

// Mirror entire directory
$filesystem->mirror('/path/source', '/path/target');

// Temp file
$tmpFile = $filesystem->tempnam('/tmp', 'prefix_', '.png');
```

### Path Manipulation

```php
use Symfony\Component\Filesystem\Path;

// Canonicalize
Path::canonicalize('/var/www/../config.ini');
// => /var/config.ini

// Join paths
Path::join('/var/www', 'vhost', 'config.ini');
// => /var/www/vhost/config.ini

// Absolute/relative conversion
Path::makeAbsolute('config/config.yaml', '/var/www/project');
Path::makeRelative('/var/www/project/config.yaml', '/var/www/project');

// Check path type
Path::isAbsolute('/tmp'); // true
Path::isRelative('tmp'); // true

// Common base path
Path::getLongestCommonBasePath('/var/www/a.txt', '/var/www/b.txt');
// => /var/www

// Directory and root
Path::getDirectory('/var/www/file.txt'); // /var/www
Path::getRoot('/etc/apache2'); // /
```

### Error Handling

```php
use Symfony\Component\Filesystem\Exception\IOExceptionInterface;

try {
    $filesystem->mkdir('/path/to/dir');
} catch (IOExceptionInterface $exception) {
    echo "Error at ".$exception->getPath();
}
```

## Full Documentation

For complete details on all methods, parameters, edge cases, and the Path utility class, see [references/filesystem.md](references/filesystem.md).
