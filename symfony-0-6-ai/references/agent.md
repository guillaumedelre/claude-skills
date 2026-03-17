# Symfony AI Agent Reference

This document covers the Agent component, tools system, input/output processors, memory providers, sub-agents, tool events, error handling, and testing patterns.

## Table of Contents

1. [Agent Architecture](#agent-architecture)
2. [Agent Creation](#agent-creation)
3. [Tools System](#tools-system)
4. [Tool Parameters](#tool-parameters)
5. [Registering Tools](#registering-tools)
6. [Tool Return Values](#tool-return-values)
7. [Third-Party Tools](#third-party-tools)
8. [Tool Error Handling](#tool-error-handling)
9. [Tool Sources (RAG)](#tool-sources-rag)
10. [Tool Events](#tool-events)
11. [Tool Configuration](#tool-configuration)
12. [Sub-Agents](#sub-agents)
13. [Input Processors](#input-processors)
14. [Output Processors](#output-processors)
15. [Memory Providers](#memory-providers)
16. [SimilaritySearch Tool](#similaritysearch-tool)
17. [Testing](#testing)
18. [Best Practices](#best-practices)

---

## Agent Architecture

### Core Components

The Agent component is built around several interconnected classes:

**Agent** - The main orchestrator that receives messages and coordinates tool execution through processors.

**Platform** - The underlying LLM interface (OpenAI, Anthropic, etc.) that handles model invocation.

**Toolbox** - Container for all available tools, manages tool registration and discovery.

**Processors** - Input and output processors that wrap the agent to transform messages before/after LLM calls.

**Memory** - Optional memory providers that augment agent context with persistent information.

### Basic Flow

```
Message Input
    ↓
InputProcessors (pre-processing)
    ↓
Agent.call() → Platform.invoke()
    ↓
Tool Execution (if needed)
    ↓
OutputProcessors (post-processing)
    ↓
Response Output
```

Each processor can inspect, modify, or replace messages before they reach the LLM, and can transform results before returning them to the caller.

---

## Agent Creation

### Basic Agent with OpenAI

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini');

$messages = new MessageBag(
    Message::forSystem('You are a helpful assistant.'),
    Message::ofUser('What is Symfony?'),
);

$result = $agent->call($messages);
echo $result->getContent();
```

### Agent with Input/Output Processors

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

// Create toolbox with tools
$toolbox = new Toolbox([/* your tools */]);

// Input processors run before LLM call
$inputProcessors = [
    new MemoryInputProcessor(
        new StaticMemoryProvider('User is an administrator')
    ),
    new AgentProcessor($toolbox),
];

// Output processors run after LLM call
$outputProcessors = [
    new AgentProcessor($toolbox),
];

$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    $inputProcessors,
    $outputProcessors
);
```

### Agent Options

```php
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o-mini',
    inputProcessors: [],
    outputProcessors: [],
    options: [
        'temperature' => 0.7,
        'max_tokens' => 1000,
        'top_p' => 0.9,
    ]
);
```

### Agent from Symfony Bundle

When using the Symfony AI Bundle, agents are autowired via service names:

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\AI\Agent\AgentInterface;

public function __construct(
    #[Autowire(service: 'ai.agent.default')]
    private AgentInterface $agent,
) {}

public function askQuestion(string $question): string
{
    return $this->agent->call(
        new MessageBag(Message::ofUser($question))
    )->getContent();
}
```

---

## Tools System

### Overview

Tools allow agents to call external functions to gather information or perform actions. Tools are registered with the agent through a Toolbox, and the LLM decides when to call them based on the task.

### Defining a Tool with #[AsTool]

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool('search_articles', 'Search for articles by keywords and filters')]
final readonly class SearchArticles
{
    public function __construct(
        private ArticleRepository $repository,
    ) {}

    /**
     * @param array<string> $keywords Search terms to find articles
     * @param string $type Article type (tutorial, guide, news)
     * @param int $limit Maximum results to return
     */
    public function __invoke(
        array $keywords,
        string $type,
        int $limit = 10,
    ): array {
        return $this->repository->search($keywords, $type, $limit);
    }
}
```

### Tool with Dependency Injection

```php
#[AsTool('get_user', 'Retrieve user by email')]
final readonly class GetUser
{
    public function __construct(
        private UserRepository $users,
    ) {}

    public function __invoke(string $email): ?array
    {
        $user = $this->users->findOneBy(['email' => $email]);
        return $user ? [
            'id' => $user->getId(),
            'name' => $user->getName(),
            'email' => $user->getEmail(),
        ] : null;
    }
}
```

### Tool with Description Parameter

```php
#[AsTool(
    name: 'calculate_discount',
    description: 'Calculate discount percentage based on order total'
)]
final readonly class CalculateDiscount
{
    public function __invoke(float $totalAmount): array
    {
        $discountPercent = match (true) {
            $totalAmount > 1000 => 15,
            $totalAmount > 500 => 10,
            $totalAmount > 100 => 5,
            default => 0,
        };

        return [
            'discount_percent' => $discountPercent,
            'discount_amount' => $totalAmount * $discountPercent / 100,
        ];
    }
}
```

### Tool with Multiple Methods

A tool class can expose multiple methods as separate tools:

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool(name: 'weather_current', description: 'Get current weather', method: 'current')]
#[AsTool(name: 'weather_forecast', description: 'Get weather forecast', method: 'forecast')]
final readonly class WeatherTool
{
    public function __construct(
        private WeatherService $service,
    ) {}

    public function current(float $latitude, float $longitude): array
    {
        return $this->service->getCurrentWeather($latitude, $longitude);
    }

    public function forecast(
        float $latitude,
        float $longitude,
        int $days = 7,
    ): array {
        return $this->service->getForecast($latitude, $longitude, $days);
    }
}
```

### Tool Naming Conventions

Tool names should be:
- Lowercase with underscores (snake_case): `search_articles`, `calculate_total`
- Descriptive of what the tool does: `create_invoice` not `do_stuff`
- Unique across the toolbox

Descriptions should:
- Be concise (1-2 sentences): "Search articles by keywords and type"
- Explain what the tool does, not how to use it
- Help the LLM understand when to call the tool

---

## Tool Parameters

### Basic Parameters

```php
#[AsTool('process_order', 'Process a customer order')]
final readonly class ProcessOrder
{
    public function __invoke(
        string $orderId,
        float $amount,
        bool $sendConfirmation = true,
    ): array {
        // Implementation
    }
}
```

### Parameter Validation with #[With]

The `#[With]` attribute adds validation constraints that the LLM is aware of:

```php
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

#[AsTool('create_user', 'Create a new user')]
final readonly class CreateUser
{
    public function __invoke(
        #[With(pattern: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')]
        string $email,

        #[With(minimum: 8, maximum: 50)]
        string $username,

        #[With(enum: ['user', 'admin', 'moderator'])]
        string $role,

        #[With(minimum: 18, maximum: 150)]
        int $age,
    ): array {
        // Implementation
    }
}
```

### Validation Attributes

- **pattern** - Regular expression the value must match
- **minimum** - Minimum value (for numbers) or length (for strings)
- **maximum** - Maximum value (for numbers) or length (for strings)
- **enum** - List of allowed values
- **description** - PHPDoc or explicit description of the parameter

```php
#[AsTool('search', 'Search with filters')]
final readonly class Search
{
    public function __invoke(
        #[With(description: 'Search query (minimum 3 characters)', minimum: 3)]
        string $query,

        #[With(description: 'Number of results to return', minimum: 1, maximum: 100)]
        int $limit = 10,

        #[With(enum: ['relevance', 'date', 'popularity'])]
        string $sortBy = 'relevance',
    ): array {
        // Implementation
    }
}
```

### Using Backed PHP Enums

PHP backed enums are automatically converted to enum constraints:

```php
enum FilterType: string
{
    case All = 'all';
    case Published = 'published';
    case Draft = 'draft';
}

#[AsTool('list_articles', 'List articles with filters')]
final readonly class ListArticles
{
    public function __invoke(
        #[With(enum: FilterType::class)]
        FilterType $status,
    ): array {
        // $status is automatically validated against FilterType cases
        return match ($status) {
            FilterType::All => $this->repository->findAll(),
            FilterType::Published => $this->repository->findPublished(),
            FilterType::Draft => $this->repository->findDrafts(),
        };
    }
}
```

The LLM receives the enum values and will only call the tool with valid values.

### Type Hints and PHPDoc

Always use type hints and add PHPDoc for clarity:

```php
#[AsTool('process_batch', 'Process multiple items')]
final readonly class ProcessBatch
{
    /**
     * @param array<string> $itemIds List of item identifiers to process
     * @param array<string, mixed> $options Processing options (optional)
     */
    public function __invoke(
        array $itemIds,
        array $options = [],
    ): array {
        // Implementation
    }
}
```

---

## Registering Tools

### Toolbox Registration

```php
use Symfony\AI\Agent\Toolbox\Toolbox;

$toolbox = new Toolbox([
    new SearchArticles($repository),
    new GetUser($userRepository),
    new CalculateDiscount(),
]);
```

### AgentProcessor with Toolbox

The AgentProcessor bridges tools to the agent:

```php
use Symfony\AI\Agent\Toolbox\AgentProcessor;

$processor = new AgentProcessor($toolbox);
$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    [$processor],  // Input processor
    [$processor],  // Output processor
);
```

### Automatic Tool Discovery from Container

In a Symfony application, tools can be auto-discovered and tagged:

```yaml
# config/services.yaml
services:
    App\Tool\SearchArticles:
        tags:
            - name: 'ai.tool'

    App\Tool\GetUser:
        tags:
            - name: 'ai.tool'
```

Then register them with the toolbox via the bundle:

```yaml
# config/packages/ai.yaml
ai:
    agent:
        default:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
            tools: ['App\Tool\SearchArticles', 'App\Tool\GetUser']
```

### ReflectionToolFactory

Automatically discover tools from a class using reflection:

```php
use Symfony\AI\Agent\Toolbox\Factory\ReflectionToolFactory;

$factory = new ReflectionToolFactory();
$tools = $factory->createTools(MyToolsClass::class);
$toolbox = new Toolbox($tools);
```

### MemoryToolFactory

Register memory providers as special tools:

```php
use Symfony\AI\Agent\Toolbox\Factory\MemoryToolFactory;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

$memoryFactory = new MemoryToolFactory();
$memoryTool = $memoryFactory->createTool(
    new StaticMemoryProvider(
        'User is a premium member',
        'Session timezone is UTC'
    )
);

$toolbox = new Toolbox([$memoryTool, /* other tools */]);
```

### ChainFactory

Combine multiple tool factories:

```php
use Symfony\AI\Agent\Toolbox\Factory\ChainFactory;

$factory = new ChainFactory([
    new ReflectionToolFactory(),
    new MemoryToolFactory(),
    // Custom factories
]);

$tools = $factory->createTools(MyToolsClass::class);
```

---

## Tool Return Values

### String Return Value

```php
#[AsTool('generate_report', 'Generate a summary report')]
final readonly class GenerateReport
{
    public function __invoke(string $topic): string
    {
        // Return narrative text
        return "The topic '$topic' covers the following key points: ...";
    }
}
```

### Array Return Value

```php
#[AsTool('search_articles', 'Search articles')]
final readonly class SearchArticles
{
    public function __invoke(string $keywords): array
    {
        return [
            [
                'id' => 1,
                'title' => 'Article Title',
                'excerpt' => 'Summary...',
                'url' => 'https://example.com/article-1',
            ],
            [
                'id' => 2,
                'title' => 'Another Article',
                'excerpt' => 'Summary...',
                'url' => 'https://example.com/article-2',
            ],
        ];
    }
}
```

### JsonSerializable Objects

```php
use JsonSerializable;

final class SearchResult implements JsonSerializable
{
    public function __construct(
        public string $id,
        public string $title,
        public string $content,
        public float $score,
    ) {}

    public function jsonSerialize(): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'content' => $this->content,
            'score' => $this->score,
        ];
    }
}

#[AsTool('search', 'Search documents')]
final readonly class Search
{
    public function __invoke(string $query): array
    {
        return [
            new SearchResult('1', 'Title 1', 'Content...', 0.95),
            new SearchResult('2', 'Title 2', 'Content...', 0.87),
        ];
    }
}
```

### Automatic JSON Conversion

Tool return values are automatically converted to JSON for the LLM:

- **String** → JSON string
- **Array** → JSON array
- **JsonSerializable** → JSON-encoded object
- **Object with public properties** → JSON object

---

## Third-Party Tools

Symfony AI provides official tool integrations:

### Brave Search

```php
use Symfony\AI\Bridge\Brave\BraveSearch;

$tool = new BraveSearch($_ENV['BRAVE_API_KEY']);
$toolbox = new Toolbox([$tool]);
```

Search the web with Brave Search API.

### Wikipedia

```php
use Symfony\AI\Bridge\Wikipedia\Wikipedia;

$tool = new Wikipedia();
$toolbox = new Toolbox([$tool]);
```

Search and retrieve Wikipedia articles.

### Tavily Search

```php
use Symfony\AI\Bridge\Tavily\TavilySearch;

$tool = new TavilySearch($_ENV['TAVILY_API_KEY']);
$toolbox = new Toolbox([$tool]);
```

Web search with research capabilities.

### SerpAPI

```php
use Symfony\AI\Bridge\SerpApi\SerpApi;

$tool = new SerpApi($_ENV['SERPAPI_API_KEY']);
$toolbox = new Toolbox([$tool]);
```

Search engine results aggregation.

### Web Crawler

```php
use Symfony\AI\Bridge\Crawler\Crawler;

$tool = new Crawler($httpClient);
$toolbox = new Toolbox([$tool]);
```

Crawl and extract content from web pages.

### Mapbox

```php
use Symfony\AI\Bridge\Mapbox\Mapbox;

$tool = new Mapbox($_ENV['MAPBOX_API_KEY']);
$toolbox = new Toolbox([$tool]);
```

Geocoding, reverse geocoding, and route optimization.

### YouTube

```php
use Symfony\AI\Bridge\YouTube\YouTube;

$tool = new YouTube($_ENV['YOUTUBE_API_KEY']);
$toolbox = new Toolbox([$tool]);
```

Search and retrieve video metadata.

### Clock

```php
use Symfony\AI\Bridge\Clock\Clock;

$tool = new Clock();
$toolbox = new Toolbox([$tool]);
```

Provides current date/time to the agent. Useful for time-aware queries.

---

## Tool Error Handling

### ToolExecutionExceptionInterface

Implement this interface on custom exceptions for better error messaging to the LLM:

```php
use Symfony\AI\Agent\Toolbox\ToolExecutionExceptionInterface;

final class ResourceNotFoundException extends \RuntimeException implements ToolExecutionExceptionInterface
{
    public function __construct(string $resourceType, string $id)
    {
        parent::__construct("$resourceType with ID '$id' not found");
    }

    public function getToolCallResult(): string
    {
        return 'The requested resource was not found. Please verify the ID and try again.';
    }
}
```

When this exception is thrown from a tool, the LLM receives the `getToolCallResult()` message instead of a raw exception, allowing it to handle the error intelligently:

```php
#[AsTool('get_user', 'Get user by ID')]
final readonly class GetUser
{
    public function __invoke(int $userId): array
    {
        $user = $this->repository->find($userId);
        if (!$user) {
            throw new ResourceNotFoundException('User', (string)$userId);
        }
        return $user->toArray();
    }
}
```

### Custom Exception Handling

```php
final class InvalidParameterException extends \RuntimeException implements ToolExecutionExceptionInterface
{
    public function __construct(
        public readonly string $parameter,
        public readonly string $reason,
    ) {
        parent::__construct("Invalid parameter '$parameter': $reason");
    }

    public function getToolCallResult(): string
    {
        return "Parameter '$this->parameter' is invalid: $this->reason";
    }
}

#[AsTool('process_payment', 'Process a payment')]
final readonly class ProcessPayment
{
    public function __invoke(float $amount, string $currency): array
    {
        if ($amount <= 0) {
            throw new InvalidParameterException('amount', 'must be greater than 0');
        }

        if (!in_array($currency, ['USD', 'EUR', 'GBP'])) {
            throw new InvalidParameterException('currency', 'unsupported currency code');
        }

        // Process payment
        return ['success' => true, 'transaction_id' => uniqid()];
    }
}
```

### FaultTolerantToolbox

Wrap a toolbox to catch and log exceptions:

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

$toolbox = new Toolbox([/* tools */]);
$faultTolerant = new FaultTolerantToolbox($toolbox);

$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    [new AgentProcessor($faultTolerant)],
    [new AgentProcessor($faultTolerant)],
);
```

---

## Tool Sources (RAG)

### HasSourcesInterface

Tools can return sources for RAG (Retrieval-Augmented Generation) patterns:

```php
use Symfony\AI\Agent\Toolbox\HasSourcesInterface;
use Symfony\AI\Agent\Toolbox\Source;

#[AsTool('retrieve_documentation', 'Retrieve Symfony documentation')]
final readonly class RetrieveDocumentation implements HasSourcesInterface
{
    private array $sources = [];

    public function __invoke(string $topic): array
    {
        $docs = $this->repository->findByTopic($topic);

        $results = [];
        foreach ($docs as $doc) {
            $results[] = $doc->getContent();

            $this->sources[] = new Source(
                title: $doc->getTitle(),
                url: $doc->getUrl(),
                metadata: ['section' => $doc->getSection()],
            );
        }

        return $results;
    }

    public function getSources(): array
    {
        return $this->sources;
    }
}
```

### Retrieving Sources from Result

```php
$messages = new MessageBag(
    Message::ofUser('How does dependency injection work in Symfony?')
);

$result = $agent->call($messages);

// Check if sources are available
if ($result instanceof HasSourcesInterface) {
    foreach ($result->getSources() as $source) {
        echo $source->getTitle() . ': ' . $source->getUrl() . PHP_EOL;
    }
}
```

### Source Objects

The `Source` class provides structured source information:

```php
use Symfony\AI\Agent\Toolbox\Source;

$source = new Source(
    title: 'Symfony Dependency Injection Component',
    url: 'https://symfony.com/doc/current/components/dependency_injection.html',
    metadata: [
        'section' => 'Components',
        'version' => '7.4',
        'type' => 'documentation',
    ],
);

echo $source->getTitle();      // "Symfony Dependency Injection Component"
echo $source->getUrl();        // URL
echo $source->getMetadata();   // Full metadata array
echo $source->get('version');  // '7.4'
```

---

## Tool Events

### Tool Event Types

The agent dispatches several events during tool execution:

**ToolCallsExecuted** - Fired when the LLM decides to call tools

**ToolCallArgumentsResolved** - Fired after arguments are validated/resolved

**ToolCallSucceeded** - Fired after successful tool execution

**ToolCallFailed** - Fired when tool execution fails

### Listening to Tool Events

```php
use Symfony\AI\Agent\Event\ToolCallSucceeded;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

final class ToolExecutionListener implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            ToolCallSucceeded::class => 'onToolSuccess',
        ];
    }

    public function onToolSuccess(ToolCallSucceeded $event): void
    {
        echo "Tool '{$event->toolName}' executed successfully\n";
        echo "Result: " . json_encode($event->result) . "\n";
    }
}
```

### ToolCallsExecuted Event

```php
use Symfony\AI\Agent\Event\ToolCallsExecuted;

final class ToolSelectionListener implements EventSubscriberInterface
{
    public function onToolsSelected(ToolCallsExecuted $event): void
    {
        foreach ($event->toolCalls as $call) {
            echo "LLM selected tool: {$call->name}\n";
            echo "Arguments: " . json_encode($call->arguments) . "\n";
        }
    }
}
```

### ToolCallFailed Event

```php
use Symfony\AI\Agent\Event\ToolCallFailed;

final class ErrorHandlingListener implements EventSubscriberInterface
{
    public function onToolFailure(ToolCallFailed $event): void
    {
        logger()->error(
            'Tool execution failed',
            [
                'tool' => $event->toolName,
                'error' => $event->exception->getMessage(),
            ]
        );
    }
}
```

### Registering Event Listeners

In Symfony:

```yaml
# config/services.yaml
services:
    App\Listener\ToolExecutionListener:
        tags:
            - { name: kernel.event_subscriber }
```

---

## Tool Configuration

### AgentProcessor Options

```php
use Symfony\AI\Agent\Toolbox\AgentProcessor;

$processor = new AgentProcessor(
    $toolbox,
    keepToolMessages: true,        // Keep tool calls in message history
    maxToolCalls: 10,              // Maximum tools to call per request
);
```

### keepToolMessages

When `true`, tool call messages are included in the message history sent back to the LLM. When `false`, they're excluded (default behavior):

```php
$processor = new AgentProcessor($toolbox, keepToolMessages: true);

// Messages sent to LLM will include:
// - User message
// - Assistant message with tool_calls
// - Tool results
```

### maxToolCalls

Limit the number of tool calls to prevent runaway costs or infinite loops:

```php
$processor = new AgentProcessor($toolbox, maxToolCalls: 5);
```

If the LLM tries to call more tools, execution stops and an error is returned.

### Tool Filtering Per Call

Filter available tools per agent call:

```php
$result = $agent->call(
    $messages,
    options: [
        'tools' => ['search_articles', 'get_user'],  // Only use these tools
    ]
);
```

Or disable tools entirely for a call:

```php
$result = $agent->call(
    $messages,
    options: [
        'tools' => [],  // Don't call any tools
    ]
);
```

---

## Sub-Agents

### Subagent Class

A sub-agent is a delegated agent that handles a specific task and returns results to the parent agent:

```php
use Symfony\AI\Agent\Subagent;

$searchAgent = new Subagent(
    agent: new Agent($platform, 'gpt-4o-mini', [new AgentProcessor($searchToolbox)]),
    name: 'search_agent',
    description: 'Specialized agent for searching articles and documents',
    systemPrompt: 'You are an expert search agent. Always return results in a structured format.',
);
```

### Using Sub-Agents as Tools

```php
#[AsTool('delegate_search', 'Delegate search to specialized agent')]
final readonly class DelegateSearch
{
    public function __construct(
        private Subagent $searchAgent,
    ) {}

    public function __invoke(string $query): array
    {
        $messages = new MessageBag(Message::ofUser($query));
        $result = $this->searchAgent->getAgent()->call($messages);
        return ['results' => $result->getContent()];
    }
}
```

### Delegation Pattern

```php
final readonly class ResearchAssistant
{
    public function __construct(
        private Subagent $summaryAgent,
        private Subagent $translationAgent,
    ) {}

    public function research(string $topic): array
    {
        // Delegate to summary agent
        $summary = $this->summaryAgent->call(
            new MessageBag(Message::ofUser("Summarize: $topic"))
        );

        // Delegate to translation agent
        $translated = $this->translationAgent->call(
            new MessageBag(Message::ofUser("Translate to Spanish: " . $summary->getContent()))
        );

        return [
            'summary' => $summary->getContent(),
            'translation' => $translated->getContent(),
        ];
    }
}
```

---

## Input Processors

### InputProcessorInterface

Input processors transform messages before they reach the LLM:

```php
use Symfony\AI\Agent\Processor\InputProcessorInterface;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Agent\AgentOptions;

final readonly class CustomInputProcessor implements InputProcessorInterface
{
    public function processInput(MessageBag $messages, AgentOptions $options): MessageBag
    {
        // Inspect or modify messages before LLM sees them
        // Return the modified MessageBag
        return $messages;
    }
}
```

### Mutating Messages

```php
final readonly class ContextEnricherProcessor implements InputProcessorInterface
{
    public function processInput(MessageBag $messages, AgentOptions $options): MessageBag
    {
        // Get the last user message
        $userMessages = array_filter(
            $messages->all(),
            fn($msg) => $msg->getRole() === 'user'
        );

        if (!empty($userMessages)) {
            $lastUserMsg = end($userMessages);

            // Create enriched message with context
            $enriched = Message::ofUser(
                $lastUserMsg->getContent() . "\n\nContext: User is logged in as admin"
            );

            // Return new MessageBag with enriched message
            return new MessageBag(...$messages->all());
        }

        return $messages;
    }
}
```

### Mutating Options

```php
final readonly class TemperatureAdjusterProcessor implements InputProcessorInterface
{
    public function processInput(MessageBag $messages, AgentOptions $options): MessageBag
    {
        // Adjust options based on message content
        if (strpos($messages->getLastMessage()->getContent(), 'creative') !== false) {
            $options->set('temperature', 0.9);
        }

        return $messages;
    }
}
```

### AgentAwareInterface

Processors can access the agent instance:

```php
use Symfony\AI\Agent\Processor\AgentAwareInterface;
use Symfony\AI\Agent\AgentInterface;

final readonly class AgentAwareProcessor implements InputProcessorInterface, AgentAwareInterface
{
    private AgentInterface $agent;

    public function setAgent(AgentInterface $agent): void
    {
        $this->agent = $agent;
    }

    public function processInput(MessageBag $messages, AgentOptions $options): MessageBag
    {
        // Use agent context
        // $this->agent->getModel(), etc.
        return $messages;
    }
}
```

---

## Output Processors

### OutputProcessorInterface

Output processors transform the agent's response after the LLM call:

```php
use Symfony\AI\Agent\Processor\OutputProcessorInterface;
use Symfony\AI\Platform\Response;
use Symfony\AI\Agent\AgentOptions;

final readonly class CustomOutputProcessor implements OutputProcessorInterface
{
    public function processOutput(Response $response, AgentOptions $options): Response
    {
        // Inspect or modify the response
        // Can replace the entire response
        return $response;
    }
}
```

### Mutating Response Content

```php
final readonly class FormatOutputProcessor implements OutputProcessorInterface
{
    public function processOutput(Response $response, AgentOptions $options): Response
    {
        $content = $response->getContent();

        // Format the output (e.g., convert to markdown)
        $formatted = $this->formatAsMarkdown($content);

        // Create new response with formatted content
        return new Response(
            content: $formatted,
            // Copy other properties from original response
        );
    }

    private function formatAsMarkdown(string $content): string
    {
        // Formatting logic
        return $content;
    }
}
```

### Replacing Response

```php
final readonly class CachingOutputProcessor implements OutputProcessorInterface
{
    public function __construct(
        private CacheInterface $cache,
    ) {}

    public function processOutput(Response $response, AgentOptions $options): Response
    {
        // Check cache for similar queries
        $cacheKey = hash('sha256', $response->getContent());

        if ($this->cache->hasItem($cacheKey)) {
            $cached = $this->cache->getItem($cacheKey)->get();
            // Return cached response instead
            return new Response(content: $cached);
        }

        // Cache the response
        $item = $this->cache->getItem($cacheKey);
        $item->set($response->getContent());
        $this->cache->save($item);

        return $response;
    }
}
```

---

## Memory Providers

### StaticMemoryProvider

Store static facts about the user/context:

```php
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;

$memory = new StaticMemoryProvider(
    'User name is John Doe',
    'User timezone is Europe/Paris',
    'User prefers concise answers',
    'User is a Symfony expert',
);

$processor = new MemoryInputProcessor($memory);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor]);
```

The static memory is injected into every message as system context.

### EmbeddingProvider

Use embeddings for semantic memory matching:

```php
use Symfony\AI\Agent\Memory\EmbeddingProvider;

$embeddingProvider = new EmbeddingProvider(
    $vectorizer,  // Embedding model
    $store,       // Vector store
    $retriever,   // Semantic search
);

$processor = new MemoryInputProcessor($embeddingProvider);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor]);
```

Retrieves semantically relevant memories based on the current query.

### MemoryInputProcessor

Applies memory to messages:

```php
use Symfony\AI\Agent\Memory\MemoryInputProcessor;

$memoryProcessor = new MemoryInputProcessor($memoryProvider);
$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    [$memoryProcessor],  // Memory enhances input
    []
);
```

### Custom MemoryProviderInterface

```php
use Symfony\AI\Agent\Memory\MemoryProviderInterface;

final readonly class DatabaseMemoryProvider implements MemoryProviderInterface
{
    public function __construct(
        private UserRepository $users,
    ) {}

    public function getMemory(string $userId): array
    {
        $user = $this->users->find($userId);

        return [
            "User name: {$user->getName()}",
            "Account created: {$user->getCreatedAt()->format('Y-m-d')}",
            "Member level: {$user->getMemberLevel()}",
            "Last activity: {$user->getLastActivityAt()->format('Y-m-d H:i')}",
        ];
    }
}

$processor = new MemoryInputProcessor(new DatabaseMemoryProvider($users));
```

### Disabling Memory Per Call

```php
$result = $agent->call(
    $messages,
    options: [
        'memory_enabled' => false,  // Skip memory processors for this call
    ]
);
```

---

## SimilaritySearch Tool

### Setup

The SimilaritySearch tool enables RAG by allowing agents to retrieve semantically similar documents:

```php
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Toolbox\AgentProcessor;

$similaritySearch = new SimilaritySearch($vectorizer, $store);
$toolbox = new Toolbox([$similaritySearch]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    [$processor],
    [$processor]
);
```

### Tool Behavior

The SimilaritySearch tool automatically exposes methods for:
- **similarity_search** - Find documents similar to a query
- **similarity_search_with_scores** - Include relevance scores

```php
// Agent calls tool automatically when relevant
$messages = new MessageBag(
    Message::forSystem('Answer questions using the similarity_search tool.'),
    Message::ofUser('What does the Symfony Cache component do?'),
);

$result = $agent->call($messages);
// Agent retrieves similar docs, synthesizes answer
echo $result->getContent();
```

### Configuration

```php
$similaritySearch = new SimilaritySearch(
    $vectorizer,
    $store,
    limit: 5,          // Number of results
    threshold: 0.7,    // Minimum similarity score
);
```

### Prompt Engineering for RAG

```php
$messages = new MessageBag(
    Message::forSystem(
        'You are a documentation assistant. '
        . 'Always use the similarity_search tool to find relevant documentation. '
        . 'Cite sources when answering questions. '
        . 'If no relevant documents are found, say so explicitly.'
    ),
    Message::ofUser('How do I create a custom Symfony command?'),
);

$result = $agent->call($messages);
```

---

## Testing

### MockAgent

Test agents without making real API calls:

```php
use Symfony\AI\Agent\MockAgent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$agent = new MockAgent([
    'What is Symfony?' => 'Symfony is a PHP web application framework.',
    'Tell me about Console' => 'The Console component creates CLI commands.',
]);

$messages = new MessageBag(Message::ofUser('What is Symfony?'));
$result = $agent->call($messages);

echo $result->getContent();  // "Symfony is a PHP web application framework."
```

### MockResponse

Return structured responses:

```php
use Symfony\AI\Agent\MockAgent;
use Symfony\AI\Platform\Test\MockResponse;

$agent = new MockAgent([
    'search' => new MockResponse(
        content: json_encode([
            ['title' => 'Article 1', 'url' => 'https://example.com/1'],
            ['title' => 'Article 2', 'url' => 'https://example.com/2'],
        ])
    ),
]);
```

### Callable Responses

Use callables for dynamic responses:

```php
$agent = new MockAgent([
    'get_time' => function (MessageBag $messages): string {
        return date('H:i:s');
    },
]);
```

### Assertions

#### assertCallCount

```php
$agent = new MockAgent(['test' => 'response']);
$agent->call(new MessageBag(Message::ofUser('test')));

$agent->assertCallCount(1);
```

#### assertCalledWith

```php
$agent->assertCalledWith('test');
$agent->assertCalledWith('What is Symfony?');
```

#### getCalls

```php
$calls = $agent->getCalls();
foreach ($calls as $call) {
    echo $call->getContent();
}
```

#### getLastCall

```php
$lastCall = $agent->getLastCall();
echo $lastCall->getContent();
```

#### reset

```php
$agent->reset();  // Clear all call history
```

### Complete Test Example

```php
use PHPUnit\Framework\TestCase;
use Symfony\AI\Agent\MockAgent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

final class SearchAgentTest extends TestCase
{
    public function testSearchReturnsResults(): void
    {
        $agent = new MockAgent([
            'search for symfony cache' => json_encode([
                ['id' => 1, 'title' => 'Cache Component', 'score' => 0.95],
                ['id' => 2, 'title' => 'Caching Strategies', 'score' => 0.87],
            ]),
        ]);

        $messages = new MessageBag(
            Message::ofUser('search for symfony cache')
        );

        $result = $agent->call($messages);
        $data = json_decode($result->getContent(), true);

        $this->assertCount(2, $data);
        $this->assertEquals('Cache Component', $data[0]['title']);
        $agent->assertCallCount(1);
        $agent->assertCalledWith('search for symfony cache');
    }
}
```

---

## Best Practices

### Tool Design

**Single Responsibility** - Each tool should do one thing well:

```php
// Good: Focused tools
#[AsTool('get_user_by_id', 'Get user by ID')]
#[AsTool('get_user_by_email', 'Get user by email')]
#[AsTool('update_user_profile', 'Update user profile')]

// Bad: Multi-purpose tool
#[AsTool('user_management', 'Manage users (get, update, delete, etc.)')]
```

**Clear Descriptions** - Help the LLM understand when to use the tool:

```php
// Good description
#[AsTool(
    'calculate_shipping_cost',
    'Calculate shipping cost based on weight, destination, and shipping method'
)]

// Vague description
#[AsTool('calc', 'Calculate something')]
```

### Error Handling

Always implement `ToolExecutionExceptionInterface` for custom exceptions:

```php
final class ValidationError extends \RuntimeException implements ToolExecutionExceptionInterface
{
    public function getToolCallResult(): string
    {
        return "Validation failed: {$this->getMessage()}";
    }
}

// LLM receives helpful error message, not full exception
throw new ValidationError('Email format is invalid');
```

### Token Management

Monitor and limit tool calls to manage token costs:

```php
// Limit to 5 tool calls max
$processor = new AgentProcessor($toolbox, maxToolCalls: 5);

// Disable tools for certain queries
$result = $agent->call($messages, options: ['tools' => []]);
```

### Memory Strategy

Choose memory providers based on use case:

- **StaticMemoryProvider** - Fixed user context (name, preferences)
- **EmbeddingProvider** - Semantic/relevant memories (conversation history)
- **DatabaseMemoryProvider** - Dynamic user data (account info, history)

```php
// User preferences (static)
$staticMemory = new StaticMemoryProvider(
    'User prefers metric units',
    'User timezone is UTC',
);

// Conversation history (semantic)
$embeddingMemory = new EmbeddingProvider($vectorizer, $store);

$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    [
        new MemoryInputProcessor($staticMemory),
        new MemoryInputProcessor($embeddingMemory),
        new AgentProcessor($toolbox),
    ]
);
```

### Testing

Always test with mocks first:

```php
// Development/testing
$testAgent = new MockAgent(['query' => 'canned response']);

// Production
$productionAgent = new Agent($platform, 'gpt-4o-mini');
```

Test tool inputs and outputs:

```php
final class MyToolTest extends TestCase
{
    public function testToolValidation(): void
    {
        $tool = new MyTool($dependency);

        $this->expectException(ValidationError::class);
        $tool(['invalid' => 'data']);
    }
}
```

### Tool Filtering

Only expose necessary tools:

```php
// Only allow search-related tools
$result = $agent->call(
    $messages,
    options: [
        'tools' => ['search_articles', 'search_users'],
    ]
);

// Disable tools for simple tasks
if ($isSimpleQuery) {
    $result = $agent->call($messages, options: ['tools' => []]);
}
```

### Fault Tolerance

Use FaultTolerantToolbox for production:

```php
$toolbox = new FaultTolerantToolbox(
    new Toolbox([/* tools */])
);

// Exceptions are logged, agent continues
$processor = new AgentProcessor($toolbox);
```

### RAG Best Practices

When using SimilaritySearch with agents:

1. **Chunk documents appropriately** (200-500 tokens per document)
2. **Include metadata** for filtering and source attribution
3. **Engineer prompts** to instruct agent when to use search
4. **Cite sources** in agent responses
5. **Monitor retrieval quality** - Relevance scores indicate confidence

```php
// Index documents with metadata
$doc = new TextDocument(
    content: 'The Console component is used to create CLI commands...',
    metadata: [
        'title' => 'Console Component',
        'url' => 'https://symfony.com/doc/current/console',
        'section' => 'Components',
    ]
);

$indexer->index($doc);

// Retrieve with filtering
$results = $retriever->retrieve(
    'How do I create commands?',
    filters: ['section' => 'Components'],
    limit: 5
);
```

