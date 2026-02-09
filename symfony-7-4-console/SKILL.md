---
name: "symfony-7-4-console"
description: "Symfony 7.4 Console component reference for building CLI commands. Use when creating, editing, testing, or debugging Symfony console commands, arguments, options, input/output styling, helpers (tables, progress bars, questions), or any CLI-related Symfony code. Triggers on: console commands, AsCommand attribute, SymfonyStyle, CommandTester, InputInterface, OutputInterface, console helpers, #[Argument], #[Option], #[MapInput], command lifecycle, console events, signal handling."
---

# Symfony 7.4 Console Component

GitHub: https://github.com/symfony/console
Docs: https://symfony.com/doc/7.4/console.html

## Installation

```bash
composer require symfony/console
```

## Quick Reference
### Create an Invokable Command (Symfony 7.3+)

```php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(name: 'app:greet')]
class GreetCommand
{
    public function __invoke(SymfonyStyle $io): int
    {
        $io->success('Hello!');
        return Command::SUCCESS;
    }
}
```

### Arguments and Options

```php
use Symfony\Component\Console\Attribute\Argument;
use Symfony\Component\Console\Attribute\Option;

public function __invoke(
    #[Argument(description: 'User name')] string $name,
    #[Option(shortcut: 'y')] bool $yell = false,
    SymfonyStyle $io
): int {
    $message = $yell ? strtoupper("Hello $name!") : "Hello $name!";
    $io->writeln($message);
    return Command::SUCCESS;
}
```

### SymfonyStyle Output

```php
$io->title('Command Title');
$io->section('Section');
$io->text('Regular text');
$io->success('Operation successful');
$io->error('An error occurred');
$io->table(['Header'], [['Cell']]);

// User input
$name = $io->ask('Your name?');
$confirmed = $io->confirm('Continue?');
$choice = $io->choice('Pick one', ['a', 'b', 'c']);
```

### Testing Commands

```php
use Symfony\Component\Console\Tester\CommandTester;

$command = $application->find('app:greet');
$tester = new CommandTester($command);

$tester->execute(['name' => 'John', '--yell' => true]);
$tester->assertCommandIsSuccessful();
$this->assertStringContainsString('HELLO JOHN', $tester->getDisplay());
```

## Documentation Structure

### Core Topics

- **[commands.md](references/commands.md)** - Command creation, arguments, options, DTOs, lifecycle
- **[output.md](references/output.md)** - SymfonyStyle, output sections, formatting, verbosity
- **[helpers.md](references/helpers.md)** - Table, ProgressBar, Question, Cursor, Tree helpers
- **[events.md](references/events.md)** - Console events, signal handling
- **[testing.md](references/testing.md)** - CommandTester, ApplicationTester, completion testing

### Advanced Topics

- **[advanced.md](references/advanced.md)** - Lockable commands, hidden commands, lazy loading, single-command apps, shell completion

## Common Patterns

### Command with Dependency Injection

```php
#[AsCommand(name: 'app:process')]
class ProcessCommand
{
    public function __construct(
        private UserRepository $users,
    ) {}

    public function __invoke(SymfonyStyle $io): int
    {
        foreach ($this->users->findAll() as $user) {
            $io->writeln($user->getName());
        }
        return Command::SUCCESS;
    }
}
```

### Progress Bar

```php
$io->progressStart(count($items));
foreach ($items as $item) {
    // Process item
    $io->progressAdvance();
}
$io->progressFinish();

// Or with progressIterate
foreach ($io->progressIterate($items) as $item) {
    // Process item
}
```

### Extended Command with Lifecycle

```php
#[AsCommand(name: 'app:process')]
class ProcessCommand extends Command
{
    protected function initialize(InputInterface $input, OutputInterface $output): void
    {
        // Initialize before interact/execute
    }

    protected function interact(InputInterface $input, OutputInterface $output): void
    {
        // Ask for missing input (skipped with --no-interaction)
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // Main logic
        return Command::SUCCESS;
    }
}
```

## Exit Codes

```php
return Command::SUCCESS; // 0
return Command::FAILURE; // 1
return Command::INVALID; // 2
```

## Global Options

All commands automatically support:
- `--help` (`-h`) - Display help
- `--verbose` (`-v`, `-vv`, `-vvv`) - Increase verbosity
- `--quiet` (`-q`) - Suppress output
- `--no-interaction` (`-n`) - Disable interaction
- `--ansi` / `--no-ansi` - Force/disable colors
- `--version` (`-V`) - Show version

## When to Read References

- **Creating commands with complex arguments/options** → Read [commands.md](references/commands.md)
- **Styling output or using SymfonyStyle** → Read [output.md](references/output.md)
- **Using tables, progress bars, or questions** → Read [helpers.md](references/helpers.md)
- **Handling signals or console events** → Read [events.md](references/events.md)
- **Writing tests for commands** → Read [testing.md](references/testing.md)
- **Shell completion, lockable commands, etc.** → Read [advanced.md](references/advanced.md)
