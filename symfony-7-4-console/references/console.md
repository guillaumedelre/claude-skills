# Symfony 7.4 Console Component - Complete Reference

## Overview

The Console component provides a framework for creating beautiful and testable command-line interfaces. It handles command registration, argument/option parsing, output formatting, and interactive input.

**GitHub**: https://github.com/symfony/console
**Documentation**: https://symfony.com/doc/7.4/console.html
**License**: MIT

## Installation

```bash
composer require symfony/console
```

## Creating Commands

### Invokable Commands (Recommended - Symfony 7.3+)

```php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(
    name: 'app:create-user',
    description: 'Creates a new user.',
    help: 'This command allows you to create a user...',
    aliases: ['app:add-user'],
)]
class CreateUserCommand
{
    public function __invoke(SymfonyStyle $io): int
    {
        $io->success('User created!');
        return Command::SUCCESS;
    }
}
```

### Extended Command Class

For lifecycle hooks and advanced features:

```php
namespace App\Command;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand(name: 'app:create-user')]
class CreateUserCommand extends Command
{
    protected function initialize(InputInterface $input, OutputInterface $output): void
    {
        // Initialize variables before interact() or execute()
    }

    protected function interact(InputInterface $input, OutputInterface $output): void
    {
        // Ask for missing arguments/options interactively
        // Not called with --no-interaction
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        // Main command logic
        return Command::SUCCESS;
    }
}
```

### Command Aliases

```php
#[AsCommand(name: 'app:create-user|app:add-user|app:new-user')]
```

### Dependency Injection

```php
use App\Service\UserManager;

#[AsCommand(name: 'app:create-user')]
class CreateUserCommand
{
    public function __construct(
        private UserManager $userManager,
    ) {
    }

    public function __invoke(SymfonyStyle $io): int
    {
        $this->userManager->create('john@example.com');
        return Command::SUCCESS;
    }
}
```

## Arguments

### Attribute-Based (Symfony 7.3+)

```php
use Symfony\Component\Console\Attribute\Argument;

public function __invoke(
    // Required argument (no default)
    #[Argument(description: 'The username')]
    string $username,

    // Optional argument (has default)
    #[Argument(description: 'Last name')]
    string $lastName = '',

    // Array argument (must be last)
    #[Argument(description: 'Additional names')]
    array $names = [],
): int {
    // ...
}
```

### Classic Configure Method

```php
use Symfony\Component\Console\Input\InputArgument;

protected function configure(): void
{
    $this
        ->addArgument('username', InputArgument::REQUIRED, 'The username')
        ->addArgument('lastName', InputArgument::OPTIONAL, 'Last name', '')
        ->addArgument('names', InputArgument::IS_ARRAY, 'Additional names');
}

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $username = $input->getArgument('username');
    $names = $input->getArgument('names');
    // ...
}
```

### Argument Modes

| Mode | Description |
|------|-------------|
| `InputArgument::REQUIRED` | Mandatory argument |
| `InputArgument::OPTIONAL` | Optional argument |
| `InputArgument::IS_ARRAY` | Multiple values (must be last) |

## Options

### Attribute-Based (Symfony 7.3+)

```php
use Symfony\Component\Console\Attribute\Option;

public function __invoke(
    #[Argument] string $name,

    // Value required (--iterations=5)
    #[Option(shortcut: 'i', description: 'Number of iterations')]
    int $iterations = 1,

    // Boolean flag (--yell)
    #[Option(shortcut: 'y')]
    bool $yell = false,

    // Negatable (--ansi / --no-ansi)
    #[Option]
    ?bool $ansi = null,

    // Array option (--role=ADMIN --role=USER)
    #[Option]
    array $roles = [],
): int {
    // ...
}
```

### Classic Configure Method

```php
use Symfony\Component\Console\Input\InputOption;

protected function configure(): void
{
    $this
        ->addOption('iterations', 'i', InputOption::VALUE_REQUIRED, 'Iterations', 1)
        ->addOption('yell', 'y', InputOption::VALUE_NONE, 'Yell the output')
        ->addOption('format', null, InputOption::VALUE_OPTIONAL, 'Output format', 'text')
        ->addOption('colors', null, InputOption::VALUE_NEGATABLE, 'Use colors')
        ->addOption('roles', null, InputOption::VALUE_IS_ARRAY | InputOption::VALUE_REQUIRED);
}
```

### Option Modes

| Mode | Description |
|------|-------------|
| `VALUE_NONE` | Boolean flag, no value |
| `VALUE_REQUIRED` | Value must be provided |
| `VALUE_OPTIONAL` | Value may be provided |
| `VALUE_NEGATABLE` | `--option` or `--no-option` |
| `VALUE_IS_ARRAY` | Multiple values |

### Options with Optional Values

```php
->addOption('yell', null, InputOption::VALUE_OPTIONAL, 'Yell?', false)

// In execute:
$optionValue = $input->getOption('yell');
if (false === $optionValue) {
    // Option NOT passed
} elseif (null === $optionValue) {
    // Option passed without value: --yell
} else {
    // Option passed with value: --yell=louder
}
```

## Input Mapping to DTOs (Symfony 7.4+)

```php
// src/Console/Input/CreateUserInput.php
namespace App\Console\Input;

use Symfony\Component\Console\Attribute\Argument;
use Symfony\Component\Console\Attribute\Option;

class CreateUserInput
{
    #[Argument]
    public string $email;

    #[Argument]
    public string $password;

    #[Option]
    public bool $admin = false;

    #[Option]
    public array $roles = [];
}

// Command
use Symfony\Component\Console\Attribute\MapInput;

#[AsCommand(name: 'app:create-user')]
class CreateUserCommand
{
    public function __invoke(#[MapInput] CreateUserInput $input): int
    {
        $email = $input->email;
        $isAdmin = $input->admin;
        return Command::SUCCESS;
    }
}
```

### Nested DTOs

```php
class PaginationInput
{
    #[Option] public int $limit = 10;
    #[Option] public int $page = 1;
}

class ListUsersInput
{
    #[Option] public ?string $role = null;
    #[MapInput] public PaginationInput $pagination;
}
```

### Property Hooks (PHP 8.4+)

```php
class CreateUserInput
{
    #[Argument]
    public string $email {
        set(string $value) {
            $this->email = strtolower(trim($value));
        }
    }
}
```

## SymfonyStyle Output

### Basic Usage

```php
use Symfony\Component\Console\Style\SymfonyStyle;

public function __invoke(SymfonyStyle $io): int
{
    // Titles
    $io->title('Main Title');
    $io->section('Section Title');

    // Text
    $io->text('Regular text');
    $io->text(['Line 1', 'Line 2', 'Line 3']);

    // Lists
    $io->listing(['Item 1', 'Item 2', 'Item 3']);

    // Tables
    $io->table(
        ['Header 1', 'Header 2'],
        [
            ['Cell 1-1', 'Cell 1-2'],
            ['Cell 2-1', 'Cell 2-2'],
        ]
    );
    $io->horizontalTable(['Header 1', 'Header 2'], [['Value 1', 'Value 2']]);

    // Definition list
    $io->definitionList(
        'Title',
        ['key1' => 'value1'],
        ['key2' => 'value2'],
    );

    // Tree (Symfony 7.3+)
    $io->tree([
        'src' => [
            'Controller' => ['DefaultController.php'],
            'Kernel.php',
        ],
    ]);

    // Blank lines
    $io->newLine(2);

    return Command::SUCCESS;
}
```

### Admonitions

```php
$io->note('This is a note');
$io->caution('This is a caution');
$io->warning('This is a warning');
$io->error('This is an error');
$io->success('This is a success');
$io->info('This is info');
```

### Progress Bars

```php
// Simple progress
$io->progressStart(100);
for ($i = 0; $i < 100; $i++) {
    $io->progressAdvance();
}
$io->progressFinish();

// Iterate with progress
foreach ($io->progressIterate($items) as $item) {
    // Process item
}

// Advanced progress bar
$progressBar = $io->createProgressBar(100);
$progressBar->setFormat(' %current%/%max% [%bar%] %percent:3s%% %elapsed:6s%/%estimated:-6s%');
$progressBar->start();
$progressBar->advance(10);
$progressBar->finish();
```

### User Input

```php
// Ask question
$name = $io->ask('What is your name?');
$name = $io->ask('What is your name?', 'John');

// With validation
$number = $io->ask('Enter a number', '1', function ($value) {
    if (!is_numeric($value)) {
        throw new \RuntimeException('Must be a number');
    }
    return (int) $value;
});

// Hidden input (passwords)
$password = $io->askHidden('Password?');

// Confirmation
$confirmed = $io->confirm('Continue?', false);

// Choice
$color = $io->choice('Pick a color', ['red', 'green', 'blue']);
$colors = $io->choice('Pick colors', ['red', 'green', 'blue'], multiSelect: true);
```

### Error Output

```php
// Write to stderr
$io->getErrorStyle()->warning('Debug information');
```

## Output Sections

```php
use Symfony\Component\Console\Output\ConsoleOutputInterface;

public function __invoke(OutputInterface $output): int
{
    if (!$output instanceof ConsoleOutputInterface) {
        throw new \LogicException('ConsoleOutputInterface required');
    }

    $section1 = $output->section();
    $section2 = $output->section();

    $section1->writeln('Hello');
    $section2->writeln('World');

    sleep(1);
    $section1->overwrite('Goodbye');
    $section2->clear();

    // Scrolling region
    $section1->setMaxHeight(5);

    return Command::SUCCESS;
}
```

## Console Helpers

### Table Helper

```php
use Symfony\Component\Console\Helper\Table;

$table = new Table($output);
$table
    ->setHeaders(['ISBN', 'Title', 'Author'])
    ->setRows([
        ['99921-58-10-7', 'Divine Comedy', 'Dante Alighieri'],
        ['9971-5-0210-0', 'A Tale of Two Cities', 'Charles Dickens'],
    ])
    ->setStyle('box')  // default, box, borderless, compact, box-double
    ->render();

// Column configuration
$table->setColumnWidth(0, 15);
$table->setColumnMaxWidth(1, 30);
```

### Progress Bar Helper

```php
use Symfony\Component\Console\Helper\ProgressBar;

$progressBar = new ProgressBar($output, 100);
$progressBar->setFormat('debug');  // normal, verbose, very_verbose, debug, minimal
$progressBar->start();

for ($i = 0; $i < 100; $i++) {
    $progressBar->advance();
    usleep(50000);
}

$progressBar->finish();
$output->writeln('');

// Custom format
$progressBar->setFormat(' %current%/%max% [%bar%] %percent:3s%% %elapsed:6s%/%estimated:-6s% %memory:6s%');

// Custom placeholders
$progressBar->setMessage('Task', 'filename');
$progressBar->setFormat(' %current%/%max% -- %filename%');
```

### Question Helper

```php
use Symfony\Component\Console\Helper\QuestionHelper;
use Symfony\Component\Console\Question\Question;
use Symfony\Component\Console\Question\ConfirmationQuestion;
use Symfony\Component\Console\Question\ChoiceQuestion;

$helper = $this->getHelper('question');

// Simple question
$question = new Question('Enter name: ', 'default');
$name = $helper->ask($input, $output, $question);

// Confirmation
$question = new ConfirmationQuestion('Continue? (y/n) ', false);
$continue = $helper->ask($input, $output, $question);

// Choice
$question = new ChoiceQuestion('Select color:', ['red', 'green', 'blue'], 0);
$question->setErrorMessage('Color %s is invalid.');
$color = $helper->ask($input, $output, $question);

// Hidden input
$question = new Question('Password: ');
$question->setHidden(true);
$question->setHiddenFallback(false);
$password = $helper->ask($input, $output, $question);

// Autocomplete
$question = new Question('Enter bundle name: ');
$question->setAutocompleterValues(['AcmeDemoBundle', 'AsseticBundle']);
$bundle = $helper->ask($input, $output, $question);

// Validation
$question = new Question('Enter email: ');
$question->setValidator(function ($answer) {
    if (!filter_var($answer, FILTER_VALIDATE_EMAIL)) {
        throw new \RuntimeException('Invalid email');
    }
    return $answer;
});
$question->setMaxAttempts(3);
```

### Cursor Helper

```php
use Symfony\Component\Console\Cursor;

$cursor = new Cursor($output);

$cursor->moveUp(2);
$cursor->moveDown(2);
$cursor->moveLeft(5);
$cursor->moveRight(5);
$cursor->moveToColumn(10);
$cursor->moveToPosition(10, 5);
$cursor->savePosition();
$cursor->restorePosition();
$cursor->clearLine();
$cursor->clearLineAfter();
$cursor->clearOutput();
$cursor->clearScreen();
$cursor->show();
$cursor->hide();
```

### Tree Helper (Symfony 7.3+)

```php
use Symfony\Component\Console\Helper\TreeHelper;
use Symfony\Component\Console\Helper\TreeNode;

$tree = TreeHelper::createTree($output);
$root = new TreeNode('Root');
$child1 = new TreeNode('Child 1');
$child2 = new TreeNode('Child 2');
$root->addChild($child1);
$root->addChild($child2);
$child1->addChild(new TreeNode('Grandchild'));

$tree->setNode($root);
$tree->render();
```

### Formatter Helper

```php
$formatter = $this->getHelper('formatter');

// Section block
$formattedLine = $formatter->formatSection('SomeSection', 'Message');

// Error block
$errorMessages = ['Error!', 'Something went wrong'];
$formattedBlock = $formatter->formatBlock($errorMessages, 'error');

// Truncate
$truncated = $formatter->truncate('Very long message', 7); // "Very..."
```

### Process Helper

```php
use Symfony\Component\Console\Helper\ProcessHelper;
use Symfony\Component\Process\Process;

$helper = $this->getHelper('process');
$process = new Process(['ls', '-la']);
$helper->run($output, $process);

// With error message
$helper->run($output, $process, 'Something went wrong');

// Must succeed
$helper->mustRun($output, $process);
```

## Console Events

### Available Events

```php
use Symfony\Component\Console\ConsoleEvents;

ConsoleEvents::COMMAND   // Before command execution
ConsoleEvents::ERROR     // On exception
ConsoleEvents::TERMINATE // After execution (always)
ConsoleEvents::SIGNAL    // On signal interrupt
```

### Event Listener

```php
use Symfony\Component\Console\Event\ConsoleCommandEvent;
use Symfony\Component\Console\Event\ConsoleErrorEvent;
use Symfony\Component\Console\Event\ConsoleTerminateEvent;
use Symfony\Component\Console\Event\ConsoleSignalEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

class ConsoleEventListener
{
    #[AsEventListener(event: ConsoleEvents::COMMAND)]
    public function onCommand(ConsoleCommandEvent $event): void
    {
        $command = $event->getCommand();
        $output = $event->getOutput();
        $output->writeln('Running: ' . $command->getName());

        // Disable command
        $event->disableCommand();
    }

    #[AsEventListener(event: ConsoleEvents::ERROR)]
    public function onError(ConsoleErrorEvent $event): void
    {
        $error = $event->getError();
        $event->setExitCode(1);

        // Change exception
        $event->setError(new \Exception('New error'));
    }

    #[AsEventListener(event: ConsoleEvents::TERMINATE)]
    public function onTerminate(ConsoleTerminateEvent $event): void
    {
        $exitCode = $event->getExitCode();
        $event->setExitCode(0);
    }

    #[AsEventListener(event: ConsoleEvents::SIGNAL)]
    public function onSignal(ConsoleSignalEvent $event): void
    {
        if ($event->getHandlingSignal() === SIGINT) {
            $event->abortExit(); // Continue execution
        }
    }
}
```

### Signal Handling in Commands

```php
use Symfony\Component\Console\Event\ConsoleSignalEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsCommand(name: 'app:long-running')]
class LongRunningCommand
{
    private bool $shouldStop = false;

    #[AsEventListener(ConsoleSignalEvent::class)]
    public function handleSignal(ConsoleSignalEvent $event): void
    {
        if (in_array($event->getHandlingSignal(), [SIGINT, SIGTERM])) {
            $this->shouldStop = true;
            $event->setExitCode(0);
        }
    }

    public function __invoke(SymfonyStyle $io): int
    {
        while (!$this->shouldStop) {
            // Long running task
            sleep(1);
        }
        return Command::SUCCESS;
    }
}
```

## Testing Commands

### CommandTester

```php
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;
use Symfony\Component\Console\Tester\CommandTester;

class CreateUserCommandTest extends KernelTestCase
{
    public function testExecute(): void
    {
        self::bootKernel();
        $application = new Application(self::$kernel);

        $command = $application->find('app:create-user');
        $commandTester = new CommandTester($command);

        $commandTester->execute([
            'username' => 'John',
            '--admin' => true,
        ]);

        $commandTester->assertCommandIsSuccessful();
        $this->assertStringContainsString('John', $commandTester->getDisplay());
    }

    public function testInteractive(): void
    {
        $commandTester = new CommandTester($command);
        $commandTester->setInputs(['John', 'yes']);
        $commandTester->execute([]);
    }
}
```

### ApplicationTester

```php
use Symfony\Component\Console\Tester\ApplicationTester;

$application = new Application();
$application->setAutoExit(false);

$tester = new ApplicationTester($application);
$tester->run(['command' => 'app:create-user', 'username' => 'John']);
```

### CommandCompletionTester

```php
use Symfony\Component\Console\Tester\CommandCompletionTester;

$tester = new CommandCompletionTester($application->get('app:greet'));
$suggestions = $tester->complete(['']);
$this->assertSame(['Alice', 'Bob'], $suggestions);
```

## Shell Completion

### Enable Completion

```bash
# Bash
php bin/console completion bash | sudo tee /etc/bash_completion.d/console-events-terminate

# Current session only
source <(php bin/console completion bash)

# Zsh
php bin/console completion zsh | sudo tee /usr/share/zsh/site-functions/_console

# Fish
php bin/console completion fish | source
```

### Custom Completion Values

```php
use Symfony\Component\Console\Completion\CompletionInput;

public function __invoke(
    #[Argument(suggestedValues: ['Alice', 'Bob', 'Charlie'])]
    string $name,

    #[Option(suggestedValues: [self::class, 'suggestFormats'])]
    string $format = 'json',
): int {
    // ...
}

public static function suggestFormats(CompletionInput $input): array
{
    return ['json', 'xml', 'csv'];
}

// With service access (non-static)
public function suggestUsers(CompletionInput $input): array
{
    return $this->userRepository->findAllUsernames();
}
```

## Verbosity Levels

```php
use Symfony\Component\Console\Output\OutputInterface;

$output->writeln('Always shown');

if ($output->isQuiet()) { /* -q */ }
if ($output->isVerbose()) { /* -v */ }
if ($output->isVeryVerbose()) { /* -vv */ }
if ($output->isDebug()) { /* -vvv */ }

// Write only at specific verbosity
$output->writeln('Debug info', OutputInterface::VERBOSITY_DEBUG);
$output->writeln('Verbose info', OutputInterface::VERBOSITY_VERBOSE);
```

| Level | Option | Constant |
|-------|--------|----------|
| Quiet | `-q` | `VERBOSITY_QUIET` |
| Normal | (none) | `VERBOSITY_NORMAL` |
| Verbose | `-v` | `VERBOSITY_VERBOSE` |
| Very Verbose | `-vv` | `VERBOSITY_VERY_VERBOSE` |
| Debug | `-vvv` | `VERBOSITY_DEBUG` |

## Exit Codes

```php
use Symfony\Component\Console\Command\Command;

return Command::SUCCESS; // 0
return Command::FAILURE; // 1
return Command::INVALID; // 2
```

## Calling Other Commands

```php
use Symfony\Component\Console\Input\ArrayInput;

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $command = $this->getApplication()->find('app:other-command');

    $greetInput = new ArrayInput([
        'username' => 'John',
        '--yell' => true,
    ]);

    return $command->run($greetInput, $output);
}
```

## Lockable Commands

Prevent multiple instances:

```php
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Command\LockableTrait;

#[AsCommand(name: 'app:singleton')]
class SingletonCommand extends Command
{
    use LockableTrait;

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        if (!$this->lock()) {
            $output->writeln('Command already running');
            return Command::SUCCESS;
        }

        try {
            // ... do work
        } finally {
            $this->release();
        }

        return Command::SUCCESS;
    }
}
```

## Hidden Commands

```php
#[AsCommand(name: 'app:internal', hidden: true)]
class InternalCommand extends Command
{
    // Not shown in command list
}
```

## Lazy Commands

Commands loaded only when needed (automatic with `#[AsCommand]`):

```php
#[AsCommand(name: 'app:heavy', description: 'Heavy command')]
class HeavyCommand extends Command
{
    // Heavy dependencies only loaded when command is actually run
}
```

## Single-Command Applications

```php
use Symfony\Component\Console\SingleCommandApplication;

(new SingleCommandApplication())
    ->setName('My App')
    ->setVersion('1.0.0')
    ->addArgument('name', InputArgument::OPTIONAL, 'Name')
    ->addOption('yell', 'y', InputOption::VALUE_NONE, 'Yell')
    ->setCode(function (InputInterface $input, OutputInterface $output): int {
        $output->writeln('Hello!');
        return Command::SUCCESS;
    })
    ->run();
```

## Terminal Information

```php
use Symfony\Component\Console\Terminal;

$terminal = new Terminal();
$width = $terminal->getWidth();
$height = $terminal->getHeight();
$colorMode = $terminal->getColorMode();
```

## Raw Input Tokens (Symfony 7.1+)

```php
$tokens = $input->getRawTokens();      // With command name
$tokens = $input->getRawTokens(true);  // Without command name

// Pass to external process
$process = new Process(['app:other', ...$input->getRawTokens(true)]);
```

## Profiling Commands

```bash
php bin/console --profile app:my-command
php bin/console -vvv --profile app:my-command  # With time/memory
php bin/console --profile --no-reset messenger:consume
```

## Key Classes Reference

| Class | Purpose |
|-------|---------|
| `Application` | Main console application |
| `Command` | Base class for commands |
| `InputInterface` | Input abstraction |
| `OutputInterface` | Output abstraction |
| `SymfonyStyle` | Styled output helper |
| `Table` | Table rendering |
| `ProgressBar` | Progress bar helper |
| `QuestionHelper` | Interactive questions |
| `Cursor` | Cursor manipulation |
| `CommandTester` | Testing commands |
| `Terminal` | Terminal capabilities |

## Global Options

| Option | Shortcut | Description |
|--------|----------|-------------|
| `--help` | `-h` | Display help |
| `--version` | `-V` | Display version |
| `--quiet` | `-q` | Quiet mode |
| `--verbose` | `-v/-vv/-vvv` | Verbosity |
| `--no-interaction` | `-n` | Disable interaction |
| `--ansi` | | Force ANSI |
| `--no-ansi` | | Disable ANSI |
| `--profile` | | Enable profiler |
| `--silent` | | No output (7.2+) |
| `--env` | `-e` | Environment (FrameworkBundle) |
| `--no-debug` | | Disable debug (FrameworkBundle) |
