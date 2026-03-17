---
name: "symfony-0-6-ai"
description: "Symfony AI components (v0.6, experimental) for building AI-powered PHP applications with LLMs. Use when working with AI agents, chat interfaces, vector stores, RAG (Retrieval-Augmented Generation), embeddings, or LLM integration in Symfony/PHP projects. Triggers on: Agent, Chat, Platform, Store, AsTool attribute, MessageBag, Message, Toolbox, SimilaritySearch, Indexer, Retriever, VectorDocument, TextDocument, OpenAI, Anthropic, Claude, GPT, Gemini, Ollama, embeddings, vectorizer, RAG, LLM integration, AI assistant, conversational AI, tool calling, function calling, MemoryProvider, InputProcessor, OutputProcessor, MockAgent, InMemoryPlatform, message stores, vector stores, ChromaDB, Pinecone."
---

# Symfony AI Components (v0.6)

**Status:** ⚠️ Experimental - Not covered by Symfony's Backward Compatibility Promise

GitHub: https://github.com/symfony/ai
Docs: https://symfony.com/doc/current/ai/

## Overview

Symfony AI provides four integrated components for building AI-powered PHP applications:

- **Platform** - Unified abstraction for 40+ AI providers (OpenAI, Anthropic, Azure, AWS, Google, Ollama, etc.)
- **Agent** - Framework for autonomous AI agents with tools, memory, and processors
- **Chat** - Conversational interfaces with message persistence
- **Store** - Vector storage and RAG capabilities with 18+ backend options

## Installation

```bash
# Install the bundle (includes all components)
composer require symfony/ai-bundle

# Or install components individually
composer require symfony/ai-platform
composer require symfony/ai-agent
composer require symfony/ai-chat
composer require symfony/ai-store
```

## Quick Reference

### 1. Basic Agent with OpenAI

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini');

$messages = new MessageBag(
    Message::forSystem('You are a helpful programming assistant.'),
    Message::ofUser('Explain dependency injection in Symfony.'),
);

$result = $agent->call($messages);
echo $result->getContent();
```

### 2. Bundle Configuration (Multi-Provider)

```yaml
# config/packages/ai.yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        ollama:
            host_url: 'http://localhost:11434'

    agent:
        default:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
            prompt: 'You are a helpful assistant.'

        research:
            platform: 'ai.platform.anthropic'
            model: 'claude-3-5-sonnet-20241022'
            prompt: 'You are a research assistant.'
```

### 3. Custom Tool with Validation

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

#[AsTool('search_articles', 'Search articles by type and priority')]
final readonly class SearchArticles
{
    public function __construct(
        private ArticleRepository $repository,
    ) {}

    /**
     * @param array<string> $keywords The search keywords
     * @param string $type Article type
     * @param int $priority Minimum priority level
     */
    public function __invoke(
        array $keywords,
        #[With(enum: ['tutorial', 'article', 'news'])]
        string $type,
        #[With(minimum: 1, maximum: 10)]
        int $priority,
    ): array {
        return $this->repository->search($keywords, $type, $priority);
    }
}
```

### 4. Agent with Tools (Autowired in Symfony)

```php
use Symfony\AI\Agent\AgentInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final readonly class ArticleAssistant
{
    public function __construct(
        #[Autowire(service: 'ai.agent.default')]
        private AgentInterface $agent,
    ) {}

    public function ask(string $question): string
    {
        $messages = new MessageBag(Message::ofUser($question));
        return $this->agent->call($messages)->getContent();
    }
}
```

### 5. Chat with Message Persistence

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Cache\Store as CacheStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini');

$cache = new FilesystemAdapter();
$messageStore = new CacheStore($cache, 'chat_session_123');
$chat = new Chat($agent, $messageStore);

// Conversation is preserved across requests
$chat->submit(Message::ofUser('What is Symfony?'));
$chat->submit(Message::ofUser('Tell me more about its components.'));
```

### 6. RAG: Indexing Documents

```php
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Indexer;

// Autowired in Symfony or manually created
$indexer = new Indexer($platform, 'text-embedding-3-small', $store);

$documents = [
    new TextDocument('Symfony is a PHP web application framework.'),
    new TextDocument('The Console component allows creating CLI commands.'),
    new TextDocument('Doctrine ORM integrates with Symfony for database access.'),
];

foreach ($documents as $doc) {
    $indexer->index($doc);
}
```

### 7. RAG: Retrieving Similar Documents

```php
use Symfony\AI\Store\Retriever;

$retriever = new Retriever($vectorizer, $store);

$documents = $retriever->retrieve('How do I create console commands?', [
    'limit' => 5,
]);

foreach ($documents as $doc) {
    echo $doc->getContent() . PHP_EOL;
}
```

### 8. Agent with RAG (SimilaritySearch Tool)

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$similaritySearch = new SimilaritySearch($vectorizer, $store);
$toolbox = new Toolbox([$similaritySearch]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

$messages = new MessageBag(
    Message::forSystem('Answer questions using only the similarity_search tool.'),
    Message::ofUser('What is Symfony Console component?'),
);

$result = $agent->call($messages);
echo $result->getContent();
```

### 9. Streaming Responses

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('You are a thoughtful philosopher.'),
    Message::ofUser('What is the meaning of life?'),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'stream' => true,
]);

foreach ($result->getContent() as $chunk) {
    echo $chunk;
    flush();
}
```

### 10. Testing with Mocks

```php
use Symfony\AI\Agent\MockAgent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// Testing agents
$agent = new MockAgent([
    'What is Symfony?' => 'Symfony is a PHP framework for web applications.',
    'Tell me about Console' => 'Console component creates CLI commands.',
]);

$messages = new MessageBag(Message::ofUser('What is Symfony?'));
$result = $agent->call($messages);

$agent->assertCallCount(1);
$agent->assertCalledWith('What is Symfony?');

// Testing platforms
use Symfony\AI\Platform\Test\InMemoryPlatform;

$platform = new InMemoryPlatform('Mocked response');
$result = $platform->invoke('gpt-4o-mini', 'test query');
```

## Documentation Structure

### Core References

- **[bundle.md](references/bundle.md)** - Bundle configuration, multi-agent orchestration, console commands, dependency injection, security
- **[platform.md](references/platform.md)** - All 40+ providers (OpenAI, Anthropic, Azure, AWS, Google, Ollama), messages, templates, streaming, embeddings, structured output, caching, failover
- **[agent.md](references/agent.md)** - Tools system (#[AsTool]), input/output processors, memory providers, sub-agents, tool events, error handling, testing (MockAgent)
- **[chat.md](references/chat.md)** - Message stores (Cache, Doctrine, Session, MongoDB, Redis, etc.), conversation persistence, custom stores
- **[store.md](references/store.md)** - Vector stores (ChromaDB, Pinecone, PostgreSQL, etc.), indexing, retrieval, RAG patterns, similarity search

## When to Read References

### Bundle Configuration
Read **bundle.md** when you need to:
- Configure multiple AI providers in YAML
- Set up agent definitions with prompts and tools
- Configure multi-agent orchestration with handoffs
- Set up vectorizers, stores, indexers, and retrievers
- Use console commands (ai:agent:call, ai:store:setup, etc.)
- Understand dependency injection patterns
- Implement tool-level security with #[IsGrantedTool]

### Platform & Providers
Read **platform.md** when you need to:
- Configure specific providers (OpenAI, Anthropic, Azure, AWS, Google, Mistral, Ollama, etc.)
- Work with message types (System, User, Assistant, Tool)
- Use message templates with variables
- Implement streaming responses
- Generate embeddings for RAG
- Use structured output (extract data into PHP classes)
- Handle images or audio input
- Implement caching or failover for reliability
- Test with InMemoryPlatform

### Agents & Tools
Read **agent.md** when you need to:
- Create custom tools with #[AsTool] attribute
- Add parameter validation with #[With]
- Implement input processors (pre-processing)
- Implement output processors (post-processing)
- Add memory to agents (static or embedding-based)
- Use sub-agents for delegation
- Handle tool errors with ToolExecutionExceptionInterface
- Configure fault-tolerant toolboxes
- Work with tool events (ToolCallsExecuted, etc.)
- Test agents with MockAgent

### Chat & Conversations
Read **chat.md** when you need to:
- Choose a message store backend
- Implement conversation persistence
- Create custom message stores
- Manage chat sessions across requests
- Use managed stores with setup/drop commands

### Vector Stores & RAG
Read **store.md** when you need to:
- Choose a vector store backend (ChromaDB, Pinecone, PostgreSQL, etc.)
- Index documents for RAG
- Retrieve similar documents
- Implement semantic search
- Use SimilaritySearch tool with agents
- Create custom store implementations
- Configure indexers and retrievers

## Console Commands

```bash
# Test platform directly
php bin/console ai:platform:invoke openai gpt-4o-mini "Hello, AI!"

# Interactive chat with agent
php bin/console ai:agent:call default

# Vector store management
php bin/console ai:store:setup chromadb.documents
php bin/console ai:store:drop chromadb.documents --force
php bin/console ai:store:index default --source=/path/to/file.txt

# Message store management
php bin/console ai:message-store:setup cache.chat
php bin/console ai:message-store:drop cache.chat
```

## Common Patterns

### Multi-Agent System with Handoffs

```yaml
# config/packages/ai.yaml
ai:
    multi_agent:
        customer_service:
            orchestrator: 'analyzer'
            handoffs:
                tech_support: ['error', 'bug', 'crash', 'not working']
                billing: ['payment', 'invoice', 'subscription', 'refund']
                product_info: ['features', 'tutorial', 'how to']
            fallback: 'general_support'
```

### Agent with Static Memory

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

$memoryProvider = new StaticMemoryProvider(
    'User name is John Doe',
    'User prefers concise answers',
    'User timezone is Europe/Paris',
);
$memoryProcessor = new MemoryInputProcessor($memoryProvider);

$agent = new Agent($platform, 'gpt-4o-mini', [$memoryProcessor]);
```

### Tool with Multiple Methods

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool(name: 'weather_current', description: 'Get current weather', method: 'current')]
#[AsTool(name: 'weather_forecast', description: 'Get weather forecast', method: 'forecast')]
final readonly class WeatherTool
{
    public function current(float $latitude, float $longitude): array
    {
        // Return current weather
    }

    public function forecast(float $latitude, float $longitude, int $days = 7): array
    {
        // Return weather forecast
    }
}
```

### Structured Output (Data Extraction)

```php
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create($apiKey, eventDispatcher: $dispatcher);

class MathReasoning
{
    public function __construct(
        public array $steps,
        public string $finalAnswer,
    ) {}
}

$messages = new MessageBag(
    Message::forSystem('You are a math tutor.'),
    Message::ofUser('How can I solve 8x + 7 = -23?'),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => MathReasoning::class,
]);

$reasoning = $result->asObject(); // Instance of MathReasoning
```

## Important Notes

1. **Experimental Status** - All Symfony AI components are experimental (v0.6.0) and not covered by Symfony's Backward Compatibility Promise. APIs may change between versions.

2. **Provider Credentials** - Store API keys in environment variables, never hardcode them:
   ```bash
   # .env
   OPENAI_API_KEY=sk-...
   ANTHROPIC_API_KEY=sk-ant-...
   ```

3. **Token Costs** - Monitor token usage, especially with tools. Use `maxToolCalls` to prevent runaway costs:
   ```php
   $processor = new AgentProcessor($toolbox, maxToolCalls: 10);
   ```

4. **Testing** - Always test with mocks (MockAgent, InMemoryPlatform) before using real APIs to avoid unnecessary costs during development.

5. **Memory Management** - InMemory stores load all data into PHP memory. Use proper backends (Redis, PostgreSQL, ChromaDB) for production.

6. **Tool Design** - Keep tools focused and single-purpose. Use meaningful descriptions to help the LLM understand when to use each tool.

7. **RAG Best Practices** - Chunk documents appropriately (200-500 tokens) for better retrieval. Include metadata for filtering.

8. **Streaming** - For web applications, use a layer like Mercure or WebSockets to push streaming responses to the browser.

9. **Security** - Use #[IsGrantedTool] to restrict tool access based on user roles:
   ```php
   use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

   #[IsGrantedTool('ROLE_ADMIN')]
   #[AsTool('delete_user', 'Delete a user account')]
   final class DeleteUser { /* ... */ }
   ```

10. **Error Handling** - Implement ToolExecutionExceptionInterface for better error messages to the LLM:
    ```php
    class ResourceNotFoundException extends \RuntimeException implements ToolExecutionExceptionInterface
    {
        public function getToolCallResult(): string
        {
            return 'Resource not found. Please verify the ID.';
        }
    }
    ```

## Examples Repository

Full examples available at: https://github.com/symfony/ai/tree/main/examples

- Platform examples (chat, streaming, embeddings, structured output)
- Agent examples (tools, memory, RAG)
- Store examples (ChromaDB, Pinecone, PostgreSQL, etc.)
- Chat examples (various message stores)

## Version Information

- Current version: v0.6.0 (Released March 5, 2026)
- Symfony compatibility: 7.1+
- PHP requirement: 8.2+
- Status: Experimental

## Additional Resources

- [Official Documentation](https://symfony.com/doc/current/ai/)
- [GitHub Repository](https://github.com/symfony/ai)
- [Changelog](https://github.com/symfony/ai/blob/main/CHANGELOG.md)
- [Component: Platform](https://github.com/symfony/ai-platform)
- [Component: Agent](https://github.com/symfony/ai-agent)
- [Component: Chat](https://github.com/symfony/ai-chat)
- [Component: Store](https://github.com/symfony/ai-store)
