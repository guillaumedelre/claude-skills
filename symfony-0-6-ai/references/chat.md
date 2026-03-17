# Symfony AI Chat Reference

This document covers the Chat component, message stores, persistence mechanisms, managed stores, console commands, and best practices for building conversational AI applications.

## Table of Contents

1. [Chat Architecture](#chat-architecture)
2. [Chat Class](#chat-class)
3. [Message Stores Interface](#message-stores-interface)
4. [Supported Message Stores](#supported-message-stores)
5. [Store Configuration](#store-configuration)
6. [Creating Custom Stores](#creating-custom-stores)
7. [Managed Stores](#managed-stores)
8. [Console Commands](#console-commands)
9. [Usage Examples](#usage-examples)
10. [Best Practices](#best-practices)

---

## Chat Architecture

### Core Components

The Chat component is built to enable multi-turn conversations with persistent message history:

**Chat** - Orchestrator class that manages conversation flow with an Agent and persists messages to a store.

**MessageStoreInterface** - Contract for storing and retrieving messages across requests.

**ManagedStoreInterface** - Extension for stores that require initialization/teardown (databases, external services).

**MessageBag** - Container for message collections that flow through the chat system.

**Agent** - The underlying AI agent that processes each message turn.

### Basic Flow

```
User Input
    ↓
Chat.submit(Message)
    ↓
Load previous messages from store
    ↓
Merge with new message
    ↓
Agent.call(MessageBag)
    ↓
Save all messages to store
    ↓
Return response
```

Each submit() call loads the conversation history, adds the new user message, gets the AI response, and persists everything.

---

## Chat Class

### Creating a Chat Instance

```php
use Symfony\AI\Chat\Chat;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Cache\Store as CacheStore;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

// Create platform and agent
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini');

// Create message store
$cache = new FilesystemAdapter();
$messageStore = new CacheStore($cache, 'chat_session_123');

// Create chat with session identifier
$chat = new Chat($agent, $messageStore);
```

### Submit Method Signature

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// Submit a user message and get AI response
$response = $chat->submit(Message::ofUser('What is Symfony?'));

// Access the response
echo $response->getContent(); // AI's text response
echo $response->getRole();    // 'assistant'

// The Chat automatically:
// 1. Loads conversation history from store
// 2. Adds your message to the history
// 3. Calls the agent
// 4. Saves all messages (user + assistant) back to store
```

### Getting Conversation History

```php
use Symfony\AI\Chat\Chat;

// After calling submit(), retrieve the full conversation
$messages = $chat->getMessages(); // MessageBag with all messages

foreach ($messages as $message) {
    echo $message->getRole() . ': ' . $message->getContent() . PHP_EOL;
}

// Output:
// system: You are a helpful assistant.
// user: What is Symfony?
// assistant: Symfony is a PHP web application framework...
// user: Tell me more about its components.
// assistant: Symfony provides many components including...
```

### Chat with Custom System Prompt

```php
use Symfony\AI\Chat\Chat;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// Inject system message into initial conversation
$initialMessages = new MessageBag(
    Message::forSystem('You are a Symfony expert. Answer only with Symfony-related information.'),
);

$chat = new Chat($agent, $messageStore, $initialMessages);

// Now all messages are prefixed with this system message
$response = $chat->submit(Message::ofUser('What is DI?'));
```

### Chat Integration with Agent

The Chat component integrates seamlessly with any Agent:

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Chat\Chat;

// Agent with tools
$toolbox = new Toolbox([
    new SearchArticles(),
    new GetDocumentation(),
]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o', [$processor], [$processor]);

// Chat with tool-enabled agent
$chat = new Chat($agent, $messageStore);
$response = $chat->submit(Message::ofUser('Find articles about caching'));

// The agent can use tools, and all messages (including tool calls) are persisted
```

---

## Message Stores Interface

### MessageStoreInterface Contract

All message stores implement the core interface:

```php
namespace Symfony\AI\Chat\Store;

interface MessageStoreInterface
{
    /**
     * Load messages for a conversation.
     *
     * @return MessageBag containing all stored messages
     */
    public function load(): MessageBag;

    /**
     * Save messages for a conversation.
     *
     * @param MessageBag $messages All messages to persist
     */
    public function save(MessageBag $messages): void;
}
```

### load() Method

```php
// Load retrieves all messages previously stored
$messages = $messageStore->load();

// Returns MessageBag (may be empty for new conversations)
foreach ($messages as $message) {
    echo $message->getRole() . PHP_EOL; // 'system', 'user', 'assistant', 'tool'
}

// Implementations return a fresh MessageBag each time
// Changes to returned bag don't affect storage until save() is called
```

### save() Method

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// Save persists the entire conversation
$messages = new MessageBag(
    Message::forSystem('You are a helpful assistant.'),
    Message::ofUser('What is Symfony?'),
    Message::ofAssistant('Symfony is a PHP framework...'),
);

$messageStore->save($messages);

// After save(), load() will return these messages
// All subsequent messages are appended
```

---

## Supported Message Stores

Symfony AI includes 10+ built-in message stores for different use cases:

| Store | Class | Backend | Persistence | Best For | Session Scoped |
|-------|-------|---------|-------------|----------|---|
| **InMemory** | `InMemoryStore` | PHP memory | Single request | Testing, demo apps | No |
| **Cache** | `Cache\Store` | Any PSR-6 cache | Configurable (Redis, File, etc.) | Most web apps | No |
| **Doctrine DBAL** | `Doctrine\Store` | SQL database | Permanent | Large scale, archival | No |
| **Session** | `Session\Store` | HTTP session | Session lifetime | Multi-page forms, simple apps | Yes |
| **MongoDB** | `MongoDB\Store` | MongoDB | Permanent | Document-oriented, scalable | No |
| **Redis** | `Redis\Store` | Redis | Configurable TTL | High performance, multi-user | No |
| **Cloudflare KV** | `CloudflareKv\Store` | Cloudflare KV | Global, distributed | Edge computing, Workers | No |
| **Meilisearch** | `Meilisearch\Store` | Meilisearch | Searchable, persistent | Message search, analytics | No |
| **Pogocache** | `Pogocache\Store` | Pogocache | TTL-based | Embedded, lightweight | No |
| **SurrealDB** | `SurrealDB\Store` | SurrealDB | Distributed | Real-time, edge computing | No |

### Quick Selection Guide

- **Testing/Development**: InMemoryStore (no config needed)
- **Simple Web Apps**: Session\Store (per-user, automatic cleanup)
- **Production (High Performance)**: Redis\Store (fast, reliable)
- **Production (Permanent Record)**: Doctrine\Store (SQL database)
- **Serverless/Edge**: CloudflareKv\Store or SurrealDB\Store
- **Document Databases**: MongoDB\Store

---

## Store Configuration

### InMemory Store (Testing)

```php
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;
use Symfony\AI\Chat\Chat;

// Lives only in current PHP execution
// Perfect for unit tests and command-line demos
$store = new InMemoryStore();
$chat = new Chat($agent, $store);

// No configuration needed
// Messages disappear when PHP process ends
```

### Cache Store (Most Common)

```yaml
# config/packages/cache.yaml
framework:
    cache:
        default: cache.app
        pools:
            cache.app: &default_cache_pool
                adapter: cache.adapter.redis
                default_lifetime: 3600
            cache.chat:
                adapter: cache.adapter.redis
                default_lifetime: 86400 # 24 hours
```

```php
use Symfony\AI\Chat\Cache\Store as CacheStore;
use Symfony\AI\Chat\Chat;

// Inject PSR-6 cache adapter
final readonly class ChatController
{
    public function __construct(
        private AgentInterface $agent,
        private CacheInterface $chatCache, // autowired from cache.chat pool
    ) {}

    public function chat(): Response
    {
        $sessionId = $this->getUser()->getId(); // or request session ID
        $store = new CacheStore($this->chatCache, "chat_$sessionId");
        $chat = new Chat($this->agent, $store);

        $response = $chat->submit(Message::ofUser($_POST['message']));
        return new JsonResponse(['response' => $response->getContent()]);
    }
}
```

### Session Store (Simple Web Apps)

```php
use Symfony\AI\Chat\Session\Store as SessionStore;
use Symfony\AI\Chat\Chat;
use Symfony\Component\HttpFoundation\Session\SessionInterface;

final readonly class ChatController
{
    public function __construct(
        private AgentInterface $agent,
    ) {}

    public function chat(SessionInterface $session): Response
    {
        // Messages automatically stored in $_SESSION
        $store = new SessionStore($session, 'conversation');
        $chat = new Chat($this->agent, $store);

        $response = $chat->submit(Message::ofUser($_POST['message']));
        return new Response($response->getContent());
    }
}
```

### Doctrine DBAL Store (Permanent Record)

```yaml
# config/packages/ai.yaml
ai:
    message_store:
        doctrine:
            connection: default
            table: ai_messages
```

```php
use Symfony\AI\Chat\Doctrine\Store as DoctrineStore;
use Doctrine\DBAL\Connection;

// Automatically creates table if needed (via migrations)
$store = new DoctrineStore(
    connection: $connection,
    tableName: 'ai_messages',
    conversationId: 'user_' . $userId,
);

$chat = new Chat($agent, $store);
$chat->submit(Message::ofUser('Hello'));

// Messages permanently stored in database
// Survives application restarts
```

### Redis Store (High Performance)

```php
use Symfony\AI\Chat\Redis\Store as RedisStore;
use Redis;

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$store = new RedisStore(
    redis: $redis,
    key: "chat:session:$sessionId",
    ttl: 86400, // 24 hours
);

$chat = new Chat($agent, $store);
$response = $chat->submit(Message::ofUser('What is Redis?'));

// Messages stored in Redis with automatic expiration
```

### MongoDB Store (Document Database)

```php
use Symfony\AI\Chat\MongoDB\Store as MongoDBStore;
use MongoDB\Client;

$client = new Client('mongodb://localhost:27017');
$db = $client->selectDatabase('mezzo_chat');
$collection = $db->selectCollection('conversations');

$store = new MongoDBStore(
    collection: $collection,
    conversationId: "user_$userId",
);

$chat = new Chat($agent, $store);
$response = $chat->submit(Message::ofUser('Tell me a story'));

// Each conversation stored as a document
// Full text search and indexing available
```

### Cloudflare KV Store (Edge/Workers)

```php
use Symfony\AI\Chat\CloudflareKv\Store as CloudflareKvStore;

// Use in Cloudflare Worker environments
$store = new CloudflareKvStore(
    namespace: KV::CHAT,
    key: "conversation:$conversationId",
    ttl: 86400,
);

$chat = new Chat($agent, $store);

// Messages replicated globally at edge locations
// Very low latency worldwide
```

### SurrealDB Store (Real-time/Edge)

```php
use Symfony\AI\Chat\SurrealDB\Store as SurrealDBStore;
use Surrealdb\Surreal;

$db = new Surreal('http://localhost:8000');
$db->signin(['user' => 'root', 'pass' => 'root']);
$db->use('chat', 'default');

$store = new SurrealDBStore(
    db: $db,
    conversationId: "session_$sessionId",
);

$chat = new Chat($agent, $store);

// Real-time sync across multiple clients
// Distributed edge deployment
```

---

## Creating Custom Stores

### Implementing MessageStoreInterface

```php
namespace App\Chat\Store;

use Symfony\AI\Chat\Store\MessageStoreInterface;
use Symfony\AI\Platform\Message\MessageBag;

final class CustomStore implements MessageStoreInterface
{
    public function __construct(
        private string $conversationId,
        private SomeBackend $backend,
    ) {}

    public function load(): MessageBag
    {
        // Retrieve messages from custom backend
        $rawMessages = $this->backend->get("conv:$this->conversationId");

        if (!$rawMessages) {
            return new MessageBag(); // Empty conversation
        }

        // Reconstruct MessageBag from stored data
        $messages = new MessageBag();
        foreach ($rawMessages as $data) {
            $messages->add(Message::from([
                'role' => $data['role'],
                'content' => $data['content'],
                'toolCalls' => $data['tool_calls'] ?? null,
            ]));
        }

        return $messages;
    }

    public function save(MessageBag $messages): void
    {
        // Convert messages to storage format
        $data = [];
        foreach ($messages as $message) {
            $data[] = [
                'role' => $message->getRole(),
                'content' => $message->getContent(),
                'tool_calls' => $message->getToolCalls(),
                'timestamp' => time(),
            ];
        }

        // Persist to custom backend
        $this->backend->set("conv:$this->conversationId", $data, ttl: 604800); // 7 days
    }
}
```

### Using Custom Store

```php
use App\Chat\Store\CustomStore;
use Symfony\AI\Chat\Chat;

$store = new CustomStore($conversationId, $customBackend);
$chat = new Chat($agent, $store);

// Works exactly like built-in stores
$response = $chat->submit(Message::ofUser('Hello'));
```

### Custom Store with Metadata

```php
namespace App\Chat\Store;

interface MetadataStore extends MessageStoreInterface
{
    public function getMetadata(string $key, mixed $default = null): mixed;
    public function setMetadata(string $key, mixed $value): void;
}

final class EnhancedStore implements MetadataStore
{
    private array $metadata = [];

    public function load(): MessageBag
    {
        // Load messages
    }

    public function save(MessageBag $messages): void
    {
        // Save messages

        // Also save metadata
        $this->backend->set("meta:$this->conversationId", $this->metadata);
    }

    public function getMetadata(string $key, mixed $default = null): mixed
    {
        return $this->metadata[$key] ?? $default;
    }

    public function setMetadata(string $key, mixed $value): void
    {
        $this->metadata[$key] = $value;
    }
}

// Usage
$store = new EnhancedStore($conversationId, $backend);
$store->setMetadata('language', 'fr');
$store->setMetadata('tone', 'formal');

$chat = new Chat($agent, $store);
```

---

## Managed Stores

Some stores require initialization and cleanup. These implement `ManagedStoreInterface`:

### ManagedStoreInterface Contract

```php
namespace Symfony\AI\Chat\Store;

interface ManagedStoreInterface extends MessageStoreInterface
{
    /**
     * Initialize store resources (create tables, collections, etc.)
     * Called once during setup, not on every request
     */
    public function setup(): void;

    /**
     * Clean up store resources (drop tables, delete data, etc.)
     * Called when tearing down test suites or debugging
     */
    public function drop(): void;
}
```

### Stores Implementing ManagedStoreInterface

- **Doctrine\Store** - Creates SQL table
- **MongoDB\Store** - Creates collection with indexes
- **Meilisearch\Store** - Creates searchable index
- **SurrealDB\Store** - Creates table definition

### Implementing Managed Store

```php
namespace App\Chat\Store;

use Symfony\AI\Chat\Store\ManagedStoreInterface;
use Symfony\AI\Platform\Message\MessageBag;

final class PostgresStore implements ManagedStoreInterface
{
    public function __construct(
        private \PDO $pdo,
        private string $conversationId,
    ) {}

    public function setup(): void
    {
        // Create table if not exists
        $this->pdo->exec(<<<SQL
            CREATE TABLE IF NOT EXISTS ai_conversations (
                id SERIAL PRIMARY KEY,
                conversation_id VARCHAR(255) NOT NULL,
                role VARCHAR(50) NOT NULL,
                content TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                UNIQUE(conversation_id, created_at)
            );
            CREATE INDEX IF NOT EXISTS idx_conv_id ON ai_conversations(conversation_id);
        SQL);
    }

    public function drop(): void
    {
        // Delete all messages for this conversation
        $stmt = $this->pdo->prepare('DELETE FROM ai_conversations WHERE conversation_id = ?');
        $stmt->execute([$this->conversationId]);
    }

    public function load(): MessageBag
    {
        $stmt = $this->pdo->prepare(
            'SELECT role, content FROM ai_conversations WHERE conversation_id = ? ORDER BY created_at'
        );
        $stmt->execute([$this->conversationId]);

        $messages = new MessageBag();
        foreach ($stmt->fetchAll(\PDO::FETCH_ASSOC) as $row) {
            $messages->add(Message::from([
                'role' => $row['role'],
                'content' => $row['content'],
            ]));
        }

        return $messages;
    }

    public function save(MessageBag $messages): void
    {
        $stmt = $this->pdo->prepare(
            'INSERT INTO ai_conversations (conversation_id, role, content) VALUES (?, ?, ?)'
        );

        foreach ($messages as $message) {
            $stmt->execute([
                $this->conversationId,
                $message->getRole(),
                $message->getContent(),
            ]);
        }
    }
}
```

### When to Use Managed Stores

- **Multi-user applications** - Permanent record of all conversations for auditing
- **Production systems** - Need persistent storage across application restarts
- **Data analysis** - Analyze conversation patterns and improve AI behavior
- **Compliance** - GDPR, HIPAA, or other regulatory requirements
- **Testing** - Isolate conversation data between test runs

---

## Console Commands

### ai:message-store:setup

Initialize a managed store (create tables, indexes, etc.):

```bash
# Set up a specific store
php bin/console ai:message-store:setup doctrine.chat

# Set up all configured stores
php bin/console ai:message-store:setup all

# With different connection
php bin/console ai:message-store:setup doctrine.chat --connection=secondary
```

### ai:message-store:drop

Drop store resources (dangerous, requires confirmation):

```bash
# Drop a specific store
php bin/console ai:message-store:drop doctrine.chat

# Skip confirmation (use in CI/CD)
php bin/console ai:message-store:drop doctrine.chat --force

# Drop all stores
php bin/console ai:message-store:drop all --force
```

### Example Setup in Tests

```php
use PHPUnit\Framework\TestCase;
use Symfony\AI\Chat\Store\ManagedStoreInterface;

final class ChatTest extends TestCase
{
    private ManagedStoreInterface $store;

    protected function setUp(): void
    {
        $this->store = new DoctrineStore($connection, 'test_conversations_' . uniqid());
        $this->store->setup(); // Create table
    }

    protected function tearDown(): void
    {
        $this->store->drop(); // Clean up
    }

    public function testChatPersistenceAcrossRequests(): void
    {
        $chat = new Chat($agent, $this->store);
        $chat->submit(Message::ofUser('First message'));

        // New chat instance, same store
        $chat2 = new Chat($agent, $this->store);
        $messages = $chat2->getMessages();

        $this->assertCount(3, $messages); // system + user + assistant
    }
}
```

---

## Usage Examples

### Simple Chat Application

```php
// src/Controller/ChatController.php
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Platform\Message\Message;
use Symfony\Component\Cache\CacheItem;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;

final class ChatController extends AbstractController
{
    public function __construct(
        private AgentInterface $agent,
        private CacheInterface $chatCache,
    ) {}

    public function message(Request $request): JsonResponse
    {
        // Get or create conversation ID (from session, URL param, etc.)
        $conversationId = $request->getSession()->getId();

        // Create message store
        $store = new CacheStore($this->chatCache, "chat:$conversationId");
        $chat = new Chat($this->agent, $store);

        // Get user message
        $userMessage = $request->request->get('message');

        // Submit to chat
        $response = $chat->submit(Message::ofUser($userMessage));

        // Return AI response
        return new JsonResponse([
            'response' => $response->getContent(),
            'role' => $response->getRole(),
        ]);
    }

    public function history(Request $request): JsonResponse
    {
        $conversationId = $request->getSession()->getId();
        $store = new CacheStore($this->chatCache, "chat:$conversationId");
        $chat = new Chat($this->agent, $store);

        $messages = $chat->getMessages();

        return new JsonResponse([
            'messages' => array_map(fn($m) => [
                'role' => $m->getRole(),
                'content' => $m->getContent(),
            ], iterator_to_array($messages)),
        ]);
    }
}
```

### Multi-Agent Conversation

```php
use Symfony\AI\Chat\Chat;
use Symfony\AI\Platform\Message\Message;

// Create agents with different personalities
$researchAgent = $container->get('ai.agent.research');
$reviewerAgent = $container->get('ai.agent.reviewer');

// Each has separate conversation history
$researchStore = new CacheStore($cache, 'research_session');
$reviewStore = new CacheStore($cache, 'review_session');

$research = new Chat($researchAgent, $researchStore);
$review = new Chat($reviewerAgent, $reviewStore);

// Multi-turn conversation
$topic = 'How to optimize Symfony performance';

$research->submit(Message::ofUser("Research: $topic"));
$researchResult = $research->submit(Message::ofUser('Summarize your findings'));

$review->submit(Message::ofUser("Review this: {$researchResult->getContent()}"));
$reviewResult = $review->submit(Message::ofUser('Rate the quality'));

echo $reviewResult->getContent();
```

### Chat with Tools (Agent Integration)

```php
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;

// Agent with tool capabilities
$toolbox = new Toolbox([
    new SearchDocumentation(),
    new QueryDatabase(),
    new FetchExternalApi(),
]);

$processor = new AgentProcessor($toolbox);
$toolAgent = new Agent($platform, 'gpt-4o', [$processor], [$processor]);

// Chat uses tool-enabled agent
$store = new DoctrineStore($connection, 'conversation_with_tools');
$store->setup();

$chat = new Chat($toolAgent, $store);

// Agent can now use tools while maintaining history
$response = $chat->submit(Message::ofUser('What are the latest Symfony changes?'));
// Agent uses SearchDocumentation tool, persists tool calls and results
```

---

## Best Practices

### Store Selection Guide

**For testing/development:**
```php
// InMemory is fastest, zero config
$store = new InMemoryStore();
```

**For single-page applications:**
```php
// Session store keeps data in user's session
$store = new SessionStore($session, 'chat');
```

**For multi-request web apps (production):**
```php
// Redis for performance, TTL-based cleanup
$store = new RedisStore($redis, "chat:$userId", ttl: 86400);
```

**For permanent record/compliance:**
```php
// Doctrine for SQL, MongoDB for documents
$store = new DoctrineStore($connection, 'conversations');
$store->setup();
```

**For serverless/edge:**
```php
// Cloudflare KV or SurrealDB for global distribution
$store = new CloudflareKvStore(KV::CHAT, "conv:$id");
```

### Performance Considerations

**Cache TTL Strategy:**
```php
// Short TTL for temporary chats (form wizards)
$tempStore = new CacheStore($cache, $id, ttl: 600); // 10 minutes

// Long TTL for ongoing conversations
$longStore = new CacheStore($cache, $id, ttl: 2592000); // 30 days
```

**Load Entire History Only When Needed:**
```php
// Good: Lazy load on first request
public function getConversation()
{
    $store = new CacheStore($cache, $id);
    $chat = new Chat($agent, $store);
    return $chat->getMessages(); // Loaded on demand
}

// Bad: Loading every message on every request
public function middleware()
{
    $this->fullHistory = $store->load(); // Every request pays this cost
}
```

**Batch Operations for Many Conversations:**
```php
// Use Doctrine bulk insert for batch saves
foreach ($conversations as $id => $messages) {
    $store = new DoctrineStore($connection, $id);
    $store->save($messages);
}
```

### Session Management

**Secure Session Handling:**
```php
// Rotate session ID after authentication
$session->migrate(true); // Creates new session, destroys old

// Then recreate chat with new session ID
$store = new SessionStore($session, 'chat');
$chat = new Chat($agent, $store);
```

**Per-User Conversations:**
```php
// Always scope to authenticated user
$conversationId = sprintf('user_%d_%s', $user->getId(), $session->getId());
$store = new CacheStore($cache, $conversationId);

// Prevents cross-user access
```

**Cleanup After Logout:**
```php
// Clear all user conversations
public function onLogout(LogoutEvent $event)
{
    $userId = $event->getToken()->getUser()->getId();
    $this->cache->deleteItem("chat:user:$userId");
}
```

### Production Recommendations

**Monitoring:**
```php
// Log slow store operations
final class LoggingStore implements MessageStoreInterface
{
    public function load(): MessageBag
    {
        $start = microtime(true);
        $messages = $this->store->load();
        $duration = microtime(true) - $start;

        if ($duration > 0.1) {
            $this->logger->warning("Slow store load: {$duration}s");
        }

        return $messages;
    }

    public function save(MessageBag $messages): void
    {
        $start = microtime(true);
        $this->store->save($messages);
        $duration = microtime(true) - $start;

        if ($duration > 0.1) {
            $this->logger->warning("Slow store save: {$duration}s");
        }
    }
}
```

**Fallback Strategy:**
```php
// Graceful degradation if primary store fails
final class FallbackStore implements MessageStoreInterface
{
    public function load(): MessageBag
    {
        try {
            return $this->primary->load();
        } catch (\Exception $e) {
            $this->logger->error('Primary store failed, using fallback', ['error' => $e->getMessage()]);
            return $this->fallback->load();
        }
    }

    public function save(MessageBag $messages): void
    {
        try {
            $this->primary->save($messages);
        } catch (\Exception $e) {
            $this->logger->error('Primary save failed, using fallback', ['error' => $e->getMessage()]);
            $this->fallback->save($messages);
        }
    }
}
```

**Data Privacy:**
```php
// Encrypt sensitive data before storage
final class EncryptedStore implements MessageStoreInterface
{
    public function save(MessageBag $messages): void
    {
        $encrypted = [];
        foreach ($messages as $message) {
            $encrypted[] = [
                'role' => $message->getRole(),
                'content' => $this->encrypt($message->getContent()),
            ];
        }

        $this->backend->save($encrypted);
    }

    public function load(): MessageBag
    {
        $encrypted = $this->backend->load();

        $messages = new MessageBag();
        foreach ($encrypted as $data) {
            $messages->add(Message::from([
                'role' => $data['role'],
                'content' => $this->decrypt($data['content']),
            ]));
        }

        return $messages;
    }

    private function encrypt(string $content): string
    {
        // Use sodium_crypto_secretbox or similar
        return base64_encode(sodium_crypto_secretbox($content, $this->nonce, $this->key));
    }

    private function decrypt(string $encrypted): string
    {
        return sodium_crypto_secretbox_open(base64_decode($encrypted), $this->nonce, $this->key);
    }
}
```

**Message Size Limits:**
```php
// Prevent runaway conversation growth
final class TruncatingStore implements MessageStoreInterface
{
    private const MAX_MESSAGES = 100;

    public function save(MessageBag $messages): void
    {
        $messages = $this->truncate($messages, self::MAX_MESSAGES);
        $this->backend->save($messages);
    }

    private function truncate(MessageBag $messages, int $maxMessages): MessageBag
    {
        $all = iterator_to_array($messages);

        if (count($all) > $maxMessages) {
            // Keep system message + most recent messages
            $system = array_filter($all, fn($m) => $m->getRole() === 'system');
            $others = array_filter($all, fn($m) => $m->getRole() !== 'system');

            $kept = array_slice($others, -($maxMessages - 1));
            return new MessageBag(...array_merge($system, $kept));
        }

        return $messages;
    }
}
```
