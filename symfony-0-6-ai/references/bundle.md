# Symfony AI Bundle Configuration Reference

This document covers the AI Bundle configuration, multi-agent orchestration, console commands, dependency injection patterns, and security features.

## Installation

```bash
composer require symfony/ai-bundle
```

The bundle automatically registers and configures all AI components (Platform, Agent, Chat, Store).

## Table of Contents

1. [Basic Configuration](#basic-configuration)
2. [Platform Configuration](#platform-configuration)
3. [Agent Configuration](#agent-configuration)
4. [Multi-Agent Orchestration](#multi-agent-orchestration)
5. [Vectorizer Configuration](#vectorizer-configuration)
6. [Store Configuration](#store-configuration)
7. [Indexer Configuration](#indexer-configuration)
8. [Retriever Configuration](#retriever-configuration)
9. [Chat Configuration](#chat-configuration)
10. [Console Commands](#console-commands)
11. [Dependency Injection](#dependency-injection)
12. [Security](#security)
13. [Profiler Integration](#profiler-integration)

---

## Basic Configuration

```yaml
# config/packages/ai.yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'

    agent:
        default:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
```

## Platform Configuration

### OpenAI

```yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
            http_client: 'app.custom_http_client' # Optional
```

### Anthropic (Claude)

```yaml
ai:
    platform:
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
            version: '2023-06-01' # Optional, API version
```

### Azure OpenAI

```yaml
ai:
    platform:
        azure_openai:
            api_key: '%env(AZURE_OPENAI_API_KEY)%'
            endpoint: '%env(AZURE_OPENAI_ENDPOINT)%'
            deployment: 'gpt-4-deployment-name'
            api_version: '2024-02-15-preview'
```

### AWS Bedrock

```yaml
ai:
    platform:
        aws_bedrock:
            region: 'us-east-1'
            credentials:
                key: '%env(AWS_ACCESS_KEY_ID)%'
                secret: '%env(AWS_SECRET_ACCESS_KEY)%'
```

### Google Gemini

```yaml
ai:
    platform:
        google:
            api_key: '%env(GOOGLE_API_KEY)%'
```

### Vertex AI

```yaml
ai:
    platform:
        vertex_ai:
            project_id: '%env(VERTEX_PROJECT_ID)%'
            location: 'us-central1'
            credentials: '%kernel.project_dir%/var/vertex-credentials.json'
```

### Mistral

```yaml
ai:
    platform:
        mistral:
            api_key: '%env(MISTRAL_API_KEY)%'
```

### Ollama (Local Models)

```yaml
ai:
    platform:
        ollama:
            host_url: 'http://localhost:11434'
            api_catalog: true # Enable dynamic model discovery
```

### Perplexity

```yaml
ai:
    platform:
        perplexity:
            api_key: '%env(PERPLEXITY_API_KEY)%'
```

### OpenRouter

```yaml
ai:
    platform:
        openrouter:
            api_key: '%env(OPENROUTER_API_KEY)%'
```

### Generic Platform (LiteLLM, etc.)

```yaml
ai:
    platform:
        generic:
            my_provider:
                base_url: 'https://api.example.com'
                api_key: '%env(CUSTOM_API_KEY)%'
                models:
                    - name: 'custom-model-1'
                      capabilities: ['text_generation']
                    - name: 'custom-embeddings-1'
                      capabilities: ['embeddings']
```

### Failover Configuration

```yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
        ollama:
            host_url: 'http://localhost:11434'

        failover:
            ollama_to_openai:
                platforms:
                    - 'ai.platform.ollama'
                    - 'ai.platform.openai'
                rate_limiter: 'limiter.failover_platform'

framework:
    rate_limiter:
        failover_platform:
            policy: 'sliding_window'
            limit: 100
            interval: '60 minutes'
```

## Agent Configuration

### Basic Agent

```yaml
ai:
    agent:
        default:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
```

### Agent with Prompt

**Simple string:**
```yaml
ai:
    agent:
        assistant:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
            prompt: 'You are a helpful programming assistant.'
```

**Advanced array format:**
```yaml
ai:
    agent:
        assistant:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
            prompt:
                text: 'You are a helpful programming assistant.'
                include_tools: true
                enable_translation: true
                translation_domain: 'ai_prompts'
```

**File-based prompt:**
```yaml
ai:
    agent:
        assistant:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
            prompt:
                file: '%kernel.project_dir%/prompts/assistant.txt'
```

### Model Configuration

**URL-style (simple):**
```yaml
ai:
    agent:
        creative:
            platform: 'ai.platform.openai'
            model: 'gpt-4o?temperature=0.9&max_output_tokens=2000'
```

**Structured format:**
```yaml
ai:
    agent:
        creative:
            platform: 'ai.platform.openai'
            model:
                name: 'gpt-4o'
                options:
                    temperature: 0.9
                    max_output_tokens: 2000
                    top_p: 0.95
```

### Agent with Static Memory

```yaml
ai:
    agent:
        personalized:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
            memory: 'User prefers concise answers. User timezone is Europe/Paris.'
```

### Agent with Dynamic Memory (Service)

```yaml
ai:
    agent:
        personalized:
            platform: 'ai.platform.openai'
            model: 'gpt-4o-mini'
            memory:
                service: 'app.memory.user_context'
```

Service implementation:
```php
namespace App\Service;

use Symfony\AI\Agent\Memory\Memory;
use Symfony\AI\Agent\Memory\MemoryProviderInterface;
use Symfony\AI\Agent\Input;

final readonly class UserContextMemory implements MemoryProviderInterface
{
    public function __construct(
        private UserRepository $userRepository,
    ) {}

    public function loadMemory(Input $input): array
    {
        $user = $this->security->getUser();

        return [
            new Memory(sprintf('Username: %s', $user->getUsername())),
            new Memory(sprintf('Timezone: %s', $user->getTimezone())),
            new Memory(sprintf('Language: %s', $user->getLocale())),
        ];
    }
}
```

### Agent with Tools

```yaml
ai:
    agent:
        research:
            platform: 'ai.platform.openai'
            model: 'gpt-4o'
            tools:
                - 'App\Tool\WebSearchTool'
                - 'App\Tool\DatabaseQueryTool'
                - 'Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch'
```

Tools are automatically discovered via the `#[AsTool]` attribute. See [agent.md](agent.md) for tool creation details.

## Multi-Agent Orchestration

### Handoff Pattern

```yaml
ai:
    multi_agent:
        customer_service:
            orchestrator: 'analyzer'
            handoffs:
                tech_support:
                    - 'error'
                    - 'bug'
                    - 'crash'
                    - 'not working'
                    - 'broken'
                billing:
                    - 'payment'
                    - 'invoice'
                    - 'subscription'
                    - 'refund'
                    - 'charge'
                product_info:
                    - 'features'
                    - 'how to'
                    - 'tutorial'
                    - 'guide'
            fallback: 'general_support'
```

The `analyzer` agent examines the user query and routes it to the appropriate specialized agent based on keywords. If no keywords match, it falls back to `general_support`.

**Usage:**
```php
use Symfony\AI\Agent\AgentInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final readonly class SupportController
{
    public function __construct(
        #[Autowire(service: 'ai.multi_agent.customer_service')]
        private AgentInterface $supportAgent,
    ) {}

    public function ask(string $question): string
    {
        $messages = new MessageBag(Message::ofUser($question));
        return $this->supportAgent->call($messages)->getContent();
    }
}
```

## Vectorizer Configuration

```yaml
ai:
    vectorizer:
        openai_small:
            platform: 'ai.platform.openai'
            model:
                name: 'text-embedding-3-small'
                options:
                    dimensions: 512

        openai_large:
            platform: 'ai.platform.openai'
            model: 'text-embedding-3-large'

        voyage:
            platform: 'ai.platform.voyage'
            model: 'voyage-3'
```

## Store Configuration

### ChromaDB

```yaml
ai:
    store:
        chromadb:
            documents:
                collection: 'symfony_docs'
                host: 'http://localhost:8000'
```

### PostgreSQL (pgvector)

```yaml
ai:
    store:
        postgres:
            articles:
                dsn: '%env(DATABASE_URL)%'
                table: 'article_embeddings'
                dimensions: 1536
```

### Pinecone

```yaml
ai:
    store:
        pinecone:
            knowledge_base:
                api_key: '%env(PINECONE_API_KEY)%'
                environment: 'us-east-1-aws'
                index: 'knowledge-base'
```

### Weaviate

```yaml
ai:
    store:
        weaviate:
            docs:
                host: 'http://localhost:8080'
                class: 'Documentation'
```

### Qdrant

```yaml
ai:
    store:
        qdrant:
            vectors:
                host: 'http://localhost:6333'
                collection: 'embeddings'
```

### Meilisearch

```yaml
ai:
    store:
        meilisearch:
            search:
                host: 'http://localhost:7700'
                api_key: '%env(MEILISEARCH_KEY)%'
                index: 'documents'
```

### MongoDB

```yaml
ai:
    store:
        mongodb:
            atlas:
                dsn: '%env(MONGODB_URL)%'
                database: 'ai_store'
                collection: 'vectors'
```

### Redis

```yaml
ai:
    store:
        redis:
            cache:
                dsn: '%env(REDIS_URL)%'
                prefix: 'vectors:'
```

### InMemory (Testing Only)

```yaml
ai:
    store:
        memory:
            test:
                strategy: 'cosine' # or 'manhattan', 'euclidean'
```

### Cache (Symfony Cache)

```yaml
ai:
    store:
        cache:
            documents:
                service: 'cache.app'
                cache_key: 'ai_vectors'
```

## Indexer Configuration

```yaml
ai:
    indexer:
        documentation:
            loader: 'Symfony\AI\Store\Document\Loader\TextFileLoader'
            vectorizer: 'ai.vectorizer.openai_small'
            store: 'ai.store.chromadb.documents'

        articles:
            loader: 'App\Loader\ArticleLoader'
            vectorizer: 'ai.vectorizer.openai_large'
            store: 'ai.store.postgres.articles'
```

Custom loader:
```php
namespace App\Loader;

use Symfony\AI\Store\Document\DocumentLoaderInterface;
use Symfony\AI\Store\Document\TextDocument;

final readonly class ArticleLoader implements DocumentLoaderInterface
{
    public function __construct(
        private ArticleRepository $repository,
    ) {}

    public function load(string $source, array $options = []): iterable
    {
        $articles = $this->repository->findAll();

        foreach ($articles as $article) {
            yield new TextDocument(
                content: $article->getContent(),
                metadata: [
                    'id' => $article->getId(),
                    'title' => $article->getTitle(),
                    'category' => $article->getCategory(),
                ]
            );
        }
    }
}
```

## Retriever Configuration

```yaml
ai:
    retriever:
        default:
            vectorizer: 'ai.vectorizer.openai_small'
            store: 'ai.store.chromadb.documents'

        articles:
            vectorizer: 'ai.vectorizer.openai_large'
            store: 'ai.store.postgres.articles'
```

## Chat Configuration

### Message Stores

**Cache-based:**
```yaml
ai:
    message_store:
        cache:
            support_chat:
                service: 'cache.app'
                key: 'chat_{session_id}'
```

**Doctrine DBAL:**
```yaml
ai:
    message_store:
        doctrine:
            persistent_chat:
                connection: 'default'
                table: 'chat_messages'
```

**Redis:**
```yaml
ai:
    message_store:
        redis:
            realtime_chat:
                dsn: '%env(REDIS_URL)%'
                key_prefix: 'chat:'
```

**MongoDB:**
```yaml
ai:
    message_store:
        mongodb:
            chat_history:
                dsn: '%env(MONGODB_URL)%'
                database: 'chat'
                collection: 'messages'
```

**Session:**
```yaml
ai:
    message_store:
        session:
            current_chat:
                session_key: 'ai_chat'
```

### Chat Service

```yaml
ai:
    chat:
        support:
            agent: 'ai.agent.default'
            message_store: 'ai.message_store.cache.support_chat'

        research:
            agent: 'ai.agent.research'
            message_store: 'ai.message_store.doctrine.persistent_chat'
```

**Usage:**
```php
use Symfony\AI\Chat\ChatInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final readonly class ChatController
{
    public function __construct(
        #[Autowire(service: 'ai.chat.support')]
        private ChatInterface $chat,
    ) {}

    public function message(string $content): string
    {
        $this->chat->submit(Message::ofUser($content));
        // Messages are automatically persisted
    }
}
```

## Console Commands

### Platform Commands

**Invoke platform directly:**
```bash
php bin/console ai:platform:invoke openai gpt-4o-mini "What is Symfony?"
```

### Agent Commands

**Interactive chat:**
```bash
php bin/console ai:agent:call default
php bin/console ai:agent:call research
```

### Store Commands

**Setup store:**
```bash
php bin/console ai:store:setup chromadb.documents
php bin/console ai:store:setup postgres.articles
```

**Drop store:**
```bash
php bin/console ai:store:drop chromadb.documents --force
```

**Index documents:**
```bash
# From file
php bin/console ai:store:index documentation --source=/path/to/file.txt

# From directory (recursively)
php bin/console ai:store:index documentation --source=/path/to/docs/
```

### Message Store Commands

**Setup message store:**
```bash
php bin/console ai:message-store:setup doctrine.persistent_chat
```

**Drop message store:**
```bash
php bin/console ai:message-store:drop doctrine.persistent_chat
```

## Dependency Injection

### Autowiring Agents

```php
use Symfony\AI\Agent\AgentInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final readonly class MyService
{
    public function __construct(
        // Autowire by name
        #[Autowire(service: 'ai.agent.default')]
        private AgentInterface $defaultAgent,

        #[Autowire(service: 'ai.agent.research')]
        private AgentInterface $researchAgent,
    ) {}
}
```

### Autowiring Platforms

```php
use Symfony\AI\Platform\PlatformInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final readonly class MyService
{
    public function __construct(
        #[Autowire(service: 'ai.platform.openai')]
        private PlatformInterface $openai,

        #[Autowire(service: 'ai.platform.anthropic')]
        private PlatformInterface $anthropic,
    ) {}
}
```

### Store Aliasing

Multiple store aliases are available based on configuration:

```yaml
ai:
    store:
        memory:
            main:
                strategy: 'cosine'
            products:
                strategy: 'manhattan'
        chromadb:
            main:
                collection: 'documents'
```

Available autowiring aliases:
- `StoreInterface $main` - First occurrence (memory.main)
- `StoreInterface $memoryMain` - Explicit type + name
- `StoreInterface $chromadbMain` - chromadb.main
- `StoreInterface $products` - memory.products

```php
use Symfony\AI\Store\StoreInterface;

final readonly class MyService
{
    public function __construct(
        StoreInterface $main,           // memory.main
        StoreInterface $chromadbMain,   // chromadb.main
        StoreInterface $products,       // memory.products
    ) {}
}
```

### Vectorizer Autowiring

```php
use Symfony\AI\Store\VectorizerInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final readonly class MyService
{
    public function __construct(
        #[Autowire(service: 'ai.vectorizer.openai_small')]
        private VectorizerInterface $vectorizer,
    ) {}
}
```

### Chat Autowiring

```php
use Symfony\AI\Chat\ChatInterface;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final readonly class MyService
{
    public function __construct(
        #[Autowire(service: 'ai.chat.support')]
        private ChatInterface $supportChat,
    ) {}
}
```

## Security

### Tool-Level Security

Restrict tool access based on user roles:

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

#[IsGrantedTool('ROLE_ADMIN')]
#[AsTool('delete_user', 'Delete a user account')]
final readonly class DeleteUserTool
{
    public function __invoke(int $userId): string
    {
        // Only accessible to ROLE_ADMIN
        $this->userManager->delete($userId);
        return 'User deleted successfully.';
    }
}
```

**Multiple roles (OR logic):**
```php
#[IsGrantedTool(['ROLE_ADMIN', 'ROLE_MODERATOR'])]
#[AsTool('ban_user', 'Ban a user')]
final readonly class BanUserTool { /* ... */ }
```

### Agent-Level Security

Configure security at the agent level:

```yaml
# config/packages/security.yaml
security:
    access_control:
        - { path: ^/api/ai, roles: ROLE_USER }
```

Use security voters for fine-grained control:

```php
namespace App\Security\Voter;

use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

final class AgentAccessVoter extends Voter
{
    protected function supports(string $attribute, mixed $subject): bool
    {
        return $attribute === 'AGENT_ACCESS' && $subject instanceof AgentInterface;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();

        // Custom logic here
        return $user->hasAgentAccess();
    }
}
```

## Profiler Integration

The AI Bundle includes a profiler panel showing:

- **Agent Execution Details** - Call traces, tools used
- **Token Usage** - Prompt tokens, completion tokens, total cost estimation
- **Performance Metrics** - Execution time, API latency
- **Tool Invocations** - Which tools were called and results

Access metadata programmatically:

```php
use Symfony\AI\Platform\Result\Metadata\TokenUsage\TokenUsage;

$result = $agent->call($messages);
$metadata = $result->getMetadata();

if ($metadata->has('token_usage')) {
    $tokenUsage = $metadata->get('token_usage');
    echo "Prompt tokens: " . $tokenUsage->promptTokens . PHP_EOL;
    echo "Completion tokens: " . $tokenUsage->completionTokens . PHP_EOL;
    echo "Total tokens: " . $tokenUsage->totalTokens . PHP_EOL;
}
```

## Configuration Validation

The bundle validates configuration at container compile time. Common errors:

**Missing API key:**
```
Configuration error: The path "ai.platform.openai.api_key" is required.
```

**Invalid platform reference:**
```
Configuration error: The platform "ai.platform.invalid" does not exist.
```

**Invalid tool class:**
```
Configuration error: Tool class "App\Tool\InvalidTool" does not have the #[AsTool] attribute.
```

## Environment Variables

```bash
# .env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=...
MISTRAL_API_KEY=...
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_ENDPOINT=https://....openai.azure.com/
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
VERTEX_PROJECT_ID=...
PINECONE_API_KEY=...
REDIS_URL=redis://localhost:6379
DATABASE_URL=postgresql://user:pass@localhost/db
MONGODB_URL=mongodb://localhost:27017
```

## Best Practices

1. **Use Environment Variables** - Never hardcode API keys
2. **Configure Failover** - Set up fallback platforms for production reliability
3. **Enable Profiler in Dev** - Monitor token usage and performance
4. **Use Managed Stores** - Run setup commands for stores that need initialization
5. **Organize Agents** - Create specialized agents for different domains
6. **Tool Security** - Always use #[IsGrantedTool] for sensitive operations
7. **Cache Results** - Use Symfony Cache for repeated queries
8. **Monitor Costs** - Track token usage with profiler and set maxToolCalls limits

## Complete Example

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
        failover:
            primary:
                platforms:
                    - 'ai.platform.ollama'
                    - 'ai.platform.openai'
                    - 'ai.platform.anthropic'
                rate_limiter: 'limiter.ai'

    vectorizer:
        embeddings:
            platform: 'ai.platform.openai'
            model: 'text-embedding-3-small'

    store:
        chromadb:
            docs:
                collection: 'documentation'

    indexer:
        docs:
            loader: 'Symfony\AI\Store\Document\Loader\TextFileLoader'
            vectorizer: 'ai.vectorizer.embeddings'
            store: 'ai.store.chromadb.docs'

    retriever:
        docs:
            vectorizer: 'ai.vectorizer.embeddings'
            store: 'ai.store.chromadb.docs'

    agent:
        assistant:
            platform: 'ai.platform.failover.primary'
            model: 'gpt-4o-mini'
            prompt: 'You are a helpful programming assistant.'
            tools:
                - 'Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch'

    message_store:
        cache:
            main:
                service: 'cache.app'

    chat:
        assistant:
            agent: 'ai.agent.assistant'
            message_store: 'ai.message_store.cache.main'

framework:
    rate_limiter:
        ai:
            policy: 'sliding_window'
            limit: 1000
            interval: '1 hour'
```
