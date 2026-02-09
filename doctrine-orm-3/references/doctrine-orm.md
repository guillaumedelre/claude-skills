# Doctrine ORM 3 Complete Reference

## Table of Contents

1. [Entities & Mapping](#entities--mapping)
2. [Associations](#associations)
3. [Repositories](#repositories)
4. [QueryBuilder](#querybuilder)
5. [DQL (Doctrine Query Language)](#dql-doctrine-query-language)
6. [Native SQL](#native-sql)
7. [Entity Manager](#entity-manager)
8. [Lifecycle Events](#lifecycle-events)
9. [Custom Types](#custom-types)
10. [Migrations](#migrations)
11. [Fixtures](#fixtures)
12. [Filters](#filters)
13. [Second Level Cache](#second-level-cache)
14. [Inheritance Mapping](#inheritance-mapping)
15. [Optimistic & Pessimistic Locking](#optimistic--pessimistic-locking)
16. [Performance Optimization](#performance-optimization)
17. [Best Practices](#best-practices)

---

## Entities & Mapping

### Basic Entity Mapping

```php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: BookRepository::class)]
#[ORM\Table(name: 'books')]
#[ORM\Index(name: 'title_idx', columns: ['title'])]
#[ORM\UniqueConstraint(name: 'isbn_unique', columns: ['isbn'])]
class Book
{
    #[ORM\Id]
    #[ORM\GeneratedValue(strategy: 'AUTO')]
    #[ORM\Column(type: 'integer')]
    private ?int $id = null;

    #[ORM\Column(type: 'string', length: 255)]
    private string $title;

    #[ORM\Column(type: 'text', nullable: true)]
    private ?string $description = null;

    #[ORM\Column(type: 'decimal', precision: 10, scale: 2)]
    private string $price;

    #[ORM\Column(type: 'boolean', options: ['default' => false])]
    private bool $published = false;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;
}
```

### Column Types

```php
// String types
#[ORM\Column(type: 'string', length: 255)]  // VARCHAR(255)
#[ORM\Column(type: 'text')]                 // TEXT

// Numeric types
#[ORM\Column(type: 'integer')]              // INT
#[ORM\Column(type: 'smallint')]             // SMALLINT
#[ORM\Column(type: 'bigint')]               // BIGINT
#[ORM\Column(type: 'decimal', precision: 10, scale: 2)]  // DECIMAL(10,2)
#[ORM\Column(type: 'float')]                // FLOAT

// Date/Time types
#[ORM\Column(type: 'datetime')]             // DATETIME (mutable)
#[ORM\Column(type: 'datetime_immutable')]   // DATETIME (immutable)
#[ORM\Column(type: 'date')]                 // DATE
#[ORM\Column(type: 'date_immutable')]       // DATE (immutable)
#[ORM\Column(type: 'time')]                 // TIME
#[ORM\Column(type: 'time_immutable')]       // TIME (immutable)

// Boolean
#[ORM\Column(type: 'boolean')]              // TINYINT(1)

// Binary
#[ORM\Column(type: 'blob')]                 // BLOB

// Array/Object
#[ORM\Column(type: 'array')]                // Serialized array
#[ORM\Column(type: 'json')]                 // JSON
#[ORM\Column(type: 'simple_array')]         // CSV string

// GUID
#[ORM\Column(type: 'guid')]                 // GUID/UUID
```

### ID Generation Strategies

```php
// AUTO - Let Doctrine choose (default)
#[ORM\Id]
#[ORM\GeneratedValue(strategy: 'AUTO')]
#[ORM\Column]
private ?int $id = null;

// IDENTITY - Auto-increment (MySQL AUTO_INCREMENT, PostgreSQL SERIAL)
#[ORM\GeneratedValue(strategy: 'IDENTITY')]

// SEQUENCE - Database sequence (PostgreSQL, Oracle)
#[ORM\GeneratedValue(strategy: 'SEQUENCE')]
#[ORM\SequenceGenerator(sequenceName: 'book_seq', initialValue: 1, allocationSize: 100)]

// UUID
#[ORM\Id]
#[ORM\Column(type: 'guid')]
#[ORM\GeneratedValue(strategy: 'UUID')]
private ?string $id = null;

// Custom generator
#[ORM\GeneratedValue(strategy: 'CUSTOM')]
#[ORM\CustomIdGenerator(class: CustomIdGenerator::class)]

// NONE - Manually assigned
#[ORM\Id]
#[ORM\Column]
private ?int $id = null; // Must be set manually before persist
```

### Embeddables

```php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Embeddable]
class Address
{
    #[ORM\Column(length: 255)]
    private ?string $street = null;

    #[ORM\Column(length: 100)]
    private ?string $city = null;

    #[ORM\Column(length: 20)]
    private ?string $zipCode = null;

    #[ORM\Column(length: 100)]
    private ?string $country = null;

    // Getters and setters...
}

#[ORM\Entity]
class User
{
    #[ORM\Embedded(class: Address::class)]
    private Address $address;

    public function __construct()
    {
        $this->address = new Address();
    }
}
```

---

## Associations

### ManyToOne / OneToMany

```php
// Book.php (owning side)
#[ORM\Entity]
class Book
{
    #[ORM\ManyToOne(targetEntity: Author::class, inversedBy: 'books')]
    #[ORM\JoinColumn(nullable: false, onDelete: 'CASCADE')]
    private ?Author $author = null;
}

// Author.php (inverse side)
#[ORM\Entity]
class Author
{
    #[ORM\OneToMany(targetEntity: Book::class, mappedBy: 'author', cascade: ['persist', 'remove'], orphanRemoval: true)]
    private Collection $books;

    public function __construct()
    {
        $this->books = new ArrayCollection();
    }

    public function addBook(Book $book): self
    {
        if (!$this->books->contains($book)) {
            $this->books->add($book);
            $book->setAuthor($this);
        }
        return $this;
    }

    public function removeBook(Book $book): self
    {
        if ($this->books->removeElement($book)) {
            if ($book->getAuthor() === $this) {
                $book->setAuthor(null);
            }
        }
        return $this;
    }
}
```

### ManyToMany

```php
// Book.php (owning side)
#[ORM\Entity]
class Book
{
    #[ORM\ManyToMany(targetEntity: Tag::class, inversedBy: 'books')]
    #[ORM\JoinTable(name: 'book_tags')]
    private Collection $tags;

    public function __construct()
    {
        $this->tags = new ArrayCollection();
    }

    public function addTag(Tag $tag): self
    {
        if (!$this->tags->contains($tag)) {
            $this->tags->add($tag);
        }
        return $this;
    }
}

// Tag.php (inverse side)
#[ORM\Entity]
class Tag
{
    #[ORM\ManyToMany(targetEntity: Book::class, mappedBy: 'tags')]
    private Collection $books;

    public function __construct()
    {
        $this->books = new ArrayCollection();
    }
}
```

### ManyToMany with Extra Fields

```php
// BookAuthor.php (join entity)
#[ORM\Entity]
#[ORM\Table(name: 'book_authors')]
class BookAuthor
{
    #[ORM\Id]
    #[ORM\ManyToOne(targetEntity: Book::class, inversedBy: 'bookAuthors')]
    private Book $book;

    #[ORM\Id]
    #[ORM\ManyToOne(targetEntity: Author::class, inversedBy: 'bookAuthors')]
    private Author $author;

    #[ORM\Column(type: 'string', length: 50)]
    private string $role; // 'main', 'co-author', 'contributor'

    #[ORM\Column(type: 'integer')]
    private int $position;
}

// Book.php
#[ORM\OneToMany(targetEntity: BookAuthor::class, mappedBy: 'book', cascade: ['persist', 'remove'])]
private Collection $bookAuthors;

// Author.php
#[ORM\OneToMany(targetEntity: BookAuthor::class, mappedBy: 'author')]
private Collection $bookAuthors;
```

### OneToOne

```php
// Book.php (owning side)
#[ORM\Entity]
class Book
{
    #[ORM\OneToOne(targetEntity: BookDetails::class, cascade: ['persist', 'remove'])]
    #[ORM\JoinColumn(nullable: false)]
    private ?BookDetails $details = null;
}

// BookDetails.php (inverse side)
#[ORM\Entity]
class BookDetails
{
    #[ORM\Id]
    #[ORM\OneToOne(targetEntity: Book::class, mappedBy: 'details')]
    private ?Book $book = null;
}
```

### Fetch Modes

```php
// LAZY - Load when accessed (default)
#[ORM\ManyToOne(targetEntity: Author::class, fetch: 'LAZY')]
private ?Author $author = null;

// EAGER - Always load with entity
#[ORM\ManyToOne(targetEntity: Author::class, fetch: 'EAGER')]
private ?Author $author = null;

// EXTRA_LAZY - Don't load collection, query on access
#[ORM\OneToMany(targetEntity: Review::class, mappedBy: 'book', fetch: 'EXTRA_LAZY')]
private Collection $reviews;
```

### Cascade Operations

```php
#[ORM\OneToMany(
    targetEntity: Review::class,
    mappedBy: 'book',
    cascade: ['persist', 'remove'],  // persist, remove, merge, detach, refresh, all
    orphanRemoval: true              // Delete orphaned entities
)]
private Collection $reviews;
```

---

## Repositories

### Custom Repository

```php
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\ORM\QueryBuilder;
use Doctrine\Persistence\ManagerRegistry;

class BookRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Book::class);
    }

    public function findByAuthor(Author $author): array
    {
        return $this->createQueryBuilder('b')
            ->andWhere('b.author = :author')
            ->setParameter('author', $author)
            ->orderBy('b.title', 'ASC')
            ->getQuery()
            ->getResult();
    }

    public function findPublishedBooksWithAuthors(): array
    {
        return $this->createQueryBuilder('b')
            ->select('b', 'a')
            ->leftJoin('b.author', 'a')
            ->where('b.published = :published')
            ->setParameter('published', true)
            ->getQuery()
            ->getResult();
    }

    public function searchByTitle(string $search): array
    {
        return $this->createQueryBuilder('b')
            ->where('b.title LIKE :search')
            ->setParameter('search', "%$search%")
            ->getQuery()
            ->getResult();
    }

    public function countByAuthor(Author $author): int
    {
        return $this->createQueryBuilder('b')
            ->select('COUNT(b.id)')
            ->where('b.author = :author')
            ->setParameter('author', $author)
            ->getQuery()
            ->getSingleScalarResult();
    }

    public function findOneByIsbn(string $isbn): ?Book
    {
        return $this->findOneBy(['isbn' => $isbn]);
    }
}
```

### Criteria API

```php
use Doctrine\Common\Collections\Criteria;

public function findExpensiveBooks(float $minPrice): array
{
    $criteria = Criteria::create()
        ->where(Criteria::expr()->gte('price', $minPrice))
        ->orderBy(['price' => Criteria::DESC])
        ->setMaxResults(10);

    return $this->matching($criteria)->toArray();
}
```

---

## QueryBuilder

### Basic QueryBuilder

```php
$qb = $entityManager->createQueryBuilder();
$books = $qb->select('b')
    ->from(Book::class, 'b')
    ->where('b.published = :published')
    ->setParameter('published', true)
    ->orderBy('b.createdAt', 'DESC')
    ->getQuery()
    ->getResult();
```

### Joins

```php
// Inner Join
$qb->select('b', 'a')
    ->from(Book::class, 'b')
    ->innerJoin('b.author', 'a')
    ->where('a.country = :country')
    ->setParameter('country', 'France');

// Left Join
$qb->select('b', 'a')
    ->from(Book::class, 'b')
    ->leftJoin('b.author', 'a');

// Join with condition
$qb->select('b', 'r')
    ->from(Book::class, 'b')
    ->leftJoin('b.reviews', 'r', 'WITH', 'r.rating >= :minRating')
    ->setParameter('minRating', 4);
```

### Where Clauses

```php
// Simple where
$qb->where('b.published = :published')
   ->setParameter('published', true);

// AND conditions
$qb->where('b.published = :published')
   ->andWhere('b.price < :maxPrice')
   ->setParameter('published', true)
   ->setParameter('maxPrice', 50);

// OR conditions
$qb->where('b.title LIKE :search')
   ->orWhere('b.description LIKE :search')
   ->setParameter('search', '%fantasy%');

// IN clause
$qb->where($qb->expr()->in('b.id', ':ids'))
   ->setParameter('ids', [1, 2, 3, 4, 5]);

// BETWEEN
$qb->where($qb->expr()->between('b.price', ':min', ':max'))
   ->setParameter('min', 10)
   ->setParameter('max', 50);

// IS NULL
$qb->where('b.deletedAt IS NULL');

// LIKE
$qb->where('b.title LIKE :search')
   ->setParameter('search', '%gatsby%');
```

### Subqueries

```php
$subQuery = $entityManager->createQueryBuilder()
    ->select('AVG(b2.price)')
    ->from(Book::class, 'b2');

$qb = $entityManager->createQueryBuilder()
    ->select('b')
    ->from(Book::class, 'b')
    ->where($qb->expr()->gt('b.price', '(' . $subQuery->getDQL() . ')'));
```

### Group By & Having

```php
$qb->select('a', 'COUNT(b.id) as bookCount')
    ->from(Author::class, 'a')
    ->leftJoin('a.books', 'b')
    ->groupBy('a.id')
    ->having('COUNT(b.id) > :minBooks')
    ->setParameter('minBooks', 5);
```

### Partial Objects

```php
$qb->select('partial b.{id, title, price}')
    ->from(Book::class, 'b');
```

### Result Types

```php
// Array of entities
$books = $qb->getQuery()->getResult();

// Single entity or null
$book = $qb->getQuery()->getOneOrNullResult();

// Single scalar value
$count = $qb->select('COUNT(b.id)')
    ->getQuery()
    ->getSingleScalarResult();

// Array result (not hydrated entities)
$results = $qb->getQuery()->getArrayResult();

// Single column as array
$titles = $qb->select('b.title')
    ->getQuery()
    ->getSingleColumnResult();
```

---

## DQL (Doctrine Query Language)

### Basic DQL

```php
$dql = 'SELECT b FROM App\Entity\Book b WHERE b.published = :published';
$query = $entityManager->createQuery($dql);
$query->setParameter('published', true);
$books = $query->getResult();
```

### DQL with Joins

```php
$dql = 'SELECT b, a FROM App\Entity\Book b JOIN b.author a WHERE a.country = :country';
$query = $entityManager->createQuery($dql);
$query->setParameter('country', 'France');
$books = $query->getResult();
```

### DQL Functions

```php
// String functions
$dql = 'SELECT b FROM App\Entity\Book b WHERE LOWER(b.title) LIKE :search';
$dql = 'SELECT SUBSTRING(b.title, 1, 10) FROM App\Entity\Book b';
$dql = 'SELECT CONCAT(a.firstName, \' \', a.lastName) as fullName FROM App\Entity\Author a';

// Math functions
$dql = 'SELECT b FROM App\Entity\Book b WHERE ABS(b.price - :targetPrice) < 5';

// Date functions
$dql = 'SELECT b FROM App\Entity\Book b WHERE YEAR(b.createdAt) = :year';
$dql = 'SELECT b FROM App\Entity\Book b WHERE MONTH(b.createdAt) = :month';

// Aggregate functions
$dql = 'SELECT COUNT(b.id), AVG(b.price), MAX(b.price), MIN(b.price) FROM App\Entity\Book b';
```

### DQL UPDATE/DELETE

```php
// UPDATE
$dql = 'UPDATE App\Entity\Book b SET b.published = :published WHERE b.author = :author';
$query = $entityManager->createQuery($dql);
$query->setParameter('published', true);
$query->setParameter('author', $author);
$numUpdated = $query->execute();

// DELETE
$dql = 'DELETE App\Entity\Book b WHERE b.createdAt < :date';
$query = $entityManager->createQuery($dql);
$query->setParameter('date', new \DateTime('-1 year'));
$numDeleted = $query->execute();
```

---

## Native SQL

### Basic Native Query

```php
$sql = 'SELECT * FROM books WHERE price > ? ORDER BY title';
$stmt = $entityManager->getConnection()->prepare($sql);
$results = $stmt->executeQuery([20])->fetchAllAssociative();
```

### Native SQL with Result Mapping

```php
$rsm = new ResultSetMapping();
$rsm->addEntityResult(Book::class, 'b');
$rsm->addFieldResult('b', 'id', 'id');
$rsm->addFieldResult('b', 'title', 'title');
$rsm->addFieldResult('b', 'price', 'price');

$sql = 'SELECT id, title, price FROM books WHERE price > ?';
$query = $entityManager->createNativeQuery($sql, $rsm);
$query->setParameter(1, 20);
$books = $query->getResult();
```

---

## Entity Manager

### Basic Operations

```php
// Persist (mark for insert)
$book = new Book();
$entityManager->persist($book);

// Flush (execute SQL)
$entityManager->flush();

// Find by primary key
$book = $entityManager->find(Book::class, $id);

// Refresh from database
$entityManager->refresh($book);

// Detach from UnitOfWork
$entityManager->detach($book);

// Clear all entities
$entityManager->clear();

// Remove (mark for delete)
$entityManager->remove($book);
$entityManager->flush();

// Check if entity is managed
$isManaged = $entityManager->contains($book);
```

### Transactions

```php
$entityManager->beginTransaction();

try {
    $book = new Book();
    $entityManager->persist($book);
    $entityManager->flush();

    // Other operations...

    $entityManager->commit();
} catch (\Exception $e) {
    $entityManager->rollback();
    throw $e;
}

// Alternative with transactional wrapper
$entityManager->transactional(function($em) use ($book) {
    $em->persist($book);
    // Automatically commits or rolls back
});
```

### Batch Processing

```php
$batchSize = 100;
for ($i = 0; $i < 10000; $i++) {
    $book = new Book();
    $book->setTitle("Book $i");
    $entityManager->persist($book);

    if (($i % $batchSize) === 0) {
        $entityManager->flush();
        $entityManager->clear(); // Free memory
    }
}
$entityManager->flush();
$entityManager->clear();
```

---

## Lifecycle Events

### Lifecycle Callbacks

```php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\HasLifecycleCallbacks]
class Book
{
    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $updatedAt;

    #[ORM\PrePersist]
    public function onPrePersist(): void
    {
        $this->createdAt = new \DateTimeImmutable();
        $this->updatedAt = new \DateTimeImmutable();
    }

    #[ORM\PreUpdate]
    public function onPreUpdate(): void
    {
        $this->updatedAt = new \DateTimeImmutable();
    }

    #[ORM\PostLoad]
    public function onPostLoad(): void
    {
        // Called after entity is loaded from database
    }

    #[ORM\PreRemove]
    public function onPreRemove(): void
    {
        // Called before entity is removed
    }

    #[ORM\PostPersist]
    public function onPostPersist(): void
    {
        // Called after entity is inserted
    }

    #[ORM\PostUpdate]
    public function onPostUpdate(): void
    {
        // Called after entity is updated
    }

    #[ORM\PostRemove]
    public function onPostRemove(): void
    {
        // Called after entity is deleted
    }
}
```

### Entity Listeners

```php
// BookListener.php
use Doctrine\ORM\Event\PrePersistEventArgs;
use Doctrine\ORM\Event\PreUpdateEventArgs;

#[ORM\EntityListener]
class BookListener
{
    #[ORM\PrePersist]
    public function prePersist(Book $book, PrePersistEventArgs $event): void
    {
        $book->setCreatedAt(new \DateTimeImmutable());
    }

    #[ORM\PreUpdate]
    public function preUpdate(Book $book, PreUpdateEventArgs $event): void
    {
        $book->setUpdatedAt(new \DateTimeImmutable());
    }
}

// Book.php
#[ORM\Entity]
#[ORM\EntityListeners([BookListener::class])]
class Book
{
    // ...
}
```

### Event Subscribers

```php
use Doctrine\Common\EventSubscriber;
use Doctrine\ORM\Events;
use Doctrine\ORM\Event\PrePersistEventArgs;
use Doctrine\ORM\Event\PreUpdateEventArgs;

class TimestampableSubscriber implements EventSubscriber
{
    public function getSubscribedEvents(): array
    {
        return [
            Events::prePersist,
            Events::preUpdate,
        ];
    }

    public function prePersist(PrePersistEventArgs $args): void
    {
        $entity = $args->getObject();

        if ($entity instanceof TimestampableInterface) {
            $entity->setCreatedAt(new \DateTimeImmutable());
        }
    }

    public function preUpdate(PreUpdateEventArgs $args): void
    {
        $entity = $args->getObject();

        if ($entity instanceof TimestampableInterface) {
            $entity->setUpdatedAt(new \DateTimeImmutable());
        }
    }
}
```

---

## Custom Types

### Custom Type Example

```php
use Doctrine\DBAL\Types\Type;
use Doctrine\DBAL\Platforms\AbstractPlatform;

class MoneyType extends Type
{
    public const NAME = 'money';

    public function getSQLDeclaration(array $column, AbstractPlatform $platform): string
    {
        return 'DECIMAL(10,2)';
    }

    public function convertToPHPValue($value, AbstractPlatform $platform): ?Money
    {
        if ($value === null) {
            return null;
        }

        return new Money($value);
    }

    public function convertToDatabaseValue($value, AbstractPlatform $platform): ?string
    {
        if ($value === null) {
            return null;
        }

        if (!$value instanceof Money) {
            throw new \InvalidArgumentException('Expected Money object');
        }

        return (string) $value->getAmount();
    }

    public function getName(): string
    {
        return self::NAME;
    }
}

// Register type
Type::addType('money', MoneyType::class);

// Use in entity
#[ORM\Column(type: 'money')]
private ?Money $price = null;
```

---

## Migrations

### Generate Migration

```bash
php bin/console make:migration
php bin/console doctrine:migrations:migrate
```

### Migration Example

```php
use Doctrine\DBAL\Schema\Schema;
use Doctrine\Migrations\AbstractMigration;

final class Version20250101000000 extends AbstractMigration
{
    public function getDescription(): string
    {
        return 'Add isbn column to books table';
    }

    public function up(Schema $schema): void
    {
        $this->addSql('ALTER TABLE books ADD isbn VARCHAR(13) DEFAULT NULL');
        $this->addSql('CREATE UNIQUE INDEX isbn_unique ON books (isbn)');
    }

    public function down(Schema $schema): void
    {
        $this->addSql('DROP INDEX isbn_unique ON books');
        $this->addSql('ALTER TABLE books DROP isbn');
    }
}
```

---

## Fixtures

### Fixture Example

```php
use Doctrine\Bundle\FixturesBundle\Fixture;
use Doctrine\Persistence\ObjectManager;

class BookFixtures extends Fixture
{
    public function load(ObjectManager $manager): void
    {
        for ($i = 0; $i < 10; $i++) {
            $book = new Book();
            $book->setTitle("Book $i");
            $book->setPrice(rand(10, 100));
            $manager->persist($book);
        }

        $manager->flush();
    }
}

// Load fixtures
// php bin/console doctrine:fixtures:load
```

---

## Filters

### SQL Filter

```php
use Doctrine\ORM\Mapping\ClassMetadata;
use Doctrine\ORM\Query\Filter\SQLFilter;

class SoftDeleteFilter extends SQLFilter
{
    public function addFilterConstraint(ClassMetadata $targetEntity, $targetTableAlias): string
    {
        if (!$targetEntity->hasField('deletedAt')) {
            return '';
        }

        return $targetTableAlias . '.deleted_at IS NULL';
    }
}

// Enable filter
$entityManager->getFilters()->enable('soft_delete');

// Disable filter
$entityManager->getFilters()->disable('soft_delete');
```

---

## Second Level Cache

### Configuration

```yaml
# config/packages/doctrine.yaml
doctrine:
    orm:
        second_level_cache:
            enabled: true
            regions:
                books_region:
                    lifetime: 3600
```

### Entity Cache

```php
#[ORM\Entity]
#[ORM\Cache(usage: 'READ_WRITE', region: 'books_region')]
class Book
{
    // Cached entity
}
```

---

## Inheritance Mapping

### Single Table Inheritance

```php
#[ORM\Entity]
#[ORM\InheritanceType('SINGLE_TABLE')]
#[ORM\DiscriminatorColumn(name: 'type', type: 'string')]
#[ORM\DiscriminatorMap(['book' => Book::class, 'ebook' => Ebook::class])]
abstract class Product
{
    #[ORM\Id]
    #[ORM\Column]
    private ?int $id = null;
}

#[ORM\Entity]
class Book extends Product
{
    #[ORM\Column]
    private ?int $pages = null;
}

#[ORM\Entity]
class Ebook extends Product
{
    #[ORM\Column]
    private ?string $fileFormat = null;
}
```

---

## Optimistic & Pessimistic Locking

### Optimistic Locking

```php
#[ORM\Entity]
class Book
{
    #[ORM\Version]
    #[ORM\Column(type: 'integer')]
    private int $version;
}
```

### Pessimistic Locking

```php
$book = $entityManager->find(
    Book::class,
    $id,
    \Doctrine\DBAL\LockMode::PESSIMISTIC_WRITE
);
```

---

## Performance Optimization

### Fetch Joins

```php
// BAD: N+1 queries
$books = $bookRepository->findAll();
foreach ($books as $book) {
    echo $book->getAuthor()->getName(); // Lazy load for each book
}

// GOOD: Single query with join
$books = $bookRepository->createQueryBuilder('b')
    ->select('b', 'a')
    ->leftJoin('b.author', 'a')
    ->getQuery()
    ->getResult();
```

### Partial Objects

```php
$books = $bookRepository->createQueryBuilder('b')
    ->select('partial b.{id, title}')
    ->getQuery()
    ->getResult();
```

### Query Result Cache

```php
$query = $entityManager->createQuery('SELECT b FROM App\Entity\Book b');
$query->setResultCacheLifetime(3600);
$query->useResultCache(true);
$books = $query->getResult();
```

---

## Best Practices

1. **Use DTOs for read-only data** - Don't hydrate entities if you don't need to modify them
2. **Batch processing** - Use flush() and clear() in loops to prevent memory issues
3. **Fetch joins** - Avoid N+1 queries by using joins
4. **Indexes** - Add indexes to frequently queried columns
5. **Lazy loading** - Use EXTRA_LAZY for large collections
6. **Transactions** - Group operations in transactions for consistency
7. **Repository methods** - Encapsulate queries in repository methods
8. **Immutable dates** - Use DateTimeImmutable instead of DateTime
9. **Avoid entities in forms** - Use DTOs to prevent mass assignment vulnerabilities
10. **Monitor queries** - Use Symfony profiler to identify slow queries
