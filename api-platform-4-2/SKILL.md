---
name: "api-platform-4-2"
description: "API Platform 4.2 reference for building REST and GraphQL APIs with Symfony. Use when creating API resources, configuring operations, working with DTOs, state providers/processors, serialization, filters, validation, security, pagination, or any API Platform-related code. Triggers on: API Platform, ApiResource, ApiProperty, ApiFilter, State Provider, State Processor, DTO, normalization groups, denormalization, OpenAPI, Hydra, GraphQL, REST API, collection operations, item operations, subresources, ApiTestCase."
---

# API Platform 4.2

GitHub: https://github.com/api-platform/api-platform
Docs: https://api-platform.com/docs/symfony/

## Overview

API Platform is a powerful framework for building REST and GraphQL APIs with Symfony. Version 4.2 introduces the new metadata system with PHP attributes, state providers/processors architecture, and improved DTO support.

## Quick Reference

### Basic API Resource

```php
use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
class Book
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private ?string $title = null;

    // Getters and setters...
}
```

### Custom Operations

```php
#[ApiResource(
    operations: [
        new Get(),
        new GetCollection(),
        new Post(),
        new Put(),
        new Patch(),
        new Delete(),
        new Get(
            uriTemplate: '/books/{id}/publish',
            name: 'publish',
            controller: PublishBookController::class
        )
    ]
)]
class Book
{
    // ...
}
```

### DTO Input/Output

```php
#[ApiResource(
    input: BookInput::class,
    output: BookOutput::class
)]
class Book
{
    // ...
}

class BookInput
{
    public string $title;
    public string $author;
}

class BookOutput
{
    public int $id;
    public string $title;
    public string $author;
    public \DateTimeInterface $createdAt;
}
```

### Serialization Groups

```php
use Symfony\Component\Serializer\Annotation\Groups;

#[ApiResource(
    normalizationContext: ['groups' => ['book:read']],
    denormalizationContext: ['groups' => ['book:write']]
)]
class Book
{
    #[Groups(['book:read'])]
    private ?int $id = null;

    #[Groups(['book:read', 'book:write'])]
    private ?string $title = null;

    #[Groups(['book:read'])]
    private ?\DateTimeInterface $createdAt = null;
}
```

### Filters

```php
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;
use ApiPlatform\Metadata\ApiFilter;

#[ApiResource]
#[ApiFilter(SearchFilter::class, properties: ['title' => 'partial', 'author' => 'exact'])]
#[ApiFilter(OrderFilter::class, properties: ['createdAt', 'title'])]
class Book
{
    // ...
}
```

### State Provider

```php
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;

class BookProvider implements ProviderInterface
{
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        // Custom logic to fetch data
        return $this->fetchBooks($uriVariables, $context);
    }
}

#[ApiResource(provider: BookProvider::class)]
class Book
{
    // ...
}
```

### State Processor

```php
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

class BookProcessor implements ProcessorInterface
{
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // Custom logic before/after persistence
        $this->sendNotification($data);
        return $data;
    }
}

#[ApiResource(processor: BookProcessor::class)]
class Book
{
    // ...
}
```

### Validation

```php
use Symfony\Component\Validator\Constraints as Assert;

#[ApiResource]
class Book
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 3, max: 255)]
    private ?string $title = null;

    #[Assert\NotBlank]
    #[Assert\Email]
    private ?string $authorEmail = null;
}
```

### Security

```php
#[ApiResource(
    security: "is_granted('ROLE_USER')",
    operations: [
        new Get(security: "is_granted('VIEW', object)"),
        new Put(security: "is_granted('EDIT', object)"),
        new Delete(security: "is_granted('ROLE_ADMIN')")
    ]
)]
class Book
{
    // ...
}
```

### Pagination

```php
#[ApiResource(
    paginationEnabled: true,
    paginationItemsPerPage: 30,
    paginationMaximumItemsPerPage: 100,
    paginationClientEnabled: true,
    paginationClientItemsPerPage: true
)]
class Book
{
    // ...
}
```

### Testing

```php
use ApiPlatform\Symfony\Bundle\Test\ApiTestCase;

class BookTest extends ApiTestCase
{
    public function testGetCollection(): void
    {
        $response = static::createClient()->request('GET', '/api/books');

        $this->assertResponseIsSuccessful();
        $this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');
        $this->assertJsonContains([
            '@context' => '/api/contexts/Book',
            '@id' => '/api/books',
            '@type' => 'hydra:Collection',
        ]);
        $this->assertCount(10, $response->toArray()['hydra:member']);
    }
}
```

## Full Documentation

For complete details including advanced operations, state architecture, GraphQL support, Mercure integration, custom controllers, events, doctrine extensions, OpenAPI customization, subresources, and all filter types, see [references/api-platform.md](references/api-platform.md).
