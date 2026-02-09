---
name: "symfony-7-4-process"
description: "Symfony 7.4 Process component reference for executing commands in sub-processes. Use when working with Process, external commands, shell execution, async processes, timeout, signals, mustRun, fromShellCommandline, getOutput, getErrorOutput, isSuccessful, subprocess management, or any command execution-related Symfony code."
---

# Symfony 7.4 Process Component

## Overview

The Symfony Process component executes commands in sub-processes, providing a secure and cross-platform way to run external commands. It replaces PHP functions like `exec()`, `passthru()`, `shell_exec()`, and `system()` with proper argument escaping and OS difference handling.

## Quick Reference

### Installation

```bash
composer require symfony/process
```

### Basic Command Execution

```php
use Symfony\Component\Process\Exception\ProcessFailedException;
use Symfony\Component\Process\Process;

$process = new Process(['ls', '-lsa']);
$process->run();

if (!$process->isSuccessful()) {
    throw new ProcessFailedException($process);
}

echo $process->getOutput();
```

### Using mustRun() (throws on failure)

```php
use Symfony\Component\Process\Exception\ProcessFailedException;
use Symfony\Component\Process\Process;

$process = new Process(['ls', '-lsa']);

try {
    $process->mustRun();
    echo $process->getOutput();
} catch (ProcessFailedException $exception) {
    echo $exception->getMessage();
}
```

### Shell Command Execution

```php
use Symfony\Component\Process\Process;

// Using fromShellCommandline for shell features (pipes, redirects, etc.)
$process = Process::fromShellCommandline('echo "$MESSAGE"');
$process->run(null, ['MESSAGE' => 'Hello World']);

echo $process->getOutput();
```

### Asynchronous Execution

```php
use Symfony\Component\Process\Process;

$process = new Process(['long-running-command']);
$process->start();

// Do other work while process runs...

$process->wait(); // Wait for completion
echo $process->getOutput();
```

### Real-time Output with Callback

```php
use Symfony\Component\Process\Process;

$process = new Process(['ls', '-lsa']);
$process->run(function ($type, $buffer): void {
    if (Process::ERR === $type) {
        echo 'ERR > '.$buffer;
    } else {
        echo 'OUT > '.$buffer;
    }
});
```

### Process Timeout

```php
use Symfony\Component\Process\Process;

$process = new Process(['long-command']);
$process->setTimeout(3600);      // Total timeout: 60 minutes
$process->setIdleTimeout(60);    // No output for 60 seconds
$process->run();
```

### Sending Signals

```php
use Symfony\Component\Process\Process;

$process = new Process(['find', '/', '-name', 'rabbit']);
$process->start();

$process->signal(SIGKILL);
```

### Finding PHP Executable

```php
use Symfony\Component\Process\PhpExecutableFinder;

$phpBinaryFinder = new PhpExecutableFinder();
$phpBinaryPath = $phpBinaryFinder->find();
// e.g., '/usr/local/bin/php'
```

## Key Methods

| Method | Purpose |
|--------|---------|
| `run()` | Execute synchronously |
| `mustRun()` | Execute synchronously, throw on failure |
| `start()` | Start asynchronously |
| `wait()` | Wait for async process completion |
| `isRunning()` | Check if process is running |
| `isSuccessful()` | Check if process completed successfully |
| `getOutput()` | Get complete stdout |
| `getErrorOutput()` | Get complete stderr |
| `stop(timeout, signal)` | Terminate process |
| `signal(signal)` | Send signal to process |
| `setTimeout(seconds)` | Set total timeout |
| `setIdleTimeout(seconds)` | Set idle timeout |

## Full Documentation
- GitHub: https://github.com/symfony/process
- Docs: https://symfony.com/doc/7.4/components/process.html
- **Full Reference**: See [references/process.md](references/process.md) for complete documentation
