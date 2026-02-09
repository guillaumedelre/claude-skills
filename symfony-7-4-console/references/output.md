# Console Output - SymfonyStyle and Formatting

## SymfonyStyle

### Basic Usage

```php
use Symfony\Component\Console\Style\SymfonyStyle;

public function __invoke(SymfonyStyle $io): int
{
    $io->title('Main Title');
    $io->section('Section Title');
    $io->text('Regular text');
    $io->newLine(2);
    return Command::SUCCESS;
}
```

### Titles and Sections

```php
$io->title('Command Title');     // Large title for command
$io->section('Section Name');    // Section separator
```

### Text Output

```php
$io->text('Single line');
$io->text(['Line 1', 'Line 2', 'Line 3']);

$io->listing(['Item 1', 'Item 2', 'Item 3']);

$io->newLine();      // 1 blank line
$io->newLine(3);     // 3 blank lines
```

### Tables

```php
// Simple table
$io->table(
    ['Header 1', 'Header 2'],
    [
        ['Cell 1-1', 'Cell 1-2'],
        ['Cell 2-1', 'Cell 2-2'],
    ]
);

// Horizontal table
$io->horizontalTable(
    ['Header 1', 'Header 2'],
    [['Value 1', 'Value 2']]
);

// Definition list (key-value pairs)
$io->definitionList(
    'Title',
    ['key1' => 'value1'],
    ['key2' => 'value2'],
);
```

### Tree (Symfony 7.3+)

```php
$io->tree([
    'src' => [
        'Controller' => [
            'DefaultController.php',
            'UserController.php',
        ],
        'Entity' => ['User.php'],
        'Kernel.php',
    ],
    'templates' => ['base.html.twig'],
]);
```

### Admonitions

```php
$io->note('Information note');
$io->note(['Line 1', 'Line 2']);

$io->caution('Important warning');
$io->caution(['Line 1', 'Line 2']);
```

### Result Messages

```php
$io->success('Operation completed successfully');
$io->info('Information without [OK] label');
$io->warning('Warning message');
$io->error('Error message');
```

### User Input

```php
// Ask question
$name = $io->ask('What is your name?');
$name = $io->ask('What is your name?', 'John');  // With default

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
$color = $io->choice('Pick a color', ['red', 'green', 'blue'], 'red');  // With default

// Multiple selection
$colors = $io->choice(
    'Pick colors (comma-separated)',
    ['red', 'green', 'blue'],
    multiSelect: true
);
```

### Progress Bars

```php
// Simple progress
$io->progressStart(100);
for ($i = 0; $i < 100; $i++) {
    // Do work
    $io->progressAdvance();
}
$io->progressFinish();

// Iterate with automatic progress
foreach ($io->progressIterate($items) as $item) {
    // Process item
}

// Advanced progress bar
$progressBar = $io->createProgressBar(100);
$progressBar->setFormat(' %current%/%max% [%bar%] %percent:3s%%');
$progressBar->start();
$progressBar->advance(10);
$progressBar->finish();
```

### Dynamic Tables

```php
$table = $io->createTable();
$table->setHeaders(['Name', 'Age']);
$table->addRow(['John', 30]);
$table->addRow(['Jane', 25]);
$table->render();
```

### Dynamic Tree (Symfony 7.3+)

```php
$tree = $io->createTree();
$root = $tree->addNode('src/');
$root->addChild('Controller/');
$root->addChild('Entity/');
$tree->render();
```

### Error Output (stderr)

```php
// Write to stderr instead of stdout
$io->getErrorStyle()->warning('Debug information');
$io->getErrorStyle()->error('Error output');
```

### URL Wrapping

```php
// Allow URLs to be wrapped for narrow terminals
$io->getOutputWrapper()->setAllowCutUrls(true);
```

## Output Sections

Create independent output regions that can be updated independently:

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

    // Overwrite section content
    $section1->overwrite('Goodbye');

    // Clear section
    $section2->clear();

    // Clear specific number of lines
    $section1->clear(2);

    // Set max height (scrolling region)
    $section1->setMaxHeight(5);

    return Command::SUCCESS;
}
```

## Raw Output

### Basic Output

```php
use Symfony\Component\Console\Output\OutputInterface;

$output->writeln('Line with newline');
$output->writeln(['Line 1', 'Line 2', 'Line 3']);
$output->write('No newline');

// Output generators
$output->writeln($this->generateLines());
```

### Verbosity Levels

```php
use Symfony\Component\Console\Output\OutputInterface;

$output->writeln('Always shown');

// Conditional output based on verbosity
if ($output->isQuiet()) { /* -q */ }
if ($output->isVerbose()) { /* -v */ }
if ($output->isVeryVerbose()) { /* -vv */ }
if ($output->isDebug()) { /* -vvv */ }

// Write only at specific verbosity
$output->writeln('Debug info', OutputInterface::VERBOSITY_DEBUG);
$output->writeln('Verbose info', OutputInterface::VERBOSITY_VERBOSE);
$output->writeln('Very verbose', OutputInterface::VERBOSITY_VERY_VERBOSE);
```

| Level | Option | Constant |
|-------|--------|----------|
| Quiet | `-q` | `VERBOSITY_QUIET` |
| Normal | (none) | `VERBOSITY_NORMAL` |
| Verbose | `-v` | `VERBOSITY_VERBOSE` |
| Very Verbose | `-vv` | `VERBOSITY_VERY_VERBOSE` |
| Debug | `-vvv` | `VERBOSITY_DEBUG` |

### Silent Mode (Symfony 7.2+)

```bash
php bin/console command --silent
```

Disables all output including errors.

## Custom Styles

Create custom styles by implementing `StyleInterface`:

```php
namespace App\Console;

use Symfony\Component\Console\Style\StyleInterface;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class CustomStyle implements StyleInterface
{
    public function __construct(
        private InputInterface $input,
        private OutputInterface $output,
    ) {}

    // Implement StyleInterface methods...
}

// Usage
public function __invoke(InputInterface $input, OutputInterface $output): int
{
    $io = new CustomStyle($input, $output);
    return Command::SUCCESS;
}
```

## Terminal Information

```php
use Symfony\Component\Console\Terminal;

$terminal = new Terminal();
$width = $terminal->getWidth();
$height = $terminal->getHeight();
$colorMode = $terminal->getColorMode();
```
