---
name: "symfony-7-4-uid"
description: "Symfony 7.4 UID component reference for generating and manipulating unique identifiers. Use when working with UID, UUID, ULID, unique identifiers, UuidV4, UuidV7, AbstractUid, Doctrine UUID type, UuidFactory, Ulid, toBase32, toBase58, toRfc4122, NilUuid, NilUlid, UlidFactory, UuidType, UlidType, or any unique identifier-related Symfony code."
---

# Symfony 7.4 UID Component

## Overview

The Symfony UID component provides an object-oriented API to generate and represent UIDs (Unique Identifiers). It supports ULIDs and UUID versions 1-8, with implementations compatible with both 32-bit and 64-bit processors.

## Installation

```bash
composer require symfony/uid
```

## Quick Reference

### Generating UUIDs

```php
use Symfony\Component\Uid\Uuid;

// UUID v4 (random) - common choice
$uuid = Uuid::v4();

// UUID v7 (time-ordered, microsecond precision) - RECOMMENDED
$uuid = Uuid::v7();

// UUID v6 (reordered time-based, better for indexing)
$uuid = Uuid::v6();

// UUID v1 (timestamp + MAC address) - use v7 instead
$uuid = Uuid::v1();

// UUID v3/v5 (name-based with namespace)
$uuid = Uuid::v3(Uuid::fromString(Uuid::NAMESPACE_DNS), 'example.com');
$uuid = Uuid::v5(Uuid::fromString(Uuid::NAMESPACE_URL), 'https://example.com');
```

### Generating ULIDs

```php
use Symfony\Component\Uid\Ulid;

$ulid = new Ulid();  // e.g., 01AN4Z07BY79KA1307SR9X4MV3
```

### Creating from Strings

```php
// UUID
$uuid = Uuid::fromString('d9e7a184-5d5b-11ea-a62a-3499710062d0');
$uuid = Uuid::fromBinary($binaryData);
$uuid = Uuid::fromBase32('6SWYGR8QAV27NACAHMK5RG0RPG');
$uuid = Uuid::fromBase58('TuetYWNHhmuSQ3xPoVLv9M');

// ULID
$ulid = Ulid::fromString('01E439TP9XJZ9RPFH3T1PYBCR8');
$ulid = Ulid::fromBase58('1BKocMc5BnrVcuq2ti4Eqm');
```

### Converting UIDs

```php
$uuid->toBinary();   // 16-byte binary string
$uuid->toBase32();   // 26-character Base32 string
$uuid->toBase58();   // 22-character Base58 string
$uuid->toRfc4122();  // Standard UUID format (36 chars)
$uuid->toHex();      // Hexadecimal with 0x prefix
$uuid->toString();   // Same as toRfc4122() for UUIDs
```

### Validation

```php
Uuid::isValid($value);  // true or false
Ulid::isValid($value);  // true or false

// With specific format
Uuid::isValid($value, Uuid::FORMAT_RFC_4122);
Uuid::isValid($value, Uuid::FORMAT_BASE_32 | Uuid::FORMAT_BASE_58);
Uuid::isValid($value, Uuid::FORMAT_ALL);
```

### Doctrine Integration

```php
use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Types\UuidType;
use Symfony\Bridge\Doctrine\IdGenerator\UuidGenerator;
use Symfony\Component\Uid\Uuid;

#[ORM\Entity]
class Product
{
    #[ORM\Id]
    #[ORM\Column(type: UuidType::NAME, unique: true)]
    #[ORM\GeneratedValue(strategy: 'CUSTOM')]
    #[ORM\CustomIdGenerator('doctrine.uuid_generator')]
    private ?Uuid $id;

    #[ORM\Column(type: UuidType::NAME)]
    private Uuid $externalId;
}
```

### Factory Pattern

```php
use Symfony\Component\Uid\Factory\UuidFactory;
use Symfony\Component\Uid\Factory\UlidFactory;

class MyService
{
    public function __construct(
        private UuidFactory $uuidFactory,
        private UlidFactory $ulidFactory,
    ) {}

    public function generate(): void
    {
        $uuid = $this->uuidFactory->create();        // Default: UUIDv7
        $uuid = $this->uuidFactory->randomBased()->create();  // UUIDv4
        $ulid = $this->ulidFactory->create();
    }
}
```

### Working with Time-Based UIDs

```php
// Get timestamp from v1, v6, v7 UUIDs or ULIDs
$uuid = Uuid::v7();
$datetime = $uuid->getDateTime();  // DateTimeImmutable

// Convert between time-based versions
$v1 = Uuid::v1();
$v6 = $v1->toV6();
$v7 = $v1->toV7();
```

## Full Documentation
- Docs: https://symfony.com/doc/7.4/components/uid.html
- GitHub: https://github.com/symfony/uid
- **Full Reference**: See [references/uid.md](references/uid.md) for complete API documentation
