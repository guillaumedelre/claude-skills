# Symfony 7.4 Filesystem Component - Complete Reference

GitHub: https://github.com/symfony/filesystem
Docs: https://symfony.com/doc/7.4/components/filesystem.html
License: MIT

## Overview

The Filesystem component provides platform-independent utilities for filesystem operations and file/directory path manipulation. It consists of two main classes: `Filesystem` for operations and `Path` for path manipulation.

## Installation

```bash
composer require symfony/filesystem
```

## Filesystem Utilities

All methods accept strings, arrays, or Traversable objects for file paths unless noted otherwise.

### mkdir

Creates a directory recursively with optional mode setting. Ignores already existing directories.

```php
$filesystem->mkdir('/tmp/photos', 0700);
$filesystem->mkdir(['/tmp/photos', '/tmp/videos']);
```

### exists

Checks for presence of one or more files/directories. Returns `false` if any are missing.

```php
$filesystem->exists('/tmp/photos');
$filesystem->exists(['rabbit.jpg', 'bottle.png']);
```

### copy

Copies a single file. Use `mirror()` for directories. By default, if the target file is newer, it is not overwritten.

```php
$filesystem->copy('image-ICC.jpg', 'image.jpg');
$filesystem->copy('image-ICC.jpg', 'image.jpg', true); // force override even if target is newer
```

### touch

Sets access and modification times on a file. Creates the file if it does not exist.

```php
$filesystem->touch('file.txt');
$filesystem->touch('file.txt', time() + 10);           // custom modification time
$filesystem->touch('file.txt', time(), time() - 10);    // custom modification and access time
```

### chown

Changes the owner of a file or directory.

```php
$filesystem->chown('lolcat.mp4', 'www-data');
$filesystem->chown('/video', 'www-data', true); // recursive
```

### chgrp

Changes the group of a file or directory.

```php
$filesystem->chgrp('lolcat.mp4', 'nginx');
$filesystem->chgrp('/video', 'nginx', true); // recursive
```

### chmod

Changes file mode/permissions.

```php
$filesystem->chmod('video.ogg', 0600);
$filesystem->chmod('src', 0700, 0000, true); // mode, umask, recursive
```

### remove

Deletes files, directories, and symlinks.

```php
$filesystem->remove(['symlink', '/path/to/directory', 'activity.log']);
```

### rename

Renames/moves a file or directory. Third parameter allows overwriting existing target.

```php
$filesystem->rename('/tmp/processed_video.ogg', '/path/to/store/video_647.ogg');
$filesystem->rename('/tmp/files', '/path/to/store/files');
$filesystem->rename('/tmp/file.ogg', '/path/to/file.ogg', true); // overwrite if exists
```

### symlink

Creates a symbolic link. If the third parameter is `true`, falls back to copying on systems that don't support symlinks.

```php
$filesystem->symlink('/path/to/source', '/path/to/destination');
$filesystem->symlink('/path/to/source', '/path/to/destination', true); // copy fallback
```

### readlink

Reads the target of a link. With `true` as second parameter, canonicalizes the path (resolves all symlinks).

```php
$filesystem->readlink('/path/to/link');
$filesystem->readlink('/path/to/link', true); // canonicalize
```

### makePathRelative

Returns the relative path from one absolute path to another.

```php
$filesystem->makePathRelative(
    '/var/lib/symfony/src/Symfony/',
    '/var/lib/symfony/src/Symfony/Component'
);
// => ../
```

### mirror

Copies all contents of a source directory into a target directory.

```php
$filesystem->mirror('/path/to/source', '/path/to/target');
```

### isAbsolutePath

Returns `true` if the given path is absolute.

```php
$filesystem->isAbsolutePath('/tmp');          // true
$filesystem->isAbsolutePath('c:\\Windows');   // true
$filesystem->isAbsolutePath('tmp');           // false
$filesystem->isAbsolutePath('../dir');        // false
```

### tempnam

Creates a temporary file with a unique name. Optional third parameter sets a suffix/extension.

```php
$filesystem->tempnam('/tmp', 'prefix_');
$filesystem->tempnam('/tmp', 'prefix_', '.png');
```

### dumpFile

Saves content to a file atomically (writes to temp file first, then renames).

```php
$filesystem->dumpFile('file.txt', 'Hello World');
```

### appendToFile

Appends content to the end of a file. Creates the file if it does not exist.

```php
$filesystem->appendToFile('logs.txt', 'Email sent to user@example.com');
$filesystem->appendToFile('logs.txt', 'Email sent to user@example.com', true); // with lock
```

### readFile

Returns file contents as a string. Available since Symfony 7.1.

```php
$contents = $filesystem->readFile('/some/path/to/file.txt');
```

## Path Manipulation Utilities

The `Path` class provides static methods for cross-platform path manipulation.

```php
use Symfony\Component\Filesystem\Path;
```

### canonicalize

Converts a path to its shortest equivalent form. Resolves `..` and `.` segments, normalizes separators to `/`.

```php
Path::canonicalize('/var/www/vhost/webmozart/../config.ini');
// => /var/www/vhost/config.ini
```

### join

Concatenates path segments with proper separators.

```php
Path::join('/var/www', 'vhost', 'config.ini');
// => /var/www/vhost/config.ini
```

### makeAbsolute

Converts a relative path to an absolute path using the given base directory.

```php
Path::makeAbsolute('config/config.yaml', '/var/www/project');
// => /var/www/project/config/config.yaml
```

### makeRelative

Converts an absolute path to a relative path based on the given base directory.

```php
Path::makeRelative('/var/www/project/config/config.yaml', '/var/www/project');
// => config/config.yaml
```

### isAbsolute / isRelative

Check whether a path is absolute or relative.

```php
Path::isAbsolute('C:\Programs\PHP\php.ini'); // true
Path::isAbsolute('/tmp');                     // true
Path::isRelative('config/config.yaml');       // true
```

### getLongestCommonBasePath

Finds the longest common base path among multiple paths.

```php
$basePath = Path::getLongestCommonBasePath(
    '/var/www/vhosts/project/httpdocs/config/config.yaml',
    '/var/www/vhosts/project/httpdocs/config/routing.yaml'
);
// => /var/www/vhosts/project/httpdocs/config
```

### isBasePath

Tests whether one path is a base path of another.

```php
Path::isBasePath('/var/www', '/var/www/project');     // true
Path::isBasePath('/var/www', '/var/www/project/..'); // true
Path::isBasePath('/var/www', '/tmp');                  // false
```

### getDirectory

Returns the directory portion of a path. Corrects PHP's `dirname()` shortcomings.

```php
Path::getDirectory('C:\Programs');
// => C:/

Path::getDirectory('/etc/apache2/sites-available');
// => /etc/apache2
```

### getRoot

Returns the root of a path.

```php
Path::getRoot('/etc/apache2/sites-available');
// => /

Path::getRoot('C:\Programs');
// => C:/
```

## Error Handling

All filesystem operations throw exceptions that implement `ExceptionInterface` or `IOExceptionInterface`. The most common exception is `IOException`.

```php
use Symfony\Component\Filesystem\Exception\IOExceptionInterface;
use Symfony\Component\Filesystem\Exception\IOException;

try {
    $filesystem->mkdir(sys_get_temp_dir().'/test');
} catch (IOExceptionInterface $exception) {
    echo "An error occurred while creating directory at ".$exception->getPath();
}
```

### Exception Hierarchy

- `ExceptionInterface` - base interface for all filesystem exceptions
- `IOExceptionInterface extends ExceptionInterface` - for I/O related exceptions
- `IOException implements IOExceptionInterface` - general I/O exception with `getPath()` method
- `FileNotFoundException extends IOException` - when a file is not found
- `InvalidArgumentException` - for invalid arguments passed to methods
