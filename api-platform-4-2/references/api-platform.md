# API Platform 4.2 Complete Reference

## Table of Contents

1. [Resources & Metadata](#resources--metadata)
2. [Operations](#operations)
3. [State Architecture](#state-architecture)
4. [DTO Input/Output](#dto-inputoutput)
5. [Serialization](#serialization)
6. [Filters](#filters)
7. [Validation](#validation)
8. [Security](#security)
9. [Pagination](#pagination)
10. [Subresources](#subresources)
11. [Events](#events)
12. [OpenAPI Documentation](#openapi-documentation)
13. [GraphQL](#graphql)
14. [Testing](#testing)
15. [Doctrine Extensions](#doctrine-extensions)
16. [Custom Controllers](#custom-controllers)
17. [Mercure Integration](#mercure-integration)

---

## Resources & Metadata

### ApiResource Attribute

The `#[ApiResource]` attribute exposes a class as an API resource.

```php
use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
class Book
{
    // ...
}
```

### ApiResource Parameters

```php
#[ApiResource(
    shortName: 'Product',                    // Custom resource name
    description: 'A product in the catalog',
    types: ['https://schema.org/Product'],   // JSON-LD types
    routePrefix: '/store',                   // URI prefix
    extraProperties: ['custom' => 'value'],  // Custom metadata
    paginationEnabled: true,
    paginationItemsPerPage: 30,
    security: "is_granted('ROLE_USER')",
    mercure: true,                           // Enable Mercure updates
    elasticsearch: true                      // Enable Elasticsearch
)]
```

### ApiProperty Attribute

Customize property behavior:

```php
use ApiPlatform\Metadata\ApiProperty;

class Book
{
    #[ApiProperty(
        identifier: true,                    // Mark as identifier
        readable: true,                      // Readable in output
        writable: false,                     // Not writable in input
        required: true,                      // Required field
        description: 'Unique identifier',
        openapiContext: ['type' => 'integer']
    )]
    private ?int $id = null;

    #[ApiProperty(
        writable: true,
        required: true,
        example: 'The Great Gatsby'
    )]
    private ?string $title = null;
}
```

### Multiple API Resources

Use multiple `#[ApiResource]` attributes for different contexts:

```php
#[ApiResource(routePrefix: '/admin')]
#[ApiResource(routePrefix: '/public', security: "is_granted('PUBLIC_ACCESS', object)")]
class Book
{
    // ...
}
```

---

## Operations

### Built-in Operations

```php
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\Put;
use ApiPlatform\Metadata\Patch;
use ApiPlatform\Metadata\Delete;

#[ApiResource(
    operations: [
        new Get(),                           // GET /books/{id}
        new GetCollection(),                 // GET /books
        new Post(),                          // POST /books
        new Put(),                           // PUT /books/{id}
        new Patch(),                         // PATCH /books/{id}
        new Delete()                         // DELETE /books/{id}
    ]
)]
```

### Custom Operations

```php
#[ApiResource(
    operations: [
        new Get(
            uriTemplate: '/books/{id}/publish',
            name: 'publish',
            controller: PublishBookController::class,
            read: false,                     // Don't fetch entity automatically
            deserialize: false,              // Don't deserialize input
            validate: false,                 // Don't validate
            write: false                     // Don't persist
        ),
        new Post(
            uriTemplate: '/books/{id}/reviews',
            name: 'add_review',
            processor: AddReviewProcessor::class
        )
    ]
)]
```

### Operation Configuration

```php
new Get(
    uriTemplate: '/books/{id}',
    uriVariables: ['id'],
    name: 'get_book',
    shortName: 'Book',
    description: 'Retrieves a book resource',
    normalizationContext: ['groups' => ['book:read', 'book:detail']],
    security: "is_granted('VIEW', object)",
    provider: BookProvider::class,
    output: BookOutput::class,
    openapiContext: [
        'summary' => 'Get a book',
        'description' => 'Retrieves a book by ID',
        'responses' => [
            '200' => ['description' => 'Book retrieved'],
            '404' => ['description' => 'Book not found']
        ]
    ]
)
```

### HTTP Methods

```php
use ApiPlatform\Metadata\HttpOperation;

#[ApiResource(
    operations: [
        new HttpOperation(
            method: 'GET',
            uriTemplate: '/custom/{id}'
        )
    ]
)]
```

---

## State Architecture

### State Providers

Providers fetch data for GET operations.

#### Simple Provider

```php
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;

class BookProvider implements ProviderInterface
{
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        // For item operations
        if ($operation instanceof Get) {
            return $this->findOneBy(['id' => $uriVariables['id']]);
        }

        // For collection operations
        return $this->findAll();
    }

    private function findOneBy(array $criteria): ?Book
    {
        // Custom fetch logic
    }

    private function findAll(): array
    {
        // Custom fetch logic
    }
}
```

#### Provider with Pagination

```php
use ApiPlatform\State\Pagination\Pagination;
use ApiPlatform\State\Pagination\PaginatorInterface;

class BookProvider implements ProviderInterface
{
    public function __construct(
        private Pagination $pagination
    ) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        $page = $this->pagination->getPage($context);
        $itemsPerPage = $this->pagination->getLimit($operation, $context);
        $offset = $this->pagination->getOffset($operation, $context);

        return new BookPaginator(
            $this->fetchBooks($offset, $itemsPerPage),
            $page,
            $itemsPerPage,
            $this->countBooks()
        );
    }
}

class BookPaginator implements PaginatorInterface, \IteratorAggregate
{
    public function __construct(
        private array $items,
        private int $currentPage,
        private int $itemsPerPage,
        private int $totalItems
    ) {}

    public function getCurrentPage(): int
    {
        return $this->currentPage;
    }

    public function getItemsPerPage(): int
    {
        return $this->itemsPerPage;
    }

    public function getTotalItems(): int
    {
        return $this->totalItems;
    }

    public function count(): int
    {
        return count($this->items);
    }

    public function getIterator(): \Traversable
    {
        return new \ArrayIterator($this->items);
    }
}
```

### State Processors

Processors handle mutations (POST, PUT, PATCH, DELETE).

#### Simple Processor

```php
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;

class BookProcessor implements ProcessorInterface
{
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // Pre-processing
        $data->setUpdatedAt(new \DateTime());

        // Persist (if needed)
        $this->save($data);

        // Post-processing
        $this->sendNotification($data);

        return $data;
    }

    private function save(Book $book): void
    {
        // Custom persistence logic
    }

    private function sendNotification(Book $book): void
    {
        // Send email, trigger event, etc.
    }
}
```

#### Processor with Existing Processor Delegation

```php
class BookProcessor implements ProcessorInterface
{
    public function __construct(
        private ProcessorInterface $persistProcessor,
        private ProcessorInterface $removeProcessor
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // Custom pre-processing
        $this->validateBusinessRules($data);

        // Delegate to standard processor
        if ($operation instanceof Delete) {
            $result = $this->removeProcessor->process($data, $operation, $uriVariables, $context);
        } else {
            $result = $this->persistProcessor->process($data, $operation, $uriVariables, $context);
        }

        // Custom post-processing
        $this->logOperation($operation, $data);

        return $result;
    }
}

// services.yaml
services:
    App\State\BookProcessor:
        arguments:
            $persistProcessor: '@api_platform.doctrine.orm.state.persist_processor'
            $removeProcessor: '@api_platform.doctrine.orm.state.remove_processor'
```

---

## DTO Input/Output

### Input DTO

Map request data to a DTO before processing:

```php
#[ApiResource(input: CreateBookInput::class)]
class Book
{
    // ...
}

class CreateBookInput
{
    #[Assert\NotBlank]
    public string $title;

    #[Assert\NotBlank]
    public string $author;

    public ?string $isbn = null;
}
```

### Output DTO

Transform entity to DTO for response:

```php
#[ApiResource(output: BookOutput::class)]
class Book
{
    // ...
}

class BookOutput
{
    public int $id;
    public string $title;
    public string $author;
    public \DateTimeInterface $publishedAt;
    public int $reviewCount;
}
```

### Data Transformer

Transform between entity and DTO:

```php
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;

class BookOutputProvider implements ProviderInterface
{
    public function __construct(
        private ProviderInterface $itemProvider
    ) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        $book = $this->itemProvider->provide($operation, $uriVariables, $context);

        if (!$book instanceof Book) {
            return $book;
        }

        return $this->transformToOutput($book);
    }

    private function transformToOutput(Book $book): BookOutput
    {
        $output = new BookOutput();
        $output->id = $book->getId();
        $output->title = $book->getTitle();
        $output->author = $book->getAuthor();
        $output->publishedAt = $book->getPublishedAt();
        $output->reviewCount = $book->getReviews()->count();

        return $output;
    }
}
```

### Input Processor

```php
class CreateBookProcessor implements ProcessorInterface
{
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        if (!$data instanceof CreateBookInput) {
            return $data;
        }

        $book = new Book();
        $book->setTitle($data->title);
        $book->setAuthor($data->author);
        $book->setIsbn($data->isbn);

        // Persist
        $this->entityManager->persist($book);
        $this->entityManager->flush();

        return $book;
    }
}
```

---

## Serialization

### Normalization Groups

Control which fields are exposed:

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

    #[Groups(['book:write'])]
    private ?string $password = null;
}
```

### Operation-Specific Groups

```php
#[ApiResource(
    operations: [
        new Get(normalizationContext: ['groups' => ['book:read', 'book:detail']]),
        new GetCollection(normalizationContext: ['groups' => ['book:read', 'book:list']]),
        new Post(denormalizationContext: ['groups' => ['book:create']]),
        new Put(denormalizationContext: ['groups' => ['book:update']])
    ]
)]
```

### Nested Groups

```php
class Book
{
    #[Groups(['book:read'])]
    private ?int $id = null;

    #[Groups(['book:read', 'book:write'])]
    private ?string $title = null;

    #[Groups(['book:read:author'])]
    private ?Author $author = null;
}

class Author
{
    #[Groups(['book:read:author'])]
    private ?int $id = null;

    #[Groups(['book:read:author'])]
    private ?string $name = null;
}
```

### Context Builder

Dynamically modify serialization context:

```php
use ApiPlatform\Serializer\SerializerContextBuilderInterface;
use Symfony\Component\HttpFoundation\Request;

class BookContextBuilder implements SerializerContextBuilderInterface
{
    public function __construct(
        private SerializerContextBuilderInterface $decorated
    ) {}

    public function createFromRequest(Request $request, bool $normalization, array $extractedAttributes = null): array
    {
        $context = $this->decorated->createFromRequest($request, $normalization, $extractedAttributes);

        // Add custom context
        if ($normalization && $this->isAdmin()) {
            $context['groups'][] = 'admin:read';
        }

        return $context;
    }

    private function isAdmin(): bool
    {
        // Check if current user is admin
    }
}
```

### MaxDepth

Prevent circular references:

```php
use Symfony\Component\Serializer\Annotation\MaxDepth;

#[ApiResource(
    normalizationContext: [
        'groups' => ['book:read'],
        'enable_max_depth' => true
    ]
)]
class Book
{
    #[Groups(['book:read'])]
    #[MaxDepth(1)]
    private ?Author $author = null;
}
```

---

## Filters

### Search Filter

```php
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
use ApiPlatform\Metadata\ApiFilter;

#[ApiResource]
#[ApiFilter(SearchFilter::class, properties: [
    'title' => 'partial',      // LIKE %value%
    'author' => 'exact',        // = value
    'isbn' => 'start',          // LIKE value%
    'description' => 'end',     // LIKE %value
    'category.name' => 'exact'  // Nested property
])]
class Book
{
    // ...
}
```

**Usage:**
- `/api/books?title=gatsby` - partial match
- `/api/books?author=Fitzgerald` - exact match
- `/api/books?category.name=Fiction` - nested property

### Date Filter

```php
use ApiPlatform\Doctrine\Orm\Filter\DateFilter;

#[ApiFilter(DateFilter::class, properties: ['publishedAt', 'createdAt'])]
class Book
{
    private ?\DateTimeInterface $publishedAt = null;
    private ?\DateTimeInterface $createdAt = null;
}
```

**Usage:**
- `/api/books?publishedAt[after]=2020-01-01`
- `/api/books?publishedAt[before]=2023-12-31`
- `/api/books?publishedAt[strictly_after]=2020-01-01`
- `/api/books?publishedAt[strictly_before]=2023-12-31`

### Range Filter

```php
use ApiPlatform\Doctrine\Orm\Filter\RangeFilter;

#[ApiFilter(RangeFilter::class, properties: ['price', 'pages'])]
class Book
{
    private ?float $price = null;
    private ?int $pages = null;
}
```

**Usage:**
- `/api/books?price[gt]=10` - greater than
- `/api/books?price[gte]=10` - greater than or equal
- `/api/books?price[lt]=50` - less than
- `/api/books?price[lte]=50` - less than or equal
- `/api/books?price[between]=10..50` - between range

### Order Filter

```php
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;

#[ApiFilter(OrderFilter::class, properties: [
    'title' => 'ASC',
    'publishedAt' => 'DESC',
    'price'
])]
class Book
{
    // ...
}
```

**Usage:**
- `/api/books?order[title]=asc`
- `/api/books?order[publishedAt]=desc`
- `/api/books?order[title]=asc&order[price]=desc`

### Boolean Filter

```php
use ApiPlatform\Doctrine\Orm\Filter\BooleanFilter;

#[ApiFilter(BooleanFilter::class, properties: ['available', 'featured'])]
class Book
{
    private bool $available = true;
    private bool $featured = false;
}
```

**Usage:**
- `/api/books?available=true`
- `/api/books?featured=1`
- `/api/books?available=false`

### Numeric Filter

```php
use ApiPlatform\Doctrine\Orm\Filter\NumericFilter;

#[ApiFilter(NumericFilter::class, properties: ['id', 'pages'])]
class Book
{
    // ...
}
```

### Exists Filter

```php
use ApiPlatform\Doctrine\Orm\Filter\ExistsFilter;

#[ApiFilter(ExistsFilter::class, properties: ['isbn', 'description'])]
class Book
{
    private ?string $isbn = null;
    private ?string $description = null;
}
```

**Usage:**
- `/api/books?exists[isbn]=true` - has ISBN
- `/api/books?exists[description]=false` - no description

### Custom Filter

```php
use ApiPlatform\Doctrine\Orm\Filter\AbstractFilter;
use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use Doctrine\ORM\QueryBuilder;

class DiscountedFilter extends AbstractFilter
{
    protected function filterProperty(
        string $property,
        $value,
        QueryBuilder $queryBuilder,
        QueryNameGeneratorInterface $queryNameGenerator,
        string $resourceClass,
        Operation $operation = null,
        array $context = []
    ): void {
        if ($property !== 'discounted') {
            return;
        }

        $alias = $queryBuilder->getRootAliases()[0];
        $parameterName = $queryNameGenerator->generateParameterName('discounted');

        if ($value === 'true' || $value === '1') {
            $queryBuilder
                ->andWhere("$alias.discount > 0")
                ->setParameter($parameterName, 0);
        }
    }

    public function getDescription(string $resourceClass): array
    {
        return [
            'discounted' => [
                'property' => 'discounted',
                'type' => 'bool',
                'required' => false,
                'description' => 'Filter books with discount'
            ]
        ];
    }
}

#[ApiFilter(DiscountedFilter::class)]
class Book
{
    // ...
}
```

---

## Validation

### Basic Validation

```php
use Symfony\Component\Validator\Constraints as Assert;

#[ApiResource]
class Book
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 3, max: 255)]
    private ?string $title = null;

    #[Assert\NotBlank]
    #[Assert\Isbn]
    private ?string $isbn = null;

    #[Assert\Email]
    private ?string $authorEmail = null;

    #[Assert\Range(min: 0, max: 10000)]
    private ?float $price = null;

    #[Assert\Positive]
    private ?int $pages = null;
}
```

### Validation Groups

```php
#[ApiResource(
    operations: [
        new Post(validationContext: ['groups' => ['Default', 'book:create']]),
        new Put(validationContext: ['groups' => ['Default', 'book:update']])
    ]
)]
class Book
{
    #[Assert\NotBlank(groups: ['book:create'])]
    private ?string $title = null;

    #[Assert\NotBlank(groups: ['book:create', 'book:update'])]
    private ?string $author = null;
}
```

### Custom Validator

```php
use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;

#[Attribute]
class ValidIsbn extends Constraint
{
    public string $message = 'The ISBN "{{ value }}" is not valid.';
}

class ValidIsbnValidator extends ConstraintValidator
{
    public function validate($value, Constraint $constraint): void
    {
        if (!$this->isValidIsbn($value)) {
            $this->context->buildViolation($constraint->message)
                ->setParameter('{{ value }}', $value)
                ->addViolation();
        }
    }

    private function isValidIsbn(string $isbn): bool
    {
        // Custom ISBN validation logic
        return true;
    }
}

class Book
{
    #[ValidIsbn]
    private ?string $isbn = null;
}
```

---

## Security

### Resource-Level Security

```php
#[ApiResource(
    security: "is_granted('ROLE_USER')",
    securityMessage: "Only authenticated users can access books."
)]
class Book
{
    // ...
}
```

### Operation-Level Security

```php
#[ApiResource(
    operations: [
        new Get(security: "is_granted('VIEW', object)"),
        new GetCollection(security: "is_granted('ROLE_USER')"),
        new Post(security: "is_granted('ROLE_ADMIN')"),
        new Put(security: "is_granted('EDIT', object)"),
        new Delete(security: "is_granted('ROLE_ADMIN')")
    ]
)]
```

### Security with Object

```php
#[ApiResource(
    security: "is_granted('ROLE_USER') and object.getOwner() == user",
    securityPostDenormalize: "is_granted('EDIT', object)"
)]
class Book
{
    private ?User $owner = null;
}
```

### Voter

```php
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class BookVoter extends Voter
{
    public const VIEW = 'VIEW';
    public const EDIT = 'EDIT';

    protected function supports(string $attribute, mixed $subject): bool
    {
        return in_array($attribute, [self::VIEW, self::EDIT])
            && $subject instanceof Book;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();

        if (!$user instanceof User) {
            return false;
        }

        /** @var Book $book */
        $book = $subject;

        return match($attribute) {
            self::VIEW => $this->canView($book, $user),
            self::EDIT => $this->canEdit($book, $user),
            default => false
        };
    }

    private function canView(Book $book, User $user): bool
    {
        return $book->isPublished() || $book->getOwner() === $user;
    }

    private function canEdit(Book $book, User $user): bool
    {
        return $book->getOwner() === $user;
    }
}
```

---

## Pagination

### Enable/Disable Pagination

```php
#[ApiResource(
    paginationEnabled: true,              // Enable pagination (default)
    paginationItemsPerPage: 30,          // Items per page (default 30)
    paginationMaximumItemsPerPage: 100,  // Max items per page
    paginationClientEnabled: true,       // Allow client to enable/disable
    paginationClientItemsPerPage: true,  // Allow client to set items per page
    paginationClientPartial: true        // Allow partial pagination
)]
```

**Usage:**
- `/api/books?page=2` - Get page 2
- `/api/books?itemsPerPage=50` - 50 items per page
- `/api/books?pagination=false` - Disable pagination

### Partial Pagination

More efficient for large datasets:

```php
#[ApiResource(
    paginationPartial: true,             // Enable partial pagination
    paginationViaCursor: [               // Cursor-based pagination
        ['field' => 'id', 'direction' => 'ASC']
    ]
)]
```

### Custom Pagination

```php
use ApiPlatform\State\Pagination\PaginatorInterface;

class BookPaginator implements PaginatorInterface
{
    private array $items;
    private int $currentPage;
    private int $itemsPerPage;
    private int $totalItems;

    public function count(): int
    {
        return count($this->items);
    }

    public function getCurrentPage(): int
    {
        return $this->currentPage;
    }

    public function getItemsPerPage(): int
    {
        return $this->itemsPerPage;
    }

    public function getLastPage(): int
    {
        return (int) ceil($this->totalItems / $this->itemsPerPage);
    }

    public function getTotalItems(): int
    {
        return $this->totalItems;
    }

    public function getIterator(): \Traversable
    {
        return new \ArrayIterator($this->items);
    }
}
```

---

## Subresources

### Define Subresource

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Link;

#[ApiResource]
class Book
{
    #[ORM\OneToMany(targetEntity: Review::class, mappedBy: 'book')]
    private Collection $reviews;
}

#[ApiResource(
    uriTemplate: '/books/{bookId}/reviews',
    uriVariables: [
        'bookId' => new Link(
            fromClass: Book::class,
            fromProperty: 'reviews'
        )
    ]
)]
class Review
{
    #[ORM\ManyToOne(targetEntity: Book::class, inversedBy: 'reviews')]
    private ?Book $book = null;
}
```

**Generates:**
- `GET /api/books/{id}/reviews` - Get all reviews for a book
- `GET /api/books/{bookId}/reviews/{id}` - Get specific review

### Nested Subresources

```php
#[ApiResource(
    uriTemplate: '/authors/{authorId}/books/{bookId}/reviews',
    uriVariables: [
        'authorId' => new Link(
            fromClass: Author::class,
            fromProperty: 'books'
        ),
        'bookId' => new Link(
            fromClass: Book::class,
            fromProperty: 'reviews'
        )
    ]
)]
class Review
{
    // ...
}
```

---

## Events

### Doctrine Events

```php
use ApiPlatform\Symfony\EventListener\EventPriorities;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Event\ViewEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class BookSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::VIEW => ['setOwner', EventPriorities::PRE_WRITE],
        ];
    }

    public function setOwner(ViewEvent $event): void
    {
        $book = $event->getControllerResult();
        $method = $event->getRequest()->getMethod();

        if (!$book instanceof Book || Request::METHOD_POST !== $method) {
            return;
        }

        $book->setOwner($this->security->getUser());
    }
}
```

### Event Priorities

- `EventPriorities::PRE_READ` - Before provider
- `EventPriorities::POST_READ` - After provider, before serialization
- `EventPriorities::PRE_DESERIALIZE` - Before deserialization
- `EventPriorities::POST_DESERIALIZE` - After deserialization
- `EventPriorities::PRE_VALIDATE` - Before validation
- `EventPriorities::POST_VALIDATE` - After validation
- `EventPriorities::PRE_WRITE` - Before processor
- `EventPriorities::POST_WRITE` - After processor
- `EventPriorities::PRE_RESPOND` - Before response
- `EventPriorities::POST_RESPOND` - After response

---

## OpenAPI Documentation

### Customize OpenAPI

```php
#[ApiResource(
    operations: [
        new Get(
            openapiContext: [
                'summary' => 'Retrieve a book',
                'description' => 'Retrieves a book resource by its identifier',
                'parameters' => [
                    [
                        'name' => 'id',
                        'in' => 'path',
                        'required' => true,
                        'schema' => ['type' => 'integer']
                    ]
                ],
                'responses' => [
                    '200' => [
                        'description' => 'Book resource',
                        'content' => [
                            'application/ld+json' => [
                                'schema' => [
                                    '$ref' => '#/components/schemas/Book'
                                ]
                            ]
                        ]
                    ],
                    '404' => ['description' => 'Resource not found']
                ]
            ]
        )
    ]
)]
```

### Global OpenAPI Customization

```php
use ApiPlatform\OpenApi\Factory\OpenApiFactoryInterface;
use ApiPlatform\OpenApi\OpenApi;
use ApiPlatform\OpenApi\Model\PathItem;

class OpenApiFactory implements OpenApiFactoryInterface
{
    public function __construct(
        private OpenApiFactoryInterface $decorated
    ) {}

    public function __invoke(array $context = []): OpenApi
    {
        $openApi = $this->decorated->__invoke($context);

        // Customize OpenAPI
        $openApi = $openApi->withInfo(
            $openApi->getInfo()
                ->withTitle('My API')
                ->withVersion('2.0.0')
                ->withDescription('Custom API description')
        );

        return $openApi;
    }
}
```

---

## GraphQL

### Enable GraphQL

```yaml
# config/packages/api_platform.yaml
api_platform:
    graphql:
        enabled: true
        graphiql:
            enabled: true
        graphql_playground:
            enabled: true
```

### GraphQL Resource

```php
#[ApiResource(
    graphQlOperations: [
        new Query(),
        new QueryCollection(),
        new Mutation(name: 'create'),
        new Mutation(name: 'update'),
        new Mutation(name: 'delete')
    ]
)]
class Book
{
    // ...
}
```

### Custom GraphQL Resolver

```php
use ApiPlatform\GraphQl\Resolver\MutationResolverInterface;

class PublishBookResolver implements MutationResolverInterface
{
    public function __invoke($item, array $context): Book
    {
        $item->setPublished(true);
        $item->setPublishedAt(new \DateTime());

        return $item;
    }
}

#[ApiResource(
    graphQlOperations: [
        new Mutation(
            name: 'publish',
            resolver: PublishBookResolver::class
        )
    ]
)]
```

---

## Testing

### ApiTestCase

```php
use ApiPlatform\Symfony\Bundle\Test\ApiTestCase;
use Hautelook\AliceBundle\PhpUnit\RefreshDatabaseTrait;

class BookTest extends ApiTestCase
{
    use RefreshDatabaseTrait;

    public function testGetCollection(): void
    {
        $response = static::createClient()->request('GET', '/api/books');

        $this->assertResponseIsSuccessful();
        $this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');

        $this->assertJsonContains([
            '@context' => '/api/contexts/Book',
            '@id' => '/api/books',
            '@type' => 'hydra:Collection',
            'hydra:totalItems' => 10
        ]);

        $this->assertCount(10, $response->toArray()['hydra:member']);
    }

    public function testCreateBook(): void
    {
        $client = static::createClient();
        $response = $client->request('POST', '/api/books', [
            'json' => [
                'title' => 'New Book',
                'author' => 'John Doe',
                'isbn' => '978-3-16-148410-0'
            ]
        ]);

        $this->assertResponseStatusCodeSame(201);
        $this->assertResponseHeaderSame('content-type', 'application/ld+json; charset=utf-8');
        $this->assertJsonContains([
            '@context' => '/api/contexts/Book',
            '@type' => 'Book',
            'title' => 'New Book',
            'author' => 'John Doe'
        ]);
        $this->assertMatchesRegularExpression('~^/api/books/\d+$~', $response->toArray()['@id']);
    }

    public function testUpdateBook(): void
    {
        $client = static::createClient();
        $iri = $this->findIriBy(Book::class, ['title' => 'Existing Book']);

        $client->request('PUT', $iri, [
            'json' => [
                'title' => 'Updated Title'
            ]
        ]);

        $this->assertResponseIsSuccessful();
        $this->assertJsonContains([
            '@id' => $iri,
            'title' => 'Updated Title'
        ]);
    }

    public function testDeleteBook(): void
    {
        $client = static::createClient();
        $iri = $this->findIriBy(Book::class, ['title' => 'Book to Delete']);

        $client->request('DELETE', $iri);

        $this->assertResponseStatusCodeSame(204);
        $this->assertNull(
            static::getContainer()->get('doctrine')->getRepository(Book::class)->findOneBy(['title' => 'Book to Delete'])
        );
    }

    public function testValidation(): void
    {
        $response = static::createClient()->request('POST', '/api/books', [
            'json' => [
                'title' => '', // Invalid: blank
                'author' => 'John Doe'
            ]
        ]);

        $this->assertResponseStatusCodeSame(422);
        $this->assertResponseHeaderSame('content-type', 'application/problem+json; charset=utf-8');

        $this->assertJsonContains([
            '@type' => 'ConstraintViolationList',
            'violations' => [
                [
                    'propertyPath' => 'title',
                    'message' => 'This value should not be blank.'
                ]
            ]
        ]);
    }

    public function testSecurity(): void
    {
        $response = static::createClient()->request('POST', '/api/books', [
            'json' => ['title' => 'Unauthorized Book']
        ]);

        $this->assertResponseStatusCodeSame(401);
    }
}
```

### Authenticated Tests

```php
public function testAuthenticatedAccess(): void
{
    $client = static::createClient();
    $user = $this->createUser();

    $client->loginUser($user);

    $response = $client->request('POST', '/api/books', [
        'json' => [
            'title' => 'Authenticated Book'
        ]
    ]);

    $this->assertResponseIsSuccessful();
}

private function createUser(): User
{
    $user = new User();
    $user->setEmail('test@example.com');
    $user->setPassword('password');

    $em = static::getContainer()->get('doctrine')->getManager();
    $em->persist($user);
    $em->flush();

    return $user;
}
```

---

## Doctrine Extensions

### Softdeleteable

```php
use Gedmo\Mapping\Annotation as Gedmo;

#[ApiResource]
#[Gedmo\SoftDeleteable(fieldName: 'deletedAt')]
class Book
{
    #[ORM\Column(type: 'datetime', nullable: true)]
    private ?\DateTimeInterface $deletedAt = null;
}
```

### Timestampable

```php
#[ApiResource]
class Book
{
    #[Gedmo\Timestampable(on: 'create')]
    #[ORM\Column(type: 'datetime')]
    private ?\DateTimeInterface $createdAt = null;

    #[Gedmo\Timestampable(on: 'update')]
    #[ORM\Column(type: 'datetime')]
    private ?\DateTimeInterface $updatedAt = null;
}
```

### Blameable

```php
#[ApiResource]
class Book
{
    #[Gedmo\Blameable(on: 'create')]
    #[ORM\ManyToOne(targetEntity: User::class)]
    private ?User $createdBy = null;

    #[Gedmo\Blameable(on: 'update')]
    #[ORM\ManyToOne(targetEntity: User::class)]
    private ?User $updatedBy = null;
}
```

---

## Custom Controllers

When you need full control over the request/response:

```php
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;

class PublishBookController extends AbstractController
{
    public function __invoke(Book $book): JsonResponse
    {
        if ($book->isPublished()) {
            return new JsonResponse(['error' => 'Book already published'], 400);
        }

        $book->setPublished(true);
        $book->setPublishedAt(new \DateTime());

        $this->entityManager->flush();

        return new JsonResponse([
            'message' => 'Book published successfully',
            'book' => [
                'id' => $book->getId(),
                'title' => $book->getTitle()
            ]
        ]);
    }
}

#[ApiResource(
    operations: [
        new Get(
            uriTemplate: '/books/{id}/publish',
            controller: PublishBookController::class,
            name: 'publish'
        )
    ]
)]
class Book
{
    // ...
}
```

---

## Mercure Integration

Real-time updates with Mercure:

```yaml
# config/packages/api_platform.yaml
api_platform:
    mercure:
        enabled: true
        hub_url: '%env(MERCURE_PUBLIC_URL)%'
```

```php
#[ApiResource(mercure: true)]
class Book
{
    // ...
}
```

When a Book is created/updated/deleted, Mercure automatically publishes updates to subscribed clients.

### Custom Mercure Topics

```php
#[ApiResource(
    mercure: [
        'topics' => [
            '@=iri(object)',
            '@="/books/latest"'
        ]
    ]
)]
```

---

## Best Practices

1. **Use DTOs for complex transformations** - Keep entity clean, handle transformations in DTOs
2. **Leverage State Providers/Processors** - Centralize business logic, reuse across operations
3. **Use serialization groups wisely** - Prevent over-fetching, control data exposure
4. **Apply security at operation level** - More granular control than resource level
5. **Write comprehensive tests** - Use ApiTestCase for integration tests
6. **Document with OpenAPI** - Custom contexts improve API documentation
7. **Use filters for common queries** - Built-in filters cover 80% of use cases
8. **Paginate collections** - Essential for performance on large datasets
9. **Validate input** - Use Symfony validators, custom validators when needed
10. **Monitor performance** - Use profiler, optimize queries with Doctrine extensions

---

## Common Patterns

### Read-Only API

```php
#[ApiResource(
    operations: [
        new Get(),
        new GetCollection()
    ]
)]
class Book
{
    // No POST, PUT, PATCH, DELETE
}
```

### Admin-Only Mutations

```php
#[ApiResource(
    operations: [
        new Get(),
        new GetCollection(),
        new Post(security: "is_granted('ROLE_ADMIN')"),
        new Put(security: "is_granted('ROLE_ADMIN')"),
        new Delete(security: "is_granted('ROLE_ADMIN')")
    ]
)]
```

### Owner-Based Access

```php
#[ApiResource(
    operations: [
        new Get(security: "is_granted('VIEW', object)"),
        new Put(security: "object.getOwner() == user"),
        new Delete(security: "object.getOwner() == user")
    ]
)]
class Book
{
    private ?User $owner = null;
}
```

### Soft Delete

```php
#[ApiResource(
    operations: [
        new Delete(
            processor: SoftDeleteProcessor::class
        )
    ]
)]
class Book
{
    private ?\DateTimeInterface $deletedAt = null;
}

class SoftDeleteProcessor implements ProcessorInterface
{
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        $data->setDeletedAt(new \DateTime());
        $this->entityManager->flush();

        return $data;
    }
}
```

### Async Processing

```php
class BookProcessor implements ProcessorInterface
{
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        // Dispatch async message
        $this->messageBus->dispatch(new ProcessBookMessage($data->getId()));

        return $data;
    }
}
```
