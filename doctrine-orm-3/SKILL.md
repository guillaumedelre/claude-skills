---
name: "doctrine-orm-3"
description: "Doctrine ORM 3 reference for database object-relational mapping with PHP and Symfony. Use when working with entities, repositories, QueryBuilder, DQL, associations, lifecycle events, migrations, fixtures, custom types, or any Doctrine ORM-related code. Triggers on: Doctrine ORM, Entity, Repository, QueryBuilder, DQL, EntityManager, OneToOne, OneToMany, ManyToOne, ManyToMany, lifecycle callbacks, PrePersist, PostPersist, PreUpdate, PostUpdate, migrations, fixtures, UnitOfWork, flush, persist, EntityRepository, findBy, findOneBy, custom repository methods, Criteria, associations, eager loading, lazy loading, fetch joins, batch processing."
---

# Doctrine ORM 3

GitHub: https://github.com/doctrine/orm
Docs: https://www.doctrine-project.org/projects/doctrine-orm/en/latest/

## Overview

Doctrine ORM 3 is a powerful object-relational mapper for PHP that provides transparent persistence for PHP objects. It uses the Data Mapper pattern and aims to completely separate your domain/business logic from the persistence layer.

## Quick Reference

### Basic Entity

```php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: BookRepository::class)]
#[ORM\Table(name: 'books')]
class Book
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private ?string $title = null;

    #[ORM\Column(type: 'text', nullable: true)]
    private ?string $description = null;

    #[ORM\Column(type: 'decimal', precision: 10, scale: 2)]
    private ?string $price = null;

    public function getId(): ?int
    {
        return $this->id;
    }

    // Getters and setters...
}
```

### Associations

```php
// ManyToOne
#[ORM\ManyToOne(targetEntity: Author::class, inversedBy: 'books')]
#[ORM\JoinColumn(nullable: false)]
private ?Author $author = null;

// OneToMany
#[ORM\OneToMany(targetEntity: Review::class, mappedBy: 'book', cascade: ['persist', 'remove'], orphanRemoval: true)]
private Collection $reviews;

// ManyToMany
#[ORM\ManyToMany(targetEntity: Tag::class, inversedBy: 'books')]
#[ORM\JoinTable(name: 'book_tags')]
private Collection $tags;

// OneToOne
#[ORM\OneToOne(targetEntity: BookDetails::class, mappedBy: 'book', cascade: ['persist', 'remove'])]
private ?BookDetails $details = null;
```

### Repository

```php
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
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

    public function findPublishedBooks(): array
    {
        return $this->findBy(['published' => true], ['createdAt' => 'DESC']);
    }
}
```

### QueryBuilder

```php
$qb = $entityManager->createQueryBuilder();
$books = $qb->select('b', 'a')
    ->from(Book::class, 'b')
    ->leftJoin('b.author', 'a')
    ->where('b.price < :maxPrice')
    ->andWhere('b.published = :published')
    ->setParameter('maxPrice', 50)
    ->setParameter('published', true)
    ->orderBy('b.createdAt', 'DESC')
    ->setMaxResults(10)
    ->getQuery()
    ->getResult();
```

### DQL (Doctrine Query Language)

```php
$dql = 'SELECT b, a FROM App\Entity\Book b JOIN b.author a WHERE b.price > :minPrice';
$query = $entityManager->createQuery($dql);
$query->setParameter('minPrice', 20);
$books = $query->getResult();
```

### Persist and Flush

```php
// Create
$book = new Book();
$book->setTitle('New Book');
$book->setAuthor($author);

$entityManager->persist($book);
$entityManager->flush();

// Update
$book->setTitle('Updated Title');
$entityManager->flush(); // No need to persist again

// Delete
$entityManager->remove($book);
$entityManager->flush();
```

### Lifecycle Callbacks

```php
#[ORM\Entity]
#[ORM\HasLifecycleCallbacks]
class Book
{
    #[ORM\Column(type: 'datetime')]
    private ?\DateTimeInterface $createdAt = null;

    #[ORM\PrePersist]
    public function onPrePersist(): void
    {
        $this->createdAt = new \DateTime();
    }

    #[ORM\PreUpdate]
    public function onPreUpdate(): void
    {
        $this->updatedAt = new \DateTime();
    }
}
```

### Native SQL

```php
$sql = 'SELECT * FROM books WHERE price > ? ORDER BY title';
$stmt = $entityManager->getConnection()->prepare($sql);
$results = $stmt->executeQuery([20])->fetchAllAssociative();
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
        $entityManager->clear(); // Detach entities from memory
    }
}
$entityManager->flush();
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
```

## Full Documentation

For complete details including advanced associations, custom types, events, filters, second-level cache, inheritance mapping, optimistic/pessimistic locking, query hints, result caching, hydration modes, and performance optimization strategies, see [references/doctrine-orm.md](references/doctrine-orm.md).
