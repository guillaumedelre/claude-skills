# Console Commands - Creation, Arguments, Options

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

For lifecycle hooks:

```php
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
    ) {}

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
): int {}
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
): int {}
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

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $iterations = $input->getOption('iterations');
    $yell = $input->getOption('yell');
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

### Basic DTO

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

#[AsCommand(name: 'app:list-users')]
class ListUsersCommand
{
    public function __invoke(#[MapInput] ListUsersInput $input): int
    {
        $limit = $input->pagination->limit;
        $page = $input->pagination->page;
        return Command::SUCCESS;
    }
}
```

### Property Hooks (PHP 8.4+)

Normalize input values:

```php
class CreateUserInput
{
    #[Argument]
    public string $email {
        set(string $value) {
            $this->email = strtolower(trim($value));
        }
    }

    #[Option]
    public array $roles = [] {
        set(array $value) {
            $this->roles = array_map('strtoupper', $value);
        }
    }
}
```

## Command Lifecycle

Commands execute in this order:

1. **initialize()** - Initialize variables before interact/execute
2. **interact()** - Ask for missing required options/arguments (not called with `--no-interaction`)
3. **__invoke()** or **execute()** - Execute command logic, must return exit status

## Raw Input Tokens (Symfony 7.1+)

Access raw command-line tokens:

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    $tokens = $input->getRawTokens();      // With command name
    $tokens = $input->getRawTokens(true);  // Without command name

    // Pass to external process
    $process = new Process(['app:other', ...$input->getRawTokens(true)]);
    $process->mustRun();

    return Command::SUCCESS;
}
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

## Exit Codes

```php
use Symfony\Component\Console\Command\Command;

return Command::SUCCESS; // 0
return Command::FAILURE; // 1
return Command::INVALID; // 2
```
