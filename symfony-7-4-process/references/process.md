# Symfony 7.4 Process Component - Complete Reference

## Overview

The Process component executes commands in sub-processes, handling OS differences and argument escaping to prevent security issues. It replaces PHP functions like `exec()`, `passthru()`, `shell_exec()`, and `system()`.

**GitHub**: https://github.com/symfony/process
**Documentation**: https://symfony.com/doc/7.4/components/process.html
**License**: MIT

## Installation

```bash
composer require symfony/process
```

## Basic Usage

### Simple Command Execution

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

### Getting Output

```php
$process->getOutput();                    // Complete stdout
$process->getErrorOutput();               // Complete stderr
$process->getIncrementalOutput();         // New output since last call
$process->getIncrementalErrorOutput();    // New error output since last call
$process->clearOutput();                  // Clear output buffer
$process->clearErrorOutput();             // Clear error buffer
```

### Using Iterators

```php
$process = new Process(['ls', '-lsa']);
$process->start();

foreach ($process as $type => $data) {
    if ($process::OUT === $type) {
        echo "\nRead from stdout: ".$data;
    } else { // $process::ERR === $type
        echo "\nRead from stderr: ".$data;
    }
}
```

### Using mustRun()

The `mustRun()` method is identical to `run()`, except that it throws a `ProcessFailedException` if the process did not exit successfully:

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

## Configuring Process Options

### Setting PHP proc_open Options

```php
$process = new Process(['...', '...', '...']);
$process->setOptions(['create_new_console' => true]);
```

## Shell Command Execution

### Using Shell Command Lines

The recommended approach is using arrays (safer):

```php
$process = new Process(['/path/command', '--option', 'argument']);
```

For shell features (pipes, redirects, etc.), use `fromShellCommandline()`:

```php
$process = Process::fromShellCommandline('echo "$MESSAGE"');
```

### Environment Variables in Shell Commands

```php
// Unix-like OSes
$process = Process::fromShellCommandline('echo "$MESSAGE"');

// Windows
$process = Process::fromShellCommandline('echo "!MESSAGE!"');

// Cross-platform
$process = Process::fromShellCommandline('echo "${:MESSAGE}"');
$process->run(null, ['MESSAGE' => 'Something to output']);
```

## Setting Environment Variables

```php
// Via constructor
$process = new Process(['...'], null, ['ENV_VAR_NAME' => 'value']);

// Via factory method
$process = Process::fromShellCommandline('...', null, ['ENV_VAR_NAME' => 'value']);

// Via run method
$process->run(null, ['ENV_VAR_NAME' => 'value']);

// Remove environment variables
$process = new Process(['...'], null, [
    'APP_ENV' => false,
    'SYMFONY_DOTENV_VARS' => false,
]);
```

## Real-time Output Processing

### Callback on Run

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

## Asynchronous Process Execution

### Basic Async Execution

```php
$process = new Process(['ls', '-lsa']);
$process->start();

while ($process->isRunning()) {
    // waiting for process to finish
}

echo $process->getOutput();
```

### Waiting for Process Completion

```php
$process = new Process(['ls', '-lsa']);
$process->start();

// ... do other things

$process->wait();

// ... do things after process finishes
```

### Waiting with Callback

```php
$process = new Process(['ls', '-lsa']);
$process->start();

$process->wait(function ($type, $buffer): void {
    if (Process::ERR === $type) {
        echo 'ERR > '.$buffer;
    } else {
        echo 'OUT > '.$buffer;
    }
});
```

### Conditional Waiting

```php
$process = new Process(['/usr/bin/php', 'slow-starting-server.php']);
$process->start();

// waits until the given condition is met
$process->waitUntil(function ($type, $output): bool {
    return $output === 'Ready. Waiting for commands...';
});

// ... do things after process is ready
```

## Standard Input Streaming

### Setting Input Before Start

```php
$process = new Process(['cat']);
$process->setInput('foobar');
$process->run();
```

### Writing to Input While Running

```php
use Symfony\Component\Process\InputStream;

$input = new InputStream();
$input->write('foo');

$process = new Process(['cat']);
$process->setInput($input);
$process->start();

// ... read process output or do other things

$input->write('bar');
$input->close();

$process->wait();

echo $process->getOutput(); // outputs: foobar
```

### Using PHP Streams

```php
$stream = fopen('php://temporary', 'w+');

$process = new Process(['cat']);
$process->setInput($stream);
$process->start();

fwrite($stream, 'foo');

// ... read process output

fwrite($stream, 'bar');
fclose($stream);

$process->wait();

echo $process->getOutput(); // outputs: 'foobar'
```

## TTY and PTY Modes

### TTY Mode (Terminal)

```php
$process = new Process(['vim']);
$process->setTty(true);
$process->run();

// Output is connected to terminal, cannot be read
dump($process->getOutput()); // null
```

### PTY Mode (Pseudo-Terminal)

```php
$process = new Process(['some-command']);
$process->setPty(true);
$process->run();
```

### Checking TTY Support

```php
use Symfony\Component\Process\Process;

$process = (new Process())->setTty(Process::isTtySupported());
```

## Stopping Processes

```php
$process = new Process(['find', '/', '-name', 'rabbit']);
$process->start();

// ... do other things

$process->stop(3, SIGINT); // timeout 3 seconds, then SIGINT signal
```

## Executing PHP Code

### Isolated PHP Execution

```php
use Symfony\Component\Process\PhpProcess;

$process = new PhpProcess(<<<EOF
    <?= 'Hello World' ?>
EOF
);
$process->run();
```

### Child Process with Same Configuration

```php
use Symfony\Component\Process\PhpSubprocess;

$childProcess = new PhpSubprocess(['bin/console', 'cache:pool:prune']);
$childProcess->run();
```

## Process Timeouts

### Total Timeout

```php
use Symfony\Component\Process\Process;

$process = new Process(['ls', '-lsa']);
$process->setTimeout(3600); // 60 minutes
$process->run();

// throws ProcessTimedOutException if timeout reached
```

### Manual Timeout Checking

```php
$process->setTimeout(3600);
$process->start();

while ($condition) {
    // ...

    $process->checkTimeout();
    usleep(200000);
}
```

### Idle Timeout

```php
$process = new Process(['something-with-variable-runtime']);
$process->setTimeout(3600);      // Total timeout
$process->setIdleTimeout(60);    // No output for 60 seconds
$process->run();
```

## Process Signals

### Sending Signals

```php
use Symfony\Component\Process\Process;

$process = new Process(['find', '/', '-name', 'rabbit']);
$process->start();

$process->signal(SIGKILL);
```

### Ignoring Signals

```php
use Symfony\Component\Process\Process;

$process = new Process(['find', '/', '-name', 'rabbit']);
$process->setIgnoredSignals([SIGKILL, SIGUSR1]);
$process->run();
```

## Process PID

```php
use Symfony\Component\Process\Process;

$process = new Process(['/usr/bin/php', 'worker.php']);
$process->start();

$pid = $process->getPid();
```

## Output Management

### Disabling Output (Memory Optimization)

```php
use Symfony\Component\Process\Process;

$process = new Process(['/usr/bin/php', 'worker.php']);
$process->disableOutput();
$process->run();

// Later, if needed
$process->enableOutput();
```

**Note:** Cannot enable/disable while process is running. With disabled output, `getOutput()`, `getErrorOutput()`, and `setIdleTimeout()` are unavailable.

## Finding Executables

### Finding Any Executable

```php
use Symfony\Component\Process\ExecutableFinder;

$executableFinder = new ExecutableFinder();
$chromedriverPath = $executableFinder->find('chromedriver');

// With defaults and extra directories
$chromedriverPath = $executableFinder->find(
    'chromedriver',
    '/path/to/chromedriver',
    ['local-bin/']
);
```

### Finding PHP Binary

```php
use Symfony\Component\Process\PhpExecutableFinder;

$phpBinaryFinder = new PhpExecutableFinder();
$phpBinaryPath = $phpBinaryFinder->find();
// e.g., '/usr/local/bin/php'
```

## Key Classes

### Process

The main class for executing commands:

- `run(?callable $callback = null, array $env = []): int` - Execute synchronously
- `mustRun(?callable $callback = null, array $env = []): static` - Execute synchronously, throw on failure
- `start(?callable $callback = null, array $env = []): void` - Start asynchronously
- `wait(?callable $callback = null): int` - Wait for completion
- `waitUntil(callable $callback): bool` - Wait until condition is met
- `isRunning(): bool` - Check if running
- `isSuccessful(): bool` - Check if completed successfully
- `getOutput(): string` - Get stdout
- `getErrorOutput(): string` - Get stderr
- `getIncrementalOutput(): string` - Get new stdout since last call
- `getIncrementalErrorOutput(): string` - Get new stderr since last call
- `clearOutput(): static` - Clear output buffer
- `clearErrorOutput(): static` - Clear error buffer
- `stop(float $timeout = 10, int $signal = null): ?int` - Stop the process
- `signal(int $signal): static` - Send signal
- `setTimeout(?float $timeout): static` - Set total timeout
- `setIdleTimeout(?float $timeout): static` - Set idle timeout
- `setTty(bool $tty): static` - Enable TTY mode
- `setPty(bool $bool): static` - Enable PTY mode
- `setInput(mixed $input): static` - Set stdin input
- `setOptions(array $options): static` - Set proc_open options
- `setEnv(array $env): static` - Set environment variables
- `setIgnoredSignals(array $signals): static` - Set signals to ignore
- `disableOutput(): static` - Disable output capture
- `enableOutput(): static` - Enable output capture
- `getPid(): ?int` - Get process ID
- `getExitCode(): ?int` - Get exit code
- `getExitCodeText(): ?string` - Get exit code as text
- `getCommandLine(): string` - Get command line
- `checkTimeout(): void` - Check for timeout (throws if timed out)
- `static fromShellCommandline(string $command, ...): static` - Create from shell command
- `static isTtySupported(): bool` - Check TTY support
- `static isPtySupported(): bool` - Check PTY support

### PhpProcess

Execute PHP code in isolation:

```php
use Symfony\Component\Process\PhpProcess;

$process = new PhpProcess('<?php echo "Hello"; ?>');
$process->run();
```

### PhpSubprocess

Create child PHP process with same configuration:

```php
use Symfony\Component\Process\PhpSubprocess;

$process = new PhpSubprocess(['bin/console', 'command']);
$process->run();
```

### ExecutableFinder

Find executables in system PATH:

```php
use Symfony\Component\Process\ExecutableFinder;

$finder = new ExecutableFinder();
$path = $finder->find('node');
```

### PhpExecutableFinder

Find the PHP binary:

```php
use Symfony\Component\Process\PhpExecutableFinder;

$finder = new PhpExecutableFinder();
$phpPath = $finder->find();
```

### InputStream

Stream input to running processes:

```php
use Symfony\Component\Process\InputStream;

$input = new InputStream();
$process->setInput($input);
$process->start();

$input->write('data');
$input->close();
```

## Exceptions

- `ProcessFailedException` - Thrown by `mustRun()` when process fails
- `ProcessTimedOutException` - Thrown when process times out
- `ProcessSignaledException` - Thrown when process is terminated by signal
- `RuntimeException` - General runtime errors
- `LogicException` - Logic errors (e.g., calling methods in wrong state)

## Best Practices

1. **Use arrays instead of shell commands** when possible for security
2. **Set timeouts** to prevent runaway processes
3. **Use mustRun()** when process failure should stop execution
4. **Use callbacks** for long-running processes to provide feedback
5. **Disable output** for background workers to save memory
6. **Use PhpSubprocess** for child PHP processes to inherit configuration
7. **Check isSuccessful()** before assuming output is valid
