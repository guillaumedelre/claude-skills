# Symfony AI Store Reference

This document covers the Store component for vector storage and Retrieval-Augmented Generation (RAG), including all supported backends, indexing, retrieval, custom implementations, and best practices for semantic search applications.

## Table of Contents

1. [Store Purpose and RAG Overview](#store-purpose-and-rag-overview)
2. [Core Components](#core-components)
3. [Supported Backends](#supported-backends)
4. [Documents and Metadata](#documents-and-metadata)
5. [Indexing Documents](#indexing-documents)
6. [Retrieving Similar Documents](#retrieving-similar-documents)
7. [Custom Store Implementation](#custom-store-implementation)
8. [Console Commands](#console-commands)
9. [Configuration Examples](#configuration-examples)
10. [RAG Patterns](#rag-patterns)
11. [Best Practices](#best-practices)

---

## Store Purpose and RAG Overview

### What is RAG?

Retrieval-Augmented Generation (RAG) is a pattern that combines:

1. **Retrieval** - Find semantically similar documents from a knowledge base
2. **Augmentation** - Include those documents in the LLM prompt
3. **Generation** - LLM generates response with that context

This solves several LLM limitations:
- **Knowledge cutoff** - LLMs have training data from specific dates
- **Proprietary data** - Can't access private/proprietary information
- **Accuracy** - Reduces hallucinations by grounding responses in facts
- **Up-to-date** - Can index real-time data (news, tickets, documentation)

### Use Cases

**Knowledge Base Search**
```
User: "How do I create a Symfony console command?"
System: Retrieves 5 documents about console commands, includes them in prompt
LLM: Generates comprehensive answer based on your documentation
```

**FAQ and Support**
```
User: "Why is my upload failing?"
System: Retrieves similar support tickets and solutions
LLM: Provides relevant solution based on historical support cases
```

**Documentation Assistant**
```
User: "What middleware does Symfony have?"
System: Retrieves all middleware-related docs
LLM: Synthesizes comprehensive answer from multiple docs
```

**Product Recommendations**
```
User: "I need a database solution for time-series data"
System: Retrieves similar product queries and solutions
LLM: Recommends best-fit products based on patterns
```

**Research and Analysis**
```
User: "Summarize the latest cloud trends"
System: Retrieves relevant research papers and articles
LLM: Creates summary grounded in real sources
```

### How Vector Storage Works

Documents are converted to **embeddings** - numerical vectors (typically 384-1536 dimensions) that capture semantic meaning:

```
"The Symfony Console component creates CLI commands"
    ↓ (embedding model: OpenAI, Cohere, Ollama)
[0.234, -0.891, 0.456, ..., 0.123]  (1536 dimensions)

"Building command-line interfaces with Symfony"
    ↓
[0.241, -0.885, 0.461, ..., 0.129]  (similar vector)

Similarity: 0.98 (cosine distance)
```

The Store backend handles:
- Storing embeddings efficiently
- Fast similarity search (finding nearest neighbors)
- Optional metadata filtering
- Updating/deleting documents
- Scaling to millions of documents

---

## Core Components

### StoreInterface

All stores implement this contract:

```php
namespace Symfony\AI\Store;

interface StoreInterface
{
    /**
     * Add a document (with embedding) to the store
     */
    public function add(VectorDocument $document): string; // Returns document ID

    /**
     * Query for similar documents
     */
    public function query(array $embedding, array $options = []): array; // Returns VectorDocuments
}
```

### ManagedStoreInterface

Stores that need setup/teardown (databases, cloud services):

```php
interface ManagedStoreInterface extends StoreInterface
{
    /**
     * Initialize the store (create tables, indexes, etc.)
     */
    public function setup(): void;

    /**
     * Drop the store (delete tables, all data)
     */
    public function drop(): void;
}
```

### Indexer

Handles document processing and embedding generation:

```php
namespace Symfony\AI\Store;

class Indexer
{
    public function __construct(
        private PlatformInterface $platform,
        private string $embeddingModel, // e.g., 'text-embedding-3-small'
        private StoreInterface $store,
        private DocumentLoaderInterface $loader = null,
    ) {}

    /**
     * Index a document (embed and store)
     */
    public function index(DocumentInterface $document): void;

    /**
     * Batch index multiple documents
     */
    public function indexBatch(array $documents, int $batchSize = 10): void;

    /**
     * Load and index files
     */
    public function indexFromPath(string $path, array $options = []): void;
}
```

### Retriever

Searches for similar documents:

```php
class Retriever
{
    public function __construct(
        private VectorizerInterface $vectorizer,
        private StoreInterface $store,
    ) {}

    /**
     * Retrieve similar documents for a query
     */
    public function retrieve(string $query, array $options = []): array;
}
```

### Vectorizer

Converts text to embeddings:

```php
interface VectorizerInterface
{
    /**
     * Generate embedding for text
     */
    public function embed(string $text): array; // Returns float array

    /**
     * Get model name
     */
    public function getModel(): string;

    /**
     * Get vector dimension
     */
    public function getDimension(): int;
}
```

---

## Supported Backends

The Store component supports 18+ backends across multiple categories.

### Backend Comparison Table

| Backend | Type | Scale | Latency | Metadata | Cost | Setup |
|---------|------|-------|---------|----------|------|-------|
| **Chroma** | Search | 100K-1M | Low | Yes | Free | Local |
| **Meilisearch** | Search | 10M+ | Medium | Yes | Free | Local |
| **Qdrant** | Vector DB | 100M+ | Ultra-low | Yes | Free | Local |
| **Pinecone** | Cloud VDB | Unlimited | Low | Yes | Paid | SaaS |
| **Weaviate** | Vector DB | Unlimited | Low | Yes | Free/Paid | Local/Cloud |
| **PostgreSQL+pgvector** | Database | 10M+ | Medium | Yes | Free | Existing |
| **MongoDB** | Database | Unlimited | Medium | Yes | Paid | Cloud |
| **Neo4j** | Graph DB | 100M+ | Medium | Yes | Free/Paid | Local/Cloud |
| **Azure AI Search** | Cloud | Unlimited | Medium | Yes | Paid | SaaS |
| **OpenSearch** | Search | 100M+ | Medium | Yes | Free/Paid | Self-hosted |
| **Supabase** | Database | 10M+ | Medium | Yes | Free/Paid | Cloud |
| **Manticore** | Search | 1B+ | Fast | Yes | Free | Local |
| **InMemory** | Cache | 100K | Ultra-fast | Yes | N/A | N/A |
| **Cache** | Cache | 10K | Ultra-fast | No | N/A | N/A |

### Cloud Providers

#### Azure AI Search
```php
use Symfony\AI\Store\Azure\AzureAiSearchStore;

$store = new AzureAiSearchStore(
    endpoint: $_ENV['AZURE_SEARCH_ENDPOINT'],
    apiKey: $_ENV['AZURE_SEARCH_API_KEY'],
    indexName: 'documents',
);
```

Configuration:
```yaml
ai:
    store:
        azure_documents:
            class: 'Symfony\AI\Store\Azure\AzureAiSearchStore'
            arguments:
                endpoint: '%env(AZURE_SEARCH_ENDPOINT)%'
                apiKey: '%env(AZURE_SEARCH_API_KEY)%'
                indexName: 'documents'
```

#### Cloudflare Vectorize
```php
use Symfony\AI\Store\Cloudflare\CloudflareStore;

$store = new CloudflareStore(
    accountId: $_ENV['CLOUDFLARE_ACCOUNT_ID'],
    apiToken: $_ENV['CLOUDFLARE_API_TOKEN'],
    namespaceId: 'documents',
);
```

#### Pinecone
```php
use Symfony\AI\Store\Pinecone\PineconeStore;

$store = new PineconeStore(
    apiKey: $_ENV['PINECONE_API_KEY'],
    environment: 'us-west-4',
    projectName: 'default',
    index: 'documents',
    namespace: 'default',
);
```

#### Weaviate
```php
use Symfony\AI\Store\Weaviate\WeaviateStore;

$store = new WeaviateStore(
    host: 'localhost',
    port: 8080,
    scheme: 'http',
    className: 'Documents',
);
```

#### Supabase
```php
use Symfony\AI\Store\Supabase\SupabaseStore;

$store = new SupabaseStore(
    url: $_ENV['SUPABASE_URL'],
    key: $_ENV['SUPABASE_ANON_KEY'],
    tableName: 'documents',
);
```

### Search Engine Backends

#### ChromaDB
```php
use Symfony\AI\Store\Chroma\ChromaStore;

$store = new ChromaStore(
    host: 'localhost',
    port: 8000,
    collectionName: 'documents',
);
```

Configuration:
```yaml
ai:
    store:
        chroma_documents:
            class: 'Symfony\AI\Store\Chroma\ChromaStore'
            arguments:
                host: 'localhost'
                port: 8000
                collectionName: 'documents'
```

#### Meilisearch
```php
use Symfony\AI\Store\Meilisearch\MeilisearchStore;

$store = new MeilisearchStore(
    host: 'http://localhost:7700',
    apiKey: $_ENV['MEILISEARCH_API_KEY'],
    indexUid: 'documents',
);
```

#### Manticore
```php
use Symfony\AI\Store\Manticore\ManticoreStore;

$store = new ManticoreStore(
    dsn: 'mysql://localhost:9306',
    indexName: 'documents_idx',
);
```

#### OpenSearch
```php
use Symfony\AI\Store\OpenSearch\OpenSearchStore;

$store = new OpenSearchStore(
    hosts: ['localhost:9200'],
    index: 'documents',
    username: 'admin',
    password: $_ENV['OPENSEARCH_PASSWORD'],
);
```

#### Qdrant
```php
use Symfony\AI\Store\Qdrant\QdrantStore;

$store = new QdrantStore(
    host: 'localhost',
    port: 6333,
    collectionName: 'documents',
    vectorSize: 1536, // For OpenAI embeddings
);
```

#### Typesense
```php
use Symfony\AI\Store\Typesense\TypesenseStore;

$store = new TypesenseStore(
    apiKey: $_ENV['TYPESENSE_API_KEY'],
    host: 'localhost',
    port: 8108,
    collectionName: 'documents',
);
```

### Database Backends

#### PostgreSQL with pgvector
```php
use Symfony\AI\Store\PostgreSQL\PostgreSQLStore;

$store = new PostgreSQLStore(
    dsn: 'postgresql://user:password@localhost/mezzo_content',
    tableName: 'documents_vectors',
    embeddingDimension: 1536,
);

// Setup table on first run
if ($isFirstRun) {
    $store->setup();
}
```

Configuration with YAML:
```yaml
ai:
    store:
        postgres_documents:
            class: 'Symfony\AI\Store\PostgreSQL\PostgreSQLStore'
            arguments:
                dsn: '%env(DATABASE_URL)%'
                tableName: 'documents_vectors'
                embeddingDimension: 1536
```

#### MongoDB
```php
use Symfony\AI\Store\MongoDB\MongoDBStore;

$store = new MongoDBStore(
    uri: $_ENV['MONGODB_URI'],
    database: 'mezzo_content',
    collection: 'documents_vectors',
);
```

#### Neo4j
```php
use Symfony\AI\Store\Neo4j\Neo4jStore;

$store = new Neo4jStore(
    uri: 'bolt://localhost:7687',
    username: 'neo4j',
    password: $_ENV['NEO4J_PASSWORD'],
);
```

#### SurrealDB
```php
use Symfony\AI\Store\SurrealDB\SurrealDBStore;

$store = new SurrealDBStore(
    endpoint: 'http://localhost:8000',
    username: 'root',
    password: $_ENV['SURREAL_PASSWORD'],
    namespace: 'mezzo',
    database: 'content',
);
```

#### MariaDB
```php
use Symfony\AI\Store\MariaDB\MariaDBStore;

$store = new MariaDBStore(
    dsn: 'mysql://user:password@localhost/mezzo_content',
    tableName: 'documents_vectors',
    embeddingDimension: 1536,
);
```

### In-Memory Backends

#### InMemory Store
Loads all vectors into PHP memory. Use only for development/testing:

```php
use Symfony\AI\Store\InMemory\InMemoryStore;

// Small datasets only
$store = new InMemoryStore();

$indexer = new Indexer($platform, 'text-embedding-3-small', $store);
$indexer->index(new TextDocument('First document'));
$indexer->index(new TextDocument('Second document'));

// Query is instant
$retriever = new Retriever($vectorizer, $store);
$results = $retriever->retrieve('search query');
```

#### Cache Store
Uses Symfony Cache component:

```php
use Symfony\AI\Store\Cache\Store as CacheStore;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter();
$store = new CacheStore($cache, 'rag_documents');

// Good for small document sets with cache invalidation
```

---

## Documents and Metadata

### TextDocument

Simple text-based documents:

```php
use Symfony\AI\Store\Document\TextDocument;

$document = new TextDocument('Symfony is a PHP web framework');

// With metadata
$document = new TextDocument(
    'Symfony is a PHP web framework',
    metadata: [
        'source' => 'docs/intro',
        'version' => '7.1',
        'brand' => 'f24',
        'timestamp' => time(),
    ]
);
```

### VectorDocument

Document with pre-computed embedding:

```php
use Symfony\AI\Store\Document\VectorDocument;

$vectorDocument = new VectorDocument(
    id: 'doc-123',
    content: 'Symfony is a PHP web framework',
    embedding: $vectorArray, // Array of floats
    metadata: [
        'source' => 'docs/intro',
        'relevance' => 'core-concept',
    ]
);
```

### Document Metadata Best Practices

Use metadata for filtering and ranking:

```php
$document = new TextDocument(
    'Doctrine ORM tutorial for Symfony',
    metadata: [
        // For filtering
        'type' => 'tutorial',
        'brand' => 'f24',
        'language' => 'en',

        // For ranking
        'importance' => 1,
        'updated_at' => '2024-03-15',
        'category' => 'database',

        // For context
        'author' => 'symfony-team',
        'source_url' => 'https://example.com/docs',
        'section' => 'ORM Configuration',
    ]
);
```

### DocumentLoader

Load documents from files:

```php
use Symfony\AI\Store\Loader\DocumentLoader;
use Symfony\AI\Store\Loader\FilesystemDocumentLoader;

// Load from file system
$loader = new FilesystemDocumentLoader();
$documents = $loader->load('/path/to/docs');

// Load with options
$documents = $loader->load('/path/to/docs', [
    'extensions' => ['md', 'txt'],
    'recursive' => true,
]);
```

Custom document loaders:

```php
use Symfony\AI\Store\Loader\DocumentLoaderInterface;

class DatabaseDocumentLoader implements DocumentLoaderInterface
{
    public function __construct(
        private EntityManagerInterface $em,
    ) {}

    public function load(string $source): array
    {
        $articles = $this->em->getRepository(Article::class)->findAll();

        return array_map(
            fn(Article $article) => new TextDocument(
                content: $article->getTitle() . "\n" . $article->getBody(),
                metadata: [
                    'id' => $article->getId(),
                    'type' => 'article',
                    'brand' => $article->getBrand(),
                    'published' => $article->getPublishedAt()?->getTimestamp(),
                ]
            ),
            $articles
        );
    }
}
```

---

## Indexing Documents

### Basic Indexing

```php
use Symfony\AI\Store\Indexer;
use Symfony\AI\Store\Document\TextDocument;

// Create indexer
$indexer = new Indexer(
    platform: $platform,
    embeddingModel: 'text-embedding-3-small',
    store: $store,
);

// Index single document
$document = new TextDocument('Symfony is a PHP web framework');
$indexer->index($document);

// Multiple documents
$documents = [
    new TextDocument('Symfony Console component creates CLI commands'),
    new TextDocument('Doctrine ORM provides database abstraction'),
    new TextDocument('API Platform builds REST and GraphQL APIs'),
];

foreach ($documents as $doc) {
    $indexer->index($doc);
}
```

### Batch Indexing

For performance with large document sets:

```php
$documents = [
    new TextDocument('Document 1'),
    new TextDocument('Document 2'),
    // ... 1000 documents
];

// Batch embedding (10 documents at a time)
$indexer->indexBatch($documents, batchSize: 10);
```

### Indexing from Files

```php
// Index all markdown files in a directory
$indexer->indexFromPath('/path/to/docs', [
    'extensions' => ['md'],
    'recursive' => true,
]);

// Index from database
$loader = new DatabaseDocumentLoader($entityManager);
$documents = $loader->load('articles');
$indexer->indexBatch($documents);
```

### Chunking Strategy

For better retrieval, split long documents:

```php
class DocumentChunker
{
    public static function chunk(
        string $content,
        int $maxTokens = 300,
        int $overlap = 50,
    ): array {
        $sentences = preg_split('/[.!?]+/', $content);
        $chunks = [];
        $currentChunk = '';

        foreach ($sentences as $sentence) {
            $sentence = trim($sentence);
            $tokens = str_word_count($sentence);

            if ($tokens + str_word_count($currentChunk) > $maxTokens) {
                if ($currentChunk) {
                    $chunks[] = $currentChunk;
                    // Overlap for context
                    $currentChunk = substr(
                        $currentChunk,
                        -($overlap * 4) // Approximate
                    ) . ' ' . $sentence;
                }
            } else {
                $currentChunk .= ($currentChunk ? ' ' : '') . $sentence;
            }
        }

        if ($currentChunk) {
            $chunks[] = $currentChunk;
        }

        return $chunks;
    }
}

// Usage
$fullDocument = $documentation->getContent();
$chunks = DocumentChunker::chunk($fullDocument, maxTokens: 300);

foreach ($chunks as $chunk) {
    $indexer->index(new TextDocument(
        $chunk,
        metadata: [
            'source' => $documentation->getSource(),
            'chunked_from' => $documentation->getId(),
        ]
    ));
}
```

### Update and Delete

```php
// Update document (re-embed and re-store)
$updated = new TextDocument(
    'Updated content',
    metadata: ['id' => 'doc-123']
);
$indexer->index($updated);

// Delete from store (backend dependent)
// Most stores provide:
$store->delete('doc-123');
```

---

## Retrieving Similar Documents

### Basic Retrieval

```php
use Symfony\AI\Store\Retriever;

$retriever = new Retriever($vectorizer, $store);

// Retrieve similar documents
$documents = $retriever->retrieve(
    'How do I create Symfony console commands?',
    [
        'limit' => 5, // Return top 5
    ]
);

foreach ($documents as $doc) {
    echo $doc->getContent() . "\n";
    echo "Similarity: " . ($doc->getMetadata()['similarity'] ?? 'N/A') . "\n";
}
```

### Query Options

```php
// Control retrieval behavior
$documents = $retriever->retrieve('search query', [
    'limit' => 10,              // Number of results
    'threshold' => 0.7,         // Minimum similarity
    'filters' => [              // Metadata filtering
        'brand' => 'f24',
        'type' => 'tutorial',
    ],
    'similarity_metric' => 'cosine', // Metric (varies by backend)
]);
```

### Advanced Filtering

Different backends support different filter syntax:

```php
// PostgreSQL with pgvector
$documents = $retriever->retrieve('query', [
    'filters' => [
        'WHERE' => 'metadata->\'brand\' = \'f24\'',
    ],
]);

// MongoDB
$documents = $retriever->retrieve('query', [
    'filters' => [
        'metadata.brand' => 'f24',
        'metadata.type' => ['$in' => ['tutorial', 'guide']],
    ],
]);

// Meilisearch
$documents = $retriever->retrieve('query', [
    'filters' => 'brand = f24 AND type = tutorial',
]);
```

### Hybrid Search

Combine vector similarity with BM25 (full-text search):

```php
class HybridRetriever
{
    public function __construct(
        private Retriever $vectorRetriever,
        private FullTextSearchIndex $textIndex,
    ) {}

    public function retrieve(string $query, array $options = []): array
    {
        // Get vector results
        $vectorResults = $this->vectorRetriever->retrieve($query, $options);

        // Get full-text results
        $textResults = $this->textIndex->search($query, $options);

        // Combine and deduplicate
        $combined = [];
        $seen = [];

        foreach ($vectorResults as $doc) {
            $id = $doc->getId();
            if (!isset($seen[$id])) {
                $combined[] = $doc;
                $seen[$id] = true;
            }
        }

        foreach ($textResults as $doc) {
            $id = $doc->getId();
            if (!isset($seen[$id])) {
                $combined[] = $doc;
                $seen[$id] = true;
            }
        }

        return array_slice($combined, 0, $options['limit'] ?? 10);
    }
}
```

### Contextual Retrieval

Retrieve documents with context windows:

```php
class ContextualRetriever
{
    public function __construct(
        private Retriever $retriever,
        private EntityManagerInterface $em,
    ) {}

    public function retrieveWithContext(
        string $query,
        int $contextBefore = 1,
        int $contextAfter = 1,
    ): array {
        $results = $this->retriever->retrieve($query);

        foreach ($results as $doc) {
            $metadata = $doc->getMetadata();
            $articleId = $metadata['article_id'] ?? null;

            if ($articleId) {
                $article = $this->em->find(Article::class, $articleId);

                // Get sections before and after
                $currentSection = $metadata['section'] ?? 0;
                $sections = $article->getSections();

                $startIdx = max(0, $currentSection - $contextBefore);
                $endIdx = min(
                    count($sections) - 1,
                    $currentSection + $contextAfter
                );

                $contextContent = implode(
                    "\n",
                    array_slice($sections, $startIdx, $endIdx - $startIdx + 1)
                );

                // Add context to document
                $metadata['context'] = $contextContent;
                $doc->setMetadata($metadata);
            }
        }

        return $results;
    }
}
```

---

## Custom Store Implementation

### Implementing StoreInterface

```php
use Symfony\AI\Store\Contract\StoreInterface;
use Symfony\AI\Store\Document\VectorDocument;

class CustomVectorStore implements StoreInterface
{
    private array $documents = [];

    /**
     * Add document to store
     */
    public function add(VectorDocument $document): string
    {
        $id = $document->getId() ?? uniqid('doc_');

        $this->documents[$id] = [
            'id' => $id,
            'content' => $document->getContent(),
            'embedding' => $document->getEmbedding(),
            'metadata' => $document->getMetadata(),
        ];

        return $id;
    }

    /**
     * Query for similar documents
     */
    public function query(array $embedding, array $options = []): array
    {
        $limit = $options['limit'] ?? 10;
        $threshold = $options['threshold'] ?? 0.0;

        // Calculate similarity for each stored document
        $similarities = [];

        foreach ($this->documents as $id => $doc) {
            $similarity = $this->cosineSimilarity(
                $embedding,
                $doc['embedding']
            );

            if ($similarity >= $threshold) {
                $similarities[$id] = $similarity;
            }
        }

        // Sort by similarity descending
        arsort($similarities);

        // Return top N documents
        $results = [];
        foreach (array_slice($similarities, 0, $limit) as $id => $similarity) {
            $doc = $this->documents[$id];

            $results[] = new VectorDocument(
                id: $id,
                content: $doc['content'],
                embedding: $doc['embedding'],
                metadata: array_merge(
                    $doc['metadata'],
                    ['similarity' => $similarity]
                )
            );
        }

        return $results;
    }

    private function cosineSimilarity(array $a, array $b): float
    {
        $dotProduct = 0.0;
        $normA = 0.0;
        $normB = 0.0;

        for ($i = 0; $i < count($a); $i++) {
            $dotProduct += $a[$i] * $b[$i];
            $normA += $a[$i] ** 2;
            $normB += $b[$i] ** 2;
        }

        $normA = sqrt($normA);
        $normB = sqrt($normB);

        if ($normA == 0 || $normB == 0) {
            return 0.0;
        }

        return $dotProduct / ($normA * $normB);
    }
}
```

### Implementing ManagedStoreInterface

For databases and cloud services:

```php
use Symfony\AI\Store\Contract\ManagedStoreInterface;
use Symfony\AI\Store\Document\VectorDocument;

class DatabaseVectorStore implements ManagedStoreInterface
{
    public function __construct(
        private \PDO $pdo,
        private string $tableName = 'vectors',
    ) {}

    public function setup(): void
    {
        // Create table with vector support
        $sql = match (self::detectDatabase($this->pdo)) {
            'pgsql' => "
                CREATE TABLE IF NOT EXISTS {$this->tableName} (
                    id VARCHAR(255) PRIMARY KEY,
                    content TEXT NOT NULL,
                    embedding vector(1536) NOT NULL,
                    metadata JSONB DEFAULT '{}'::jsonb,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                );
                CREATE INDEX idx_embedding ON {$this->tableName} USING ivfflat (embedding vector_cosine_ops);
            ",
            'mysql' => "
                CREATE TABLE IF NOT EXISTS {$this->tableName} (
                    id VARCHAR(255) PRIMARY KEY,
                    content LONGTEXT NOT NULL,
                    embedding JSON NOT NULL,
                    metadata JSON DEFAULT '{}',
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    KEY idx_id (id)
                ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
            ",
        };

        foreach (explode(';', $sql) as $statement) {
            if (trim($statement)) {
                $this->pdo->exec($statement);
            }
        }
    }

    public function drop(): void
    {
        $this->pdo->exec("DROP TABLE IF EXISTS {$this->tableName}");
    }

    public function add(VectorDocument $document): string
    {
        $id = $document->getId() ?? uniqid('doc_');
        $embedding = json_encode($document->getEmbedding());
        $metadata = json_encode($document->getMetadata());

        $stmt = $this->pdo->prepare("
            INSERT INTO {$this->tableName}
            (id, content, embedding, metadata)
            VALUES (:id, :content, :embedding, :metadata)
            ON DUPLICATE KEY UPDATE
                content = VALUES(content),
                embedding = VALUES(embedding),
                metadata = VALUES(metadata)
        ");

        $stmt->execute([
            ':id' => $id,
            ':content' => $document->getContent(),
            ':embedding' => $embedding,
            ':metadata' => $metadata,
        ]);

        return $id;
    }

    public function query(array $embedding, array $options = []): array
    {
        $limit = $options['limit'] ?? 10;

        // Database-specific similarity query
        $sql = match (self::detectDatabase($this->pdo)) {
            'pgsql' => "
                SELECT
                    id, content, embedding, metadata,
                    1 - (embedding <=> :embedding::vector) as similarity
                FROM {$this->tableName}
                ORDER BY embedding <=> :embedding::vector
                LIMIT :limit
            ",
            'mysql' => "
                SELECT
                    id, content, embedding, metadata,
                    1 - JSON_EXTRACT(embedding, '$') as similarity
                FROM {$this->tableName}
                ORDER BY similarity DESC
                LIMIT :limit
            ",
        };

        $stmt = $this->pdo->prepare($sql);
        $stmt->execute([
            ':embedding' => json_encode($embedding),
            ':limit' => $limit,
        ]);

        $results = [];
        while ($row = $stmt->fetch(\PDO::FETCH_ASSOC)) {
            $results[] = new VectorDocument(
                id: $row['id'],
                content: $row['content'],
                embedding: json_decode($row['embedding'], true),
                metadata: array_merge(
                    json_decode($row['metadata'], true) ?? [],
                    ['similarity' => (float)$row['similarity']]
                )
            );
        }

        return $results;
    }

    private static function detectDatabase(\PDO $pdo): string
    {
        return $pdo->getAttribute(\PDO::ATTR_DRIVER_NAME);
    }
}
```

### Usage in Symfony

Register custom store in services:

```yaml
# config/services.yaml
services:
    App\Store\CustomVectorStore:
        arguments:
            $tableName: 'document_vectors'

    ai.store.custom:
        alias: App\Store\CustomVectorStore
```

Use in Indexer:

```php
$store = $container->get('ai.store.custom');
$indexer = new Indexer($platform, 'text-embedding-3-small', $store);
```

---

## Console Commands

### ai:store:setup

Initialize a managed store:

```bash
# Setup ChromaDB collection
php bin/console ai:store:setup chroma.documents

# Setup PostgreSQL with pgvector
php bin/console ai:store:setup postgres.articles
```

### ai:store:drop

Drop a store (DANGEROUS):

```bash
# Drop with confirmation
php bin/console ai:store:drop chroma.documents

# Force drop without confirmation
php bin/console ai:store:drop chroma.documents --force
```

### ai:store:index

Index documents from a source:

```bash
# Index from file
php bin/console ai:store:index default --source=/path/to/docs

# Index from database table
php bin/console ai:store:index default --source=articles --table=article

# Index with options
php bin/console ai:store:index default \
    --source=/docs \
    --extensions=md,txt \
    --batch-size=20 \
    --model=text-embedding-3-small
```

---

## Configuration Examples

### Basic Store Configuration

```yaml
# config/packages/ai.yaml
ai:
    vectorizer:
        model: 'text-embedding-3-small'
        dimension: 1536

    store:
        # In-memory for development
        default:
            class: 'Symfony\AI\Store\InMemory\InMemoryStore'

        # ChromaDB for testing
        chroma_documents:
            class: 'Symfony\AI\Store\Chroma\ChromaStore'
            arguments:
                host: '%env(CHROMA_HOST)%'
                port: '%env(int:CHROMA_PORT)%'
                collectionName: 'documents'

    indexer:
        default:
            store: 'ai.store.default'
            model: '%env(EMBEDDING_MODEL)%'

    retriever:
        default:
            store: 'ai.store.default'
            limit: 5
            threshold: 0.7
```

### PostgreSQL with pgvector

```yaml
ai:
    store:
        postgres_articles:
            class: 'Symfony\AI\Store\PostgreSQL\PostgreSQLStore'
            arguments:
                dsn: '%env(DATABASE_URL)%'
                tableName: 'articles_vectors'
                embeddingDimension: 1536

    indexer:
        articles:
            store: 'ai.store.postgres_articles'
            model: 'text-embedding-3-small'

    retriever:
        articles:
            store: 'ai.store.postgres_articles'
            limit: 10
            threshold: 0.6
```

First time setup:

```php
// src/Command/SetupArticleVectors.php
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\DependencyInjection\Attribute\Autowire;

#[AsCommand(name: 'app:vectors:setup')]
class SetupArticleVectorsCommand extends Command
{
    public function __construct(
        #[Autowire(service: 'ai.store.postgres_articles')]
        private StoreInterface $store,
    ) {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        if ($this->store instanceof ManagedStoreInterface) {
            $this->store->setup();
            $output->writeln('Article vectors table created.');
        }

        return Command::SUCCESS;
    }
}
```

### Multi-Tenant Configuration

```yaml
ai:
    store:
        # Separate collection per brand
        f24_documents:
            class: 'Symfony\AI\Store\Chroma\ChromaStore'
            arguments:
                collectionName: 'documents_f24'

        rfi_documents:
            class: 'Symfony\AI\Store\Chroma\ChromaStore'
            arguments:
                collectionName: 'documents_rfi'

        mcd_documents:
            class: 'Symfony\AI\Store\Chroma\ChromaStore'
            arguments:
                collectionName: 'documents_mcd'

    indexer:
        f24:
            store: 'ai.store.f24_documents'
            model: 'text-embedding-3-small'

        rfi:
            store: 'ai.store.rfi_documents'
            model: 'text-embedding-3-small'
```

Usage:

```php
#[Autowire(service: 'ai.store.%env(BRAND)%_documents')]
private StoreInterface $store,
```

### Hybrid Search Configuration

```yaml
ai:
    store:
        hybrid_articles:
            class: 'App\Store\HybridStore'
            arguments:
                vectorStore: '@ai.store.postgres_articles'
                fullTextIndex: '@app.search.meilisearch'

    indexer:
        hybrid_articles:
            store: 'ai.store.hybrid_articles'
            model: 'text-embedding-3-small'
```

---

## RAG Patterns

### Basic RAG Workflow

```php
use Symfony\AI\Store\Indexer;
use Symfony\AI\Store\Retriever;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

class DocumentAssistant
{
    public function __construct(
        private Agent $agent,
        private Retriever $retriever,
    ) {}

    public function answerQuestion(string $question): string
    {
        // Step 1: Retrieve relevant documents
        $documents = $this->retriever->retrieve($question, ['limit' => 5]);

        // Step 2: Build context from documents
        $context = "Based on the following documentation:\n\n";
        foreach ($documents as $doc) {
            $context .= "- " . $doc->getContent() . "\n";
        }

        // Step 3: Ask agent with context
        $messages = new MessageBag(
            Message::forSystem(
                'You are a documentation assistant. Answer based only on the provided context.'
            ),
            Message::ofUser($context . "\n\nQuestion: " . $question)
        );

        // Step 4: Return response
        return $this->agent->call($messages)->getContent();
    }
}
```

### Agent with SimilaritySearch Tool

```php
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Toolbox\AgentProcessor;

// Create tool for agent to use
$similaritySearch = new SimilaritySearch(
    vectorizer: $vectorizer,
    store: $store
);

// Add to toolbox
$toolbox = new Toolbox([$similaritySearch]);
$processor = new AgentProcessor($toolbox);

// Create agent with tool
$agent = new Agent(
    platform: $platform,
    modelName: 'gpt-4o-mini',
    inputProcessors: [$processor],
    outputProcessors: [$processor]
);

// Agent will autonomously use similarity_search tool
$messages = new MessageBag(
    Message::forSystem('Use similarity_search tool to find relevant documentation.'),
    Message::ofUser('How do I configure Symfony console commands?')
);

$result = $agent->call($messages);
echo $result->getContent();
```

### RAG with Citation

```php
class CitedDocumentAssistant
{
    public function answerWithCitations(string $question): array
    {
        $documents = $this->retriever->retrieve($question, ['limit' => 3]);

        // Build citations list
        $citations = [];
        $context = "Context:\n";

        foreach ($documents as $i => $doc) {
            $citations[$i] = [
                'title' => $doc->getMetadata()['title'] ?? 'Unknown',
                'source' => $doc->getMetadata()['source_url'] ?? null,
                'excerpt' => substr($doc->getContent(), 0, 200),
            ];
            $context .= "[$i] " . $doc->getContent() . "\n";
        }

        // Get response with citations
        $messages = new MessageBag(
            Message::forSystem('Provide answer with citations using [0], [1], etc.'),
            Message::ofUser($context . "\n\nQuestion: " . $question)
        );

        $answer = $this->agent->call($messages)->getContent();

        return [
            'answer' => $answer,
            'citations' => $citations,
        ];
    }
}
```

### Dynamic RAG with Metadata Filtering

```php
class FilteredDocumentAssistant
{
    public function answerForBrand(string $question, string $brand): string
    {
        // Retrieve only for specific brand
        $documents = $this->retriever->retrieve($question, [
            'limit' => 5,
            'filters' => [
                'brand' => $brand,
                'published' => ['$exists' => true],
            ]
        ]);

        // Build context
        $context = "Available documentation for $brand:\n";
        foreach ($documents as $doc) {
            $context .= "- " . $doc->getContent() . "\n";
        }

        $messages = new MessageBag(
            Message::forSystem("You are a $brand documentation assistant."),
            Message::ofUser($context . "\n\nQuestion: " . $question)
        );

        return $this->agent->call($messages)->getContent();
    }
}
```

---

## Best Practices

### Backend Selection Guide

**Use InMemory for:**
- Development and testing
- Small datasets (< 10K documents)
- Rapid prototyping

**Use PostgreSQL + pgvector for:**
- Existing PostgreSQL databases
- Data that's already in SQL
- Cost-conscious production
- Strong consistency needed
- Medium scale (100K-10M documents)

**Use Pinecone for:**
- SaaS convenience
- Managed infrastructure
- Large scale (millions of documents)
- Don't want to manage database
- Multi-region needed

**Use ChromaDB for:**
- Quick local development
- Learning vector databases
- Small production deployments
- Open-source requirement

**Use Meilisearch for:**
- Hybrid search (vector + full-text)
- Fast retrieval important
- Nice search UI/dashboard

**Use MongoDB for:**
- Existing MongoDB infrastructure
- Need document database + vector search
- Flexible schema important

### Performance Considerations

**Embedding Generation**
- Batch documents when possible (30-50% faster)
- Cache embeddings to avoid re-computing
- Use smaller models for development (faster, cheaper)

```php
// Batch is faster than individual
$indexer->indexBatch($documents, batchSize: 50); // Not individual loops
```

**Similarity Search**
- Add indexes on frequently filtered metadata
- Use reasonable limits (10-50, rarely more)
- Consider threshold filtering early

```php
// Fast: Limited results + threshold
$documents = $retriever->retrieve($query, [
    'limit' => 10,
    'threshold' => 0.7,
]);

// Slow: Retrieving thousands of results
$documents = $retriever->retrieve($query, ['limit' => 10000]);
```

**Storage Optimization**
- Delete unused documents regularly
- Archive old documents to separate store
- Use metadata for pruning strategies

```php
// Periodically clean up old documents
$deleteOldDocsCommand = "DELETE FROM vectors WHERE created_at < NOW() - INTERVAL '90 days'";
```

### Chunking Strategies

**Size-based chunking (simpler):**
- 300-500 token chunks
- Good for general knowledge

```php
// Roughly 75 words = 300 tokens
$chunks = str_split($text, 75 * 4); // 4 chars per token estimate
```

**Sentence-based chunking (better):**
- Complete sentences
- Natural boundaries
- Preserves meaning

```php
$sentences = preg_split('/[.!?]+/', $text);
$chunks = [];
$current = '';

foreach ($sentences as $sentence) {
    $tokens = str_word_count($sentence) * 1.3; // Estimate
    if ((str_word_count($current) * 1.3) + $tokens > 500) {
        if ($current) $chunks[] = $current;
        $current = $sentence;
    } else {
        $current .= ($current ? ' ' : '') . $sentence;
    }
}
```

**Semantic chunking (most sophisticated):**
- Split where meaning changes
- Use embedding similarity
- Best quality but slower

```php
class SemanticChunker
{
    public function __construct(private VectorizerInterface $vectorizer) {}

    public function chunk(string $text): array
    {
        $sentences = preg_split('/[.!?]+/', $text);
        $chunks = [];
        $current = [];
        $lastEmbedding = null;

        foreach ($sentences as $sentence) {
            if (!$sentence = trim($sentence)) continue;

            $embedding = $this->vectorizer->embed($sentence);

            // If embedding differs significantly, start new chunk
            if ($lastEmbedding && !$this->isSimilar($embedding, $lastEmbedding)) {
                $chunks[] = implode('. ', $current) . '.';
                $current = [];
            }

            $current[] = $sentence;
            $lastEmbedding = $embedding;
        }

        if ($current) {
            $chunks[] = implode('. ', $current) . '.';
        }

        return $chunks;
    }

    private function isSimilar(array $a, array $b, float $threshold = 0.7): bool
    {
        // Cosine similarity
        return $this->cosineSimilarity($a, $b) > $threshold;
    }
}
```

### Metadata Usage

**Effective metadata patterns:**

```php
$document = new TextDocument(
    $content,
    metadata: [
        // Essential for filtering
        'brand' => 'f24',
        'type' => 'article',
        'language' => 'en',

        // For ranking/sorting
        'published_at' => $article->getPublishedAt()->getTimestamp(),
        'importance' => $article->getImportance(),
        'author_id' => $article->getAuthorId(),

        // For deduplication
        'source_hash' => md5($content),
        'source_url' => $article->getUrl(),

        // For reconstruction
        'article_id' => $article->getId(),
        'section_index' => $index,
        'chunk_number' => $chunkNumber,
    ]
);
```

### Production Setup Checklist

- [ ] Use managed store (PostgreSQL, Pinecone, etc.), not InMemory
- [ ] Configure embedding model API keys in .env
- [ ] Set up scheduled jobs to keep indexes current
- [ ] Implement monitoring for retrieval quality
- [ ] Add rate limiting for API calls
- [ ] Use batch processing for large document sets
- [ ] Set up backup for critical vector data
- [ ] Monitor costs (embeddings are per-token)
- [ ] Implement cache for frequent queries
- [ ] Document document format expectations
- [ ] Set up alerts for store failures
- [ ] Test failover to secondary store

### Monitoring and Maintenance

```php
class VectorStoreMonitor
{
    public function __construct(
        private StoreInterface $store,
    ) {}

    public function getHealth(): array
    {
        return [
            'store_type' => get_class($this->store),
            'is_managed' => $this->store instanceof ManagedStoreInterface,
            'test_document' => $this->testIndexRetrieval(),
            'approximate_size' => $this->estimateSize(),
        ];
    }

    private function testIndexRetrieval(): bool
    {
        try {
            $vectorizer = new OpenAIVectorizer($_ENV['OPENAI_API_KEY']);
            $testEmbedding = $vectorizer->embed('test');

            $results = $this->store->query($testEmbedding, ['limit' => 1]);
            return true;
        } catch (\Throwable $e) {
            return false;
        }
    }

    private function estimateSize(): ?int
    {
        // Backend specific
        return null;
    }
}
```

---

## Common Issues and Solutions

**Issue: Retrieval returns irrelevant documents**
- Solution: Increase threshold, adjust chunking strategy, improve document metadata
- Check embedding model is appropriate for domain

**Issue: Slow embedding generation**
- Solution: Use batch indexing, smaller embedding model, cache results
- Consider using GPU-accelerated vectorizer

**Issue: High cost per query**
- Solution: Cache query results, batch queries, use cheaper embedding model
- Consider reranking top-k results with a simpler metric

**Issue: Store not found errors in production**
- Solution: Initialize managed stores in deployment script
- Use health check commands before indexing

**Issue: Memory usage increasing**
- Solution: Don't use InMemory store in production
- Implement periodic cleanup of old documents
- Monitor embedding cache size
