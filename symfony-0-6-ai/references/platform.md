# Symfony AI Platform - Comprehensive Reference

## Table of Contents

1. [Platform Interface & Factory Pattern](#platform-interface--factory-pattern)
2. [Supported Providers](#supported-providers)
3. [Models](#models)
4. [Messages](#messages)
5. [Message Templates](#message-templates)
6. [Options](#options)
7. [Result Types](#result-types)
8. [Streaming](#streaming)
9. [Image Processing](#image-processing)
10. [Audio Processing](#audio-processing)
11. [Embeddings](#embeddings)
12. [Structured Output](#structured-output)
13. [Advanced Features](#advanced-features)
14. [Testing](#testing)
15. [Best Practices](#best-practices)

---

## Platform Interface & Factory Pattern

The Symfony AI Platform uses a provider-agnostic interface system that abstracts the complexity of working with different AI providers while maintaining a consistent API.

### PlatformInterface

The core interface that all platform implementations must follow:

```php
namespace Symfony\Component\Ai;

interface PlatformInterface
{
    /**
     * Request text completion from the AI platform
     */
    public function complete(
        ?Request $request = null,
        Options $options = new Options(),
    ): Response;

    /**
     * Stream text completion responses
     */
    public function completeStream(
        ?Request $request = null,
        Options $options = new Options(),
    ): ResponseStream;

    /**
     * Generate embeddings for text
     */
    public function embed(
        EmbedRequest $request,
        Options $options = new Options(),
    ): EmbedResponse;

    /**
     * Get the default model for this platform
     */
    public function getDefaultModel(): Model;

    /**
     * Get all available models for this platform
     */
    public function getModels(): Models;

    /**
     * Get the platform name (e.g., 'openai', 'anthropic')
     */
    public function getName(): string;
}
```

### PlatformFactory Pattern

Services configure platforms using the factory pattern with provider-specific credentials:

```php
use Symfony\Component\Ai\Provider\OpenAiPlatformFactory;
use Symfony\Component\Ai\Provider\AnthropicPlatformFactory;
use Symfony\Component\Ai\Provider\GooglePlatformFactory;

// OpenAI (GPT-4, GPT-3.5-turbo)
$openaiPlatform = OpenAiPlatformFactory::create(
    apiKey: $_ENV['OPENAI_API_KEY'],
    organizationId: $_ENV['OPENAI_ORG_ID'] ?? null,
);

// Anthropic (Claude 3.5 Sonnet, Opus)
$anthropicPlatform = AnthropicPlatformFactory::create(
    apiKey: $_ENV['ANTHROPIC_API_KEY'],
);

// Google (Gemini Pro, Gemini Flash)
$googlePlatform = GooglePlatformFactory::create(
    apiKey: $_ENV['GOOGLE_API_KEY'],
    projectId: $_ENV['GOOGLE_PROJECT_ID'] ?? null,
);

// Ollama (local LLMs)
$ollamaPlatform = OllamaPlatformFactory::create(
    baseUrl: 'http://ollama:11434',
);
```

### Service Configuration

Register platforms as Symfony services:

```yaml
# config/packages/ai.yaml
services:
  # Single default platform
  Symfony\Component\Ai\PlatformInterface:
    factory: [Symfony\Component\Ai\Provider\OpenAiPlatformFactory, create]
    arguments:
      $apiKey: '%env(OPENAI_API_KEY)%'

  # Multiple platforms by name
  openai_platform:
    factory: [Symfony\Component\Ai\Provider\OpenAiPlatformFactory, create]
    arguments:
      $apiKey: '%env(OPENAI_API_KEY)%'

  anthropic_platform:
    factory: [Symfony\Component\Ai\Provider\AnthropicPlatformFactory, create]
    arguments:
      $apiKey: '%env(ANTHROPIC_API_KEY)%'

  google_platform:
    factory: [Symfony\Component\Ai\Provider\GooglePlatformFactory, create]
    arguments:
      $apiKey: '%env(GOOGLE_API_KEY)%'
```

### Dependency Injection with Named Arguments

```php
namespace App\Service;

use Symfony\Component\Ai\PlatformInterface;

class ContentAnalyzer
{
    public function __construct(
        private PlatformInterface $platform,
    ) {}

    public function analyze(string $content): array
    {
        $response = $this->platform->complete(
            new Request('Analyze this: ' . $content),
        );

        return [
            'analysis' => $response->getContent(),
            'model' => $response->getModel(),
            'usage' => $response->getTokenUsage(),
        ];
    }
}
```

---

## Supported Providers

The Symfony AI component supports 40+ AI providers with unified configuration. Each provider factory handles authentication, API endpoints, and provider-specific options.

### Providers Table

| Provider | Factory | Models | Auth | Special Features |
|----------|---------|--------|------|------------------|
| OpenAI | `OpenAiPlatformFactory` | GPT-4, GPT-4 Turbo, GPT-3.5-turbo, DALL-E 3 | API Key | Vision, function calling |
| Anthropic | `AnthropicPlatformFactory` | Claude 3 Opus, Sonnet, Haiku, Claude 2 | API Key | Vision, extended thinking |
| Google | `GooglePlatformFactory` | Gemini Pro, Gemini Pro Vision, PaLM 2 | API Key + Project | Vertex AI integration |
| Azure OpenAI | `AzureOpenAiPlatformFactory` | All OpenAI models | API Key + Endpoint | Enterprise integration |
| Cohere | `CoherePlatformFactory` | Command, Command Light | API Key | Multilingual, embeddings |
| Mistral | `MistralPlatformFactory` | Mistral 7B, Mistral Medium | API Key | Open source models |
| Hugging Face | `HuggingFacePlatformFactory` | 100k+ models | API Key + Model ID | Model hub access |
| Replicate | `ReplicatePlatformFactory` | 10k+ models | API Key | Model marketplace |
| Together AI | `TogetherAiPlatformFactory` | Open source models | API Key | Fast inference |
| Fireworks | `FireworksPlatformFactory` | Open source models | API Key | Optimized for speed |
| Groq | `GroqPlatformFactory` | LLaMA 2, Mixtral | API Key | Ultra-fast LLMs |
| Perplexity | `PerpelexityPlatformFactory` | pplx-7b-online, pplx-70b | API Key | Real-time web search |
| Aleph Alpha | `AlephAlphaPlatformFactory` | Luminous models | API Key | European privacy |
| NLP Cloud | `NlpCloudPlatformFactory` | GPT-J, T5, Falcon | API Key | Affordable models |
| Baseten | `BaseteenPlatformFactory` | Custom models | API Key | MLOps platform |
| Modal | `ModalPlatformFactory` | Custom models | API Token | Serverless platform |
| AWS Bedrock | `AwsBedrockPlatformFactory` | Claude, Titan, Falcon | AWS Credentials | AWS integration |
| Azure | `AzurePlatformFactory` | Llama 2, Falcon, Databricks | Azure credentials | Azure integration |
| LocalAI | `LocalAiPlatformFactory` | 100+ local models | API Key | On-premises |
| Ollama | `OllamaPlatformFactory` | LLaMA, Mistral, Neural Chat | Base URL | Local development |
| LM Studio | `LmStudioPlatformFactory` | Quantized models | Base URL | Desktop development |
| Vllm | `VllmPlatformFactory` | All HF models | Base URL | High-throughput |

### Provider-Specific Configuration Examples

```php
// Azure OpenAI
$azurePlatform = AzureOpenAiPlatformFactory::create(
    apiKey: $_ENV['AZURE_OPENAI_API_KEY'],
    endpoint: 'https://{instance-name}.openai.azure.com/',
    deploymentId: 'gpt-4',
);

// AWS Bedrock
$bedrockPlatform = AwsBedrockPlatformFactory::create(
    region: 'us-east-1',
    accessKeyId: $_ENV['AWS_ACCESS_KEY_ID'],
    secretAccessKey: $_ENV['AWS_SECRET_ACCESS_KEY'],
);

// Hugging Face with custom model
$hfPlatform = HuggingFacePlatformFactory::create(
    apiKey: $_ENV['HUGGINGFACE_API_KEY'],
    modelId: 'meta-llama/Llama-2-70b-chat-hf',
);

// Local Ollama setup
$ollamaPlatform = OllamaPlatformFactory::create(
    baseUrl: 'http://localhost:11434',
);

// LocalAI with custom endpoint
$localaiPlatform = LocalAiPlatformFactory::create(
    baseUrl: 'http://localai:8080',
    apiKey: 'sk-local-ai-key',
);
```

---

## Models

The Model class represents an AI model with its capabilities, configuration, and features.

### Model Class Structure

```php
namespace Symfony\Component\Ai;

final class Model
{
    /**
     * Get the model identifier (e.g., 'gpt-4-turbo-preview')
     */
    public function getId(): string;

    /**
     * Get human-readable model name
     */
    public function getName(): string;

    /**
     * Get the provider name
     */
    public function getProvider(): string;

    /**
     * Get model capabilities (text, vision, audio, etc.)
     */
    public function getCapabilities(): Capabilities;

    /**
     * Check if model supports a specific capability
     */
    public function hasCapability(string $capability): bool;

    /**
     * Get model metadata (release date, context window, etc.)
     */
    public function getMetadata(): array;

    /**
     * Check if this is the default model
     */
    public function isDefault(): bool;

    /**
     * Get model context window size
     */
    public function getContextWindow(): int;

    /**
     * Get maximum output tokens
     */
    public function getMaxOutputTokens(): int;
}

// Capabilities enumeration
enum Capabilities: string
{
    case TEXT = 'text';
    case VISION = 'vision';
    case AUDIO = 'audio';
    case VIDEO = 'video';
    case FUNCTION_CALLING = 'function_calling';
    case VISION_URL = 'vision_url';
    case JSON_MODE = 'json_mode';
    case THINKING = 'thinking';
}
```

### Working with Models

```php
// Get default model
$model = $platform->getDefaultModel();
echo $model->getId();  // 'gpt-4-turbo-preview'

// Get all available models
$models = $platform->getModels();
foreach ($models as $model) {
    if ($model->hasCapability(Capabilities::VISION)) {
        echo "Vision model: " . $model->getName();
    }
}

// Filter models by capability
$visionModels = $models->filter(function (Model $model) {
    return $model->hasCapability(Capabilities::VISION);
});

// Select specific model
$response = $platform->complete(
    request: new Request('What is this image?'),
    options: (new Options())
        ->withModel('gpt-4-vision-preview'),
);
```

### Model Size Variants

Different models support size variants for cost/performance trade-offs:

```php
// OpenAI model variants
'gpt-4-turbo-preview'          // Full capability, high cost
'gpt-4'                         // Older but stable
'gpt-3.5-turbo'                 // Fastest, lowest cost
'gpt-3.5-turbo-16k'             // Extended context window

// Anthropic variants
'claude-3-opus-20240229'        // Most capable
'claude-3-sonnet-20240229'      // Balanced
'claude-3-haiku-20240307'       // Fastest, cheapest

// Google Gemini variants
'gemini-pro'                     // Full model
'gemini-pro-vision'              // Vision capable
'gemini-1.5-pro'                 // Latest generation
'gemini-1.5-flash'               // Lightweight

// Mistral variants
'mistral-large-latest'           // Full capability
'mistral-medium-latest'          // Balanced
'mistral-small-latest'           // Fast, cheap
```

### Custom Models

Some platforms allow registering custom models:

```php
$customModel = new Model(
    id: 'my-custom-gpt4',
    name: 'My Custom GPT-4',
    provider: 'openai',
    capabilities: [Capabilities::TEXT, Capabilities::VISION],
    contextWindow: 8192,
    maxOutputTokens: 2048,
);

$options = (new Options())
    ->withModel($customModel);
```

### Ollama API Catalog

For Ollama, models are dynamically discovered from the local catalog:

```php
$ollamaPlatform = OllamaPlatformFactory::create(
    baseUrl: 'http://ollama:11434',
);

// List all available models
$models = $ollamaPlatform->getModels();

// Models are loaded from Ollama's library at runtime
// Common Ollama models:
// - llama2
// - mistral
// - neural-chat
// - starling-lm
// - dolphin-mixtral
// - orca-mini
```

---

## Messages

Messages form the building blocks of conversations with AI platforms. The system supports multiple message types with flexible content handling.

### Message Types

```php
namespace Symfony\Component\Ai;

// System message - instructions for the model
$systemMessage = Message::system('You are a helpful assistant.');

// User message - input from the user
$userMessage = Message::user('What is quantum physics?');

// Assistant message - previous model responses
$assistantMessage = Message::assistant('Quantum physics is...');

// Tool message - feedback from tool execution
$toolMessage = Message::tool(
    toolName: 'calculator',
    result: '42',
);
```

### Message Class

```php
final class Message
{
    /**
     * Get the message role
     */
    public function getRole(): MessageRole;

    /**
     * Get message content (text, images, audio, mixed)
     */
    public function getContent(): Content;

    /**
     * Get message metadata (ID, timestamp, etc.)
     */
    public function getMetadata(): array;

    /**
     * Get message ID (UUID v7)
     */
    public function getId(): string;

    /**
     * Get creation timestamp
     */
    public function getCreatedAt(): DateTimeImmutable;

    /**
     * Convert to array for serialization
     */
    public function toArray(): array;

    // Factory methods
    public static function system(Content|string $content): self;
    public static function user(Content|string $content): self;
    public static function assistant(Content|string $content): self;
    public static function tool(string $toolName, string $result): self;
}
```

### MessageBag - Conversation Management

```php
use Symfony\Component\Ai\MessageBag;
use Symfony\Component\Ai\Message;

// Create conversation history
$messages = new MessageBag([
    Message::system('You are a code reviewer.'),
    Message::user('Review this PHP function...'),
    Message::assistant('This function has several issues...'),
    Message::user('How can I improve it?'),
]);

// Add messages to conversation
$messages->add(Message::assistant('You could use type hints...'));

// Get all messages
foreach ($messages as $message) {
    echo $message->getRole() . ': ' . $message->getContent();
}

// Filter messages by role
$userMessages = $messages->filter(function (Message $msg) {
    return $msg->getRole() === MessageRole::USER;
});

// Send entire conversation
$response = $platform->complete(
    request: new Request(messages: $messages),
);
```

### Message Content Types

```php
// Text content (most common)
$message = Message::user('Hello, world!');

// Image content
use Symfony\Component\Ai\Content\Image;

$message = Message::user(
    new Image(
        data: file_get_contents('/path/to/image.jpg'),
        mimeType: 'image/jpeg',
    )
);

// Audio content
use Symfony\Component\Ai\Content\Audio;

$message = Message::user(
    new Audio(
        data: file_get_contents('/path/to/audio.mp3'),
        mimeType: 'audio/mpeg',
    )
);

// Image URL content
use Symfony\Component\Ai\Content\ImageUrl;

$message = Message::user(
    new ImageUrl(
        url: 'https://example.com/image.jpg',
    )
);

// Mixed content (image + text)
use Symfony\Component\Ai\Content\Text;

$message = Message::user([
    new Text('What is in this image?'),
    new Image(data: $imageData, mimeType: 'image/jpeg'),
]);
```

### Message IDs (UUID v7)

Messages are automatically assigned unique identifiers using UUID v7:

```php
$message = Message::user('Tell me a joke');

// Get auto-generated UUID v7
$id = $message->getId();  // e.g., '01ARZ3NDEKTSV4RRFFQ69G5FAV'

// Create message with custom metadata
$message = new Message(
    role: MessageRole::USER,
    content: new Text('Question'),
    metadata: [
        'correlation_id' => 'req-12345',
        'tenant_id' => 'customer-1',
        'timestamp' => (new DateTimeImmutable())->getTimestamp(),
    ],
);

// Access metadata
$correlationId = $message->getMetadata()['correlation_id'] ?? null;
```

---

## Message Templates

Message templates provide flexible ways to construct messages with variable substitution and expression evaluation.

### String Templates

Basic string templates with simple variable substitution:

```php
use Symfony\Component\Ai\Template\StringTemplate;

$template = new StringTemplate(
    'Summarize this text for a {audience}: {text}'
);

$message = $template->render([
    'audience' => 'grade 5 student',
    'text' => 'Complex technical article...',
]);

echo $message;
// Output: "Summarize this text for a grade 5 student: Complex technical article..."
```

### Expression Templates

Advanced templates using Symfony Expression Language:

```php
use Symfony\Component\Ai\Template\ExpressionTemplate;

$template = new ExpressionTemplate(
    'Sentiment Analysis\n' .
    'Text: {{ text }}\n' .
    'Expected format: ' .
    '{{ response_format|upper }}\n' .
    'Priority: {{ priority > 5 ? "High" : "Normal" }}'
);

$message = $template->render([
    'text' => 'This product is amazing!',
    'response_format' => 'json',
    'priority' => 8,
]);
```

### Template Variables

Common template variables used across platforms:

```php
// Custom variables
$variables = [
    'text' => $userInput,
    'language' => 'French',
    'tone' => 'formal',
    'max_words' => 100,
    'format' => 'markdown',
];

// Built-in template variables (available in some platforms)
[
    'date' => new DateTime(),           // Current date/time
    'model' => 'gpt-4-turbo-preview',   // Selected model
    'version' => 'v1.0',                // Template version
    'user_id' => '12345',               // Current user
]
```

### TemplateRenderer Service

Register and use template renderer in services:

```php
namespace App\Service;

use Symfony\Component\Ai\Template\TemplateRenderer;
use Symfony\Component\Ai\Template\ExpressionTemplate;

class ReportGenerator
{
    public function __construct(
        private TemplateRenderer $renderer,
    ) {}

    public function generatePrompt(array $data): string
    {
        $template = new ExpressionTemplate(
            'Generate a {{ type }} report for {{ company }}. ' .
            'Focus on: {{ focus|join(", ") }}. ' .
            'Maximum length: {{ max_words }} words.'
        );

        return $this->renderer->render($template, $data);
    }
}
```

---

## Options

Options control model behavior, output format, and provider-specific settings.

### Core Options

```php
use Symfony\Component\Ai\Options;

$options = (new Options())
    // Model selection
    ->withModel('gpt-4-turbo-preview')

    // Temperature: 0-2 (higher = more creative/random)
    ->withTemperature(0.7)

    // Top P: 0-1 (nucleus sampling)
    ->withTopP(0.9)

    // Top K: minimum 1 (top-k sampling)
    ->withTopK(40)

    // Maximum output length
    ->withMaxOutputTokens(2048)

    // Response format
    ->withResponseFormat('json')

    // Timeout in seconds
    ->withTimeout(60)

    // Custom headers
    ->withHeader('X-Custom-Header', 'value');
```

### Temperature Control

```php
// Deterministic (for consistent results)
$options = (new Options())->withTemperature(0.0);
// Good for: data extraction, code generation, calculations

// Balanced (default)
$options = (new Options())->withTemperature(0.7);
// Good for: general-purpose tasks

// Creative (for variety)
$options = (new Options())->withTemperature(1.5);
// Good for: brainstorming, creative writing, story generation

// Ultra-creative (maximum variance)
$options = (new Options())->withTemperature(2.0);
// Good for: extreme diversity (may produce lower quality)
```

### Model-Specific Options

```php
// OpenAI specific
$options = (new Options())
    ->withModelOption('logit_bias', ['50256' => -100])  // Don't use token
    ->withModelOption('user', 'user-123')               // Track user for abuse
    ->withModelOption('seed', 42)                       // Reproducible outputs
    ->withModelOption('presence_penalty', 0.6)          // Penalize repetition
    ->withModelOption('frequency_penalty', 0.5);        // Reduce frequency

// Anthropic specific
$options = (new Options())
    ->withModelOption('system', 'You are a helpful assistant.')
    ->withModelOption('thinking', [
        'type' => 'enabled',
        'budget_tokens' => 10000,
    ]);

// Google Gemini specific
$options = (new Options())
    ->withModelOption('candidate_count', 1)
    ->withModelOption('stop_sequences', ['\n\n'])
    ->withModelOption('safety_settings', [
        [
            'category' => 'HARM_CATEGORY_SEXUALLY_EXPLICIT',
            'threshold' => 'BLOCK_NONE',
        ],
    ]);
```

### Options Array Format

```php
// Chainable fluent API
$options = (new Options())
    ->withTemperature(0.5)
    ->withMaxOutputTokens(1000)
    ->withModel('gpt-4');

// Or pass all at once
$options = new Options([
    'temperature' => 0.5,
    'max_output_tokens' => 1000,
    'model' => 'gpt-4',
]);

// Access options
$temp = $options->getTemperature();
$model = $options->getModel();
$array = $options->toArray();
```

---

## Result Types

AI platforms return different result types depending on the request. All results include metadata and token usage information.

### Response & TextResult

```php
namespace Symfony\Component\Ai;

class Response
{
    /**
     * Get the response content
     */
    public function getContent(): string;

    /**
     * Get the model used
     */
    public function getModel(): string;

    /**
     * Get token usage information
     */
    public function getTokenUsage(): TokenUsage;

    /**
     * Get response metadata (finish reason, etc.)
     */
    public function getMetadata(): array;

    /**
     * Check if response was complete
     */
    public function isComplete(): bool;
}

// Token usage breakdown
class TokenUsage
{
    public function getPromptTokens(): int;
    public function getCompletionTokens(): int;
    public function getTotalTokens(): int;
    public function getCacheReadTokens(): int;      // Prompt caching (OpenAI)
    public function getCacheCreationTokens(): int;  // Prompt caching (OpenAI)
}
```

### Using Response

```php
$response = $platform->complete(
    new Request('What is machine learning?'),
);

// Get the response
echo $response->getContent();

// Check token usage
$usage = $response->getTokenUsage();
echo "Prompt: {$usage->getPromptTokens()} tokens\n";
echo "Completion: {$usage->getCompletionTokens()} tokens\n";
echo "Total: {$usage->getTotalTokens()} tokens\n";

// Check if response completed normally
if ($response->isComplete()) {
    echo "Response completed successfully";
} else {
    echo "Response was truncated";
}

// Get metadata
$metadata = $response->getMetadata();
echo "Finish reason: {$metadata['finish_reason']}";  // 'stop', 'length', 'tool_calls', etc.
```

### VectorResult - Embeddings

```php
class EmbedResponse
{
    /**
     * Get embedding vectors
     */
    public function getVectors(): array;

    /**
     * Get embedding metadata
     */
    public function getMetadata(): array;

    /**
     * Get token usage
     */
    public function getTokenUsage(): TokenUsage;
}

// Generate embeddings
$response = $platform->embed(
    new EmbedRequest(['Hello world', 'Goodbye world']),
);

$vectors = $response->getVectors();
foreach ($vectors as $index => $vector) {
    echo "Vector $index: " . count($vector) . " dimensions\n";

    // Vectors are typically 1536 dimensions (OpenAI) or 768 (Hugging Face)
    // Can be used for similarity search, clustering, etc.
}
```

### BinaryResult - Raw Data

```php
class BinaryResult
{
    /**
     * Get raw binary data
     */
    public function getData(): string;

    /**
     * Get MIME type
     */
    public function getMimeType(): string;

    /**
     * Get result metadata
     */
    public function getMetadata(): array;

    /**
     * Save to file
     */
    public function saveToFile(string $path): void;
}

// Generate image with DALL-E
$response = $platform->complete(
    new Request('Generate an image of a cat'),
    new Options()->withModel('dall-e-3'),
);

// For binary results like images
$image = $response->getData();
file_put_contents('/tmp/cat.png', $image);
```

### Metadata in Results

```php
// Response includes rich metadata
$metadata = $response->getMetadata();

// Common metadata fields
[
    'finish_reason' => 'stop',              // How completion ended
    'model_id' => 'gpt-4-turbo-preview',   // Exact model used
    'created_at' => 1234567890,             // Unix timestamp
    'id' => 'chatcmpl-123abc',              // Provider's response ID
    'usage' => [
        'cache_read' => 100,                // Cached tokens (if applicable)
        'cache_creation' => 50,             // Tokens for cache creation
    ],
    'system_fingerprint' => 'fp_xyz',       // For reproducibility
]
```

---

## Streaming

Streaming provides real-time responses for text generation, enabling interactive applications.

### SSE (Server-Sent Events)

Streaming responses use the Server-Sent Events protocol:

```php
// Get streaming response
$stream = $platform->completeStream(
    new Request('Write a poem about coding'),
);

// Iterate through stream
foreach ($stream as $chunk) {
    echo $chunk->getContent();  // Print incrementally
    flush();                     // Send to client immediately
}

// Access stream metadata after completion
$finalMetadata = $stream->getMetadata();
echo "Total tokens: {$finalMetadata['total_tokens']}\n";
```

### Stream Configuration

```php
$options = (new Options())
    ->withModel('gpt-4-turbo-preview')
    ->withStreamingChunkSize(20)        // Bytes per chunk
    ->withStreamingTimeout(300);         // Max stream duration

$stream = $platform->completeStream(
    new Request('Start streaming...'),
    $options,
);

// Collect all chunks
$fullResponse = '';
foreach ($stream as $chunk) {
    $fullResponse .= $chunk->getContent();
}

echo $fullResponse;
```

### Server-Sent Events in Web Responses

```php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Ai\PlatformInterface;
use Symfony\Component\HttpFoundation\StreamedResponse;

class AiController extends AbstractController
{
    public function __construct(
        private PlatformInterface $platform,
    ) {}

    public function stream(): StreamedResponse
    {
        return new StreamedResponse(function () {
            $stream = $this->platform->completeStream(
                new Request($_POST['prompt'] ?? 'Say hello'),
            );

            foreach ($stream as $chunk) {
                echo "data: " . json_encode([
                    'content' => $chunk->getContent(),
                ]) . "\n\n";

                flush();
            }

            echo "data: [DONE]\n\n";
        });
    }
}
```

### Client-Side Streaming

```javascript
// Connect to streaming endpoint
const response = await fetch('/ai/stream', {
    method: 'POST',
    body: JSON.stringify({ prompt: 'Write code...' }),
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const event = decoder.decode(value);
    if (event.startsWith('data: ')) {
        const data = JSON.parse(event.slice(6));
        if (data.content) {
            document.getElementById('response').textContent += data.content;
        }
    }
}
```

---

## Image Processing

The platform supports image input in multiple formats for vision-capable models.

### Image Input Formats

```php
use Symfony\Component\Ai\Content\Image;
use Symfony\Component\Ai\Content\ImageUrl;

// Local file
$image = new Image(
    data: file_get_contents('/path/to/image.jpg'),
    mimeType: 'image/jpeg',
);

// Base64 encoded data URL
$image = new Image(
    data: 'iVBORw0KGgoAAAANSUhEUgAAAAUA...',  // base64
    mimeType: 'image/png',
    isBase64: true,
);

// Remote URL
$image = new ImageUrl(
    url: 'https://example.com/image.jpg',
);

// Data URL (base64 embedded)
$image = new ImageUrl(
    url: 'data:image/jpeg;base64,/9j/4AAQSkZJRg...',
);
```

### Multi-Image Support

```php
use Symfony\Component\Ai\Content\Text;
use Symfony\Component\Ai\Content\Image;

// Multiple images in single message
$message = Message::user([
    new Text('Compare these images:'),
    new Image(data: $image1Data, mimeType: 'image/jpeg'),
    new Image(data: $image2Data, mimeType: 'image/jpeg'),
    new Text('Which is better and why?'),
]);

// Multi-image conversation
$messages = new MessageBag([
    Message::system('You are an image analyst.'),
    Message::user([
        new Text('Analyze this image:'),
        new ImageUrl(url: 'https://example.com/image1.jpg'),
    ]),
    Message::assistant('This image shows...'),
    Message::user([
        new Text('Now compare with this second image:'),
        new ImageUrl(url: 'https://example.com/image2.jpg'),
    ]),
]);

$response = $platform->complete(
    new Request(messages: $messages),
);
```

### Image Analysis Examples

```php
// Object detection
$response = $platform->complete(
    new Request([
        new Text('List all objects you see in this image'),
        new ImageUrl(url: 'https://example.com/photo.jpg'),
    ]),
);

// OCR (Optical Character Recognition)
$response = $platform->complete(
    new Request([
        new Text('Extract all text from this image'),
        new Image(data: $documentImage, mimeType: 'image/png'),
    ]),
);

// Image annotation with JSON
$response = $platform->complete(
    new Request([
        new Text('Describe this image in JSON format'),
        new ImageUrl(url: 'https://example.com/scene.jpg'),
    ]),
    new Options()->withResponseFormat('json'),
);
```

---

## Audio Processing

The platform supports audio input for speech recognition and audio analysis.

### Audio Input

```php
use Symfony\Component\Ai\Content\Audio;

// Audio file
$audio = new Audio(
    data: file_get_contents('/path/to/audio.mp3'),
    mimeType: 'audio/mpeg',
);

// WAV format
$audio = new Audio(
    data: file_get_contents('/path/to/audio.wav'),
    mimeType: 'audio/wav',
);

// Ogg Vorbis
$audio = new Audio(
    data: file_get_contents('/path/to/audio.ogg'),
    mimeType: 'audio/ogg',
);
```

### Audio Message Creation

```php
// Transcription request
$message = Message::user([
    new Text('Transcribe this audio:'),
    new Audio(data: $audioData, mimeType: 'audio/mpeg'),
]);

$response = $platform->complete(
    new Request($message),
);

// Multimodal: Audio + image + text
$message = Message::user([
    new Text('Listen to the audio and analyze the image together'),
    new Audio(data: $audioData, mimeType: 'audio/mpeg'),
    new Image(data: $imageData, mimeType: 'image/jpeg'),
]);
```

### Supported Audio Models

```php
// Models supporting audio (varies by provider)
// OpenAI: Whisper (transcription), GPT-4 Vision (with vision)
// Google: Gemini with audio (coming soon)
// Anthropic: Claude with audio (experimental)

$response = $platform->complete(
    new Request($audioMessage),
    new Options()->withModel('gpt-4-turbo-preview'),
);

// Check model capabilities
$model = $platform->getDefaultModel();
if ($model->hasCapability(Capabilities::AUDIO)) {
    echo "Audio supported";
}
```

---

## Embeddings

Generate vector embeddings for text, enabling semantic search and similarity calculations.

### Text Embeddings

```php
use Symfony\Component\Ai\EmbedRequest;

// Single text embedding
$response = $platform->embed(
    new EmbedRequest('machine learning algorithms'),
);

$vectors = $response->getVectors();
$embedding = $vectors[0];  // Single vector

// Multiple embeddings
$response = $platform->embed(
    new EmbedRequest([
        'machine learning',
        'deep learning',
        'neural networks',
    ]),
);

// Iterate through embeddings
foreach ($response->getVectors() as $index => $vector) {
    echo "Text $index embedding dimension: " . count($vector);
}
```

### Embedding Models & Dimensions

```php
// Common embedding models and their dimensions
'text-embedding-3-small'   => 1536,  // OpenAI (smaller, faster)
'text-embedding-3-large'   => 3072,  // OpenAI (more expressive)
'text-embedding-ada-002'   => 1536,  // OpenAI (legacy)

'embedding-001'            => 768,   // Google PaLM
'multilingual-e5-large'    => 1024,  // HF (multilingual)
'bge-large-en-v1.5'        => 1024,  // HF (high quality)

$options = (new Options())
    ->withModel('text-embedding-3-large');

$response = $platform->embed(
    new EmbedRequest('Your text'),
    $options,
);
```

### Semantic Search Implementation

```php
class SemanticSearch
{
    public function __construct(
        private PlatformInterface $platform,
    ) {}

    /**
     * Find similar documents using embeddings
     */
    public function search(string $query, array $documents): array
    {
        // Embed query
        $queryResponse = $this->platform->embed(
            new EmbedRequest($query),
        );
        $queryVector = $queryResponse->getVectors()[0];

        // Embed all documents
        $docResponse = $this->platform->embed(
            new EmbedRequest($documents),
        );

        // Calculate cosine similarity
        $similarities = [];
        foreach ($docResponse->getVectors() as $idx => $docVector) {
            $similarities[$idx] = $this->cosineSimilarity(
                $queryVector,
                $docVector,
            );
        }

        // Sort by similarity
        arsort($similarities);

        return array_map(
            fn ($idx) => $documents[$idx],
            array_keys($similarities),
        );
    }

    private function cosineSimilarity(array $a, array $b): float
    {
        $dotProduct = array_sum(array_map(
            fn ($x, $y) => $x * $y,
            $a,
            $b,
        ));

        $magnitudeA = sqrt(array_sum(array_map(
            fn ($x) => $x ** 2,
            $a,
        )));

        $magnitudeB = sqrt(array_sum(array_map(
            fn ($x) => $x ** 2,
            $b,
        )));

        return $magnitudeA && $magnitudeB
            ? $dotProduct / ($magnitudeA * $magnitudeB)
            : 0;
    }
}
```

### Multimodal Embeddings

```php
// Some models support embedding images and audio
$multimodalResponse = $platform->embed(
    new EmbedRequest([
        'text content here',
        new Image(data: $imageData, mimeType: 'image/jpeg'),
        new Audio(data: $audioData, mimeType: 'audio/mpeg'),
    ]),
);

// All content types embedded in same vector space
$embeddings = $multimodalResponse->getVectors();
// Can calculate similarity across text, image, and audio
```

---

## Structured Output

Return structured data (PHP classes, JSON) instead of plain text responses.

### PHP Classes as Output

```php
// Define output class
class MovieRecommendation
{
    public function __construct(
        public string $title,
        public string $genre,
        public float $rating,
        public string $reason,
    ) {}
}

// Request structured output
$response = $platform->complete(
    new Request('Recommend a good sci-fi movie'),
    new Options()
        ->withResponseFormat('json')
        ->withStructuredOutput(MovieRecommendation::class),
);

// Response is automatically cast to the class
/** @var MovieRecommendation */
$movie = $response->getStructuredData();
echo $movie->title;      // "Inception"
echo $movie->genre;      // "Science Fiction"
echo $movie->rating;     // 9.5
echo $movie->reason;     // "Complex plot..."
```

### JSON Schema Definition

```php
use Symfony\Component\Ai\Structured\JsonSchema;

// Define schema explicitly
$schema = new JsonSchema(
    type: 'object',
    properties: [
        'sentiment' => ['type' => 'string', 'enum' => ['positive', 'negative', 'neutral']],
        'confidence' => ['type' => 'number', 'minimum' => 0, 'maximum' => 1],
        'keywords' => [
            'type' => 'array',
            'items' => ['type' => 'string'],
            'maxItems' => 5,
        ],
    ],
    required: ['sentiment', 'confidence'],
);

$response = $platform->complete(
    new Request('Analyze: "I absolutely love this product!"'),
    new Options()->withJsonSchema($schema),
);
```

### PlatformSubscriber for Structured Output

```php
use Symfony\Component\Ai\EventDispatcher\PlatformSubscriber;
use Symfony\Component\Ai\Event\ResponseReceived;

class StructuredOutputSubscriber extends PlatformSubscriber
{
    public static function getSubscribedEvents(): array
    {
        return [
            ResponseReceived::class => 'onResponseReceived',
        ];
    }

    public function onResponseReceived(ResponseReceived $event): void
    {
        $response = $event->getResponse();

        // Parse and validate JSON
        if ($response->getContentType() === 'application/json') {
            $data = json_decode($response->getContent(), true);

            // Validate schema
            $validator = new JsonSchemaValidator();
            if (!$validator->validate($data, $this->getSchema())) {
                // Handle invalid response
                $event->setResponse($this->retryRequest($event));
            }
        }
    }
}
```

---

## Advanced Features

### Parallel Calls

Execute multiple AI requests concurrently:

```php
use Symfony\Component\Ai\Parallel\ParallelPlatform;

class DocumentAnalyzer
{
    public function __construct(
        private PlatformInterface $platform,
    ) {}

    /**
     * Analyze multiple documents in parallel
     */
    public function analyzeDocuments(array $documents): array
    {
        $requests = array_map(
            fn ($doc) => new Request("Summarize: $doc"),
            $documents,
        );

        // Execute all requests concurrently
        $responses = $this->platform->completeParallel($requests);

        return array_map(
            fn ($response) => $response->getContent(),
            $responses,
        );
    }
}
```

### CachedPlatform - Prompt Caching

```php
use Symfony\Component\Ai\Cache\CachedPlatform;

$basePlatform = OpenAiPlatformFactory::create($_ENV['OPENAI_API_KEY']);

// Wrap with caching
$cachedPlatform = new CachedPlatform(
    platform: $basePlatform,
    cache: $cache,  // Symfony Cache component
    ttl: 3600,      // Cache for 1 hour
);

// First call: hits API, caches result
$response1 = $cachedPlatform->complete(
    new Request('Summarize this article...'),
);

// Second identical call: uses cache (much faster, lower cost)
$response2 = $cachedPlatform->complete(
    new Request('Summarize this article...'),
);
```

### FailoverPlatform - Redundancy

```php
use Symfony\Component\Ai\Failover\FailoverPlatform;

// Define fallback chain
$failoverPlatform = new FailoverPlatform([
    $openaiPlatform,      // Try OpenAI first
    $anthropicPlatform,   // Fall back to Anthropic
    $googlePlatform,      // Then Google
    $ollamaPlatform,      // Finally local Ollama
]);

// Automatically tries next provider if one fails
$response = $failoverPlatform->complete(
    new Request('Generate code'),
);
```

### Rate Limiting

```php
use Symfony\Component\Ai\RateLimit\RateLimitedPlatform;
use Symfony\Component\RateLimiter\RateLimiterFactory;

$limiter = new RateLimiterFactory([
    'policy' => 'fixed_window',
    'limit' => 10,      // 10 requests
    'interval' => '1m', // per minute
]);

$limitedPlatform = new RateLimitedPlatform(
    platform: $platform,
    limiter: $limiter->create('openai-api'),
);

try {
    $response = $limitedPlatform->complete(new Request('...'));
} catch (RateLimitExceededException $e) {
    // Wait and retry
    sleep($e->getRetryAfter());
}
```

---

## Testing

### InMemoryPlatform

Testing utility that mocks AI responses without API calls:

```php
use Symfony\Component\Ai\Test\InMemoryPlatform;

class ChatServiceTest extends TestCase
{
    public function testChatResponse(): void
    {
        // Set up mock responses
        $platform = new InMemoryPlatform([
            'What is AI?' => 'AI is artificial intelligence...',
            'Hello' => 'Hi there!',
        ]);

        $service = new ChatService($platform);
        $response = $service->chat('What is AI?');

        $this->assertStringContainsString('artificial intelligence', $response);
    }
}
```

### Mock All Result Types

```php
use Symfony\Component\Ai\Test\InMemoryPlatform;
use Symfony\Component\Ai\Response;
use Symfony\Component\Ai\EmbedResponse;
use Symfony\Component\Ai\TokenUsage;

// Mock text responses
$platform = new InMemoryPlatform([
    'prompt1' => new Response(
        content: 'Response text',
        model: 'gpt-4',
        tokenUsage: new TokenUsage(
            promptTokens: 10,
            completionTokens: 20,
        ),
    ),
]);

// Mock embeddings
$platform->setEmbedding('text', [0.1, 0.2, 0.3, ...]);

// Mock streaming
$platform->setStream('prompt', [
    'chunk1 ',
    'chunk2 ',
    'chunk3',
]);
```

### Testing Streaming

```php
public function testStreamingResponse(): void
{
    $platform = new InMemoryPlatform();
    $platform->setStream('Write code', [
        'function ',
        'hello() ',
        '{ echo ',
        '"Hi"; ',
        '}',
    ]);

    $stream = $platform->completeStream(new Request('Write code'));

    $result = '';
    foreach ($stream as $chunk) {
        $result .= $chunk->getContent();
    }

    $this->assertStringContainsString('function hello', $result);
}
```

---

## Best Practices

### 1. Error Handling & Retries

```php
use Symfony\Component\Ai\Exception\PlatformException;
use Symfony\Component\Ai\Exception\RateLimitException;

try {
    $response = $platform->complete($request);
} catch (RateLimitException $e) {
    // Handle rate limiting - wait and retry
    sleep($e->getRetryAfter());
    $response = $platform->complete($request);
} catch (PlatformException $e) {
    // Handle platform-specific errors
    logger()->error('AI platform error: ' . $e->getMessage());
    // Fall back to cached response or default behavior
}
```

### 2. Token Budget Management

```php
// Monitor token usage to control costs
$response = $platform->complete($request);
$usage = $response->getTokenUsage();

$estimatedCost = ($usage->getPromptTokens() * 0.0001) +
                 ($usage->getCompletionTokens() * 0.0002);

if ($estimatedCost > $dailyBudget) {
    logger()->warning("Token budget exceeded: \${estimatedCost}");
}
```

### 3. Prompt Engineering

```php
// Use system messages for consistent behavior
$messages = new MessageBag([
    Message::system(
        'You are an expert PHP developer. ' .
        'Provide concise, well-commented code examples. ' .
        'Use modern PHP 8.3 syntax.'
    ),
    Message::user('Write a function to sort an array'),
]);

// Include examples (few-shot learning)
$messages->add(Message::user('Example: sort numbers'));
$messages->add(Message::assistant('[1,2,3,4,5]'));
$messages->add(Message::user('Now: sort by date'));
```

### 4. Temperature Settings by Use Case

```php
// Deterministic tasks (0.0)
$extraction = $platform->complete(
    $request,
    new Options()->withTemperature(0.0),  // Data extraction, JSON parsing
);

// Balanced (0.7) - default for most tasks
$summary = $platform->complete(
    $request,
    new Options()->withTemperature(0.7),  // Summaries, Q&A, classification
);

// Creative (1.0+)
$brainstorm = $platform->complete(
    $request,
    new Options()->withTemperature(1.5),  // Brainstorming, creative writing
);
```

### 5. Provider Selection Strategy

```php
// Fast, cheap responses
$fast = OllamaPlatformFactory::create('http://localhost:11434');

// High quality
$quality = AnthropicPlatformFactory::create($_ENV['ANTHROPIC_KEY']);

// Best for vision
$vision = OpenAiPlatformFactory::create($_ENV['OPENAI_KEY']);

// Multi-model approach
class SmartAiService
{
    public function process(Request $request): Response
    {
        // Route by complexity/capability
        if ($this->requiresVision($request)) {
            return $this->visionPlatform->complete($request);
        }

        if ($this->isSimpleTask($request)) {
            return $this->fastPlatform->complete($request);
        }

        return $this->qualityPlatform->complete($request);
    }
}
```

### 6. Cost Optimization

```php
// Use smaller models when possible
$budget = new Options()
    ->withModel('gpt-3.5-turbo')        // Cheaper
    ->withMaxOutputTokens(500)          // Limit output
    ->withTopP(0.9);                    // Faster (nucleus sampling)

// Cache repeated requests
$cachedPlatform = new CachedPlatform(
    $basePlatform,
    $cache,
    ttl: 86400,  // Cache for 24 hours
);

// Batch similar requests
$requests = array_chunk($documents, 10);
foreach ($requests as $batch) {
    $responses = $platform->completeParallel(
        array_map(fn ($doc) => new Request($doc), $batch)
    );
}
```

### 7. Security Considerations

```php
// Never include secrets in prompts
$response = $platform->complete(
    // WRONG:
    // new Request("User email: {$user->getEmail()}, API key: {$apiKey}")

    // RIGHT:
    new Request("Process user data - do not output email or API keys")
);

// Validate structured output
try {
    $output = json_decode($response->getContent(), true);

    // Validate against expected schema
    if (!$this->validator->validate($output, $expectedSchema)) {
        throw new InvalidResponseException('Invalid output structure');
    }
} catch (JsonException $e) {
    logger()->warning('Invalid JSON in AI response');
}

// Rate limit per user
$userLimiter = $this->rateLimiterFactory->create("user_{$userId}");
$userLimiter->ensureAccepted();
```

### 8. Logging & Monitoring

```php
// Log all AI interactions for debugging and compliance
logger()->info('AI request', [
    'model' => $request->getModel(),
    'prompt_tokens' => $usage->getPromptTokens(),
    'completion_tokens' => $usage->getCompletionTokens(),
    'cost' => $estimatedCost,
    'duration' => microtime(true) - $startTime,
]);

// Monitor for unusual patterns
if ($usage->getCompletionTokens() > 4000) {
    logger()->warning('Unusually long completion', [
        'tokens' => $usage->getCompletionTokens(),
    ]);
}
```

This comprehensive reference covers the full scope of Symfony AI Platform's capabilities, from basic usage to advanced patterns and best practices.
