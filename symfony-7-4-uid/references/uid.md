# Symfony 7.4 UID Component - Complete Reference

## Installation

```bash
composer require symfony/uid
```

## UUIDs (Universally Unique Identifiers)

UUIDs are 128-bit numbers represented as five groups of hexadecimal characters: `xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx`

### UUID Versions

#### UUID v1 (Time-based)

```php
use Symfony\Component\Uid\Uuid;

$uuid = Uuid::v1();
// Returns Symfony\Component\Uid\UuidV1 instance
```

Generates using timestamp and MAC address. **Not recommended** - use UuidV7 instead.

#### UUID v3 (Name-based, MD5)

```php
$namespace = Uuid::fromString(Uuid::NAMESPACE_OID);
$uuid = Uuid::v3($namespace, $name);
// Returns Symfony\Component\Uid\UuidV3 instance
```

**Predefined namespaces:**
- `Uuid::NAMESPACE_DNS`
- `Uuid::NAMESPACE_URL`
- `Uuid::NAMESPACE_OID`
- `Uuid::NAMESPACE_X500`

#### UUID v4 (Random)

```php
$uuid = Uuid::v4();
// Returns Symfony\Component\Uid\UuidV4 instance
```

#### UUID v5 (Name-based, SHA-1)

```php
$uuid = Uuid::v5($namespace, $name);
// Returns Symfony\Component\Uid\UuidV5 instance
```

#### UUID v6 (Reordered time-based)

```php
$uuid = Uuid::v6();
// Returns Symfony\Component\Uid\UuidV6 instance
```

Lexicographically sortable. Better for database indexing than v1.

#### UUID v7 (UNIX timestamp) - RECOMMENDED

```php
$uuid = Uuid::v7();
// Returns Symfony\Component\Uid\UuidV7 instance
```

Time-ordered with high-resolution Unix Epoch timestamp. **Recommended** over v1 and v6.

> **Note:** In Symfony 7.4, precision increased from milliseconds to microseconds.

#### UUID v8 (Custom)

```php
$uuid = Uuid::v8('d9e7a184-5d5b-11ea-a62a-3499710062d0');
// Returns Symfony\Component\Uid\UuidV8 instance
```

### Creating UUIDs from Other Formats

```php
$uuid = Uuid::fromString('d9e7a184-5d5b-11ea-a62a-3499710062d0');
$uuid = Uuid::fromBinary("\xd9\xe7\xa1\x84\x5d\x5b\x11\xea\xa6\x2a\x34\x99\x71\x00\x62\xd0");
$uuid = Uuid::fromBase32('6SWYGR8QAV27NACAHMK5RG0RPG');
$uuid = Uuid::fromBase58('TuetYWNHhmuSQ3xPoVLv9M');
$uuid = Uuid::fromRfc4122('d9e7a184-5d5b-11ea-a62a-3499710062d0');
```

### UUID Factory

```php
use Symfony\Component\Uid\Factory\UuidFactory;

class FooService
{
    public function __construct(private UuidFactory $uuidFactory) {}

    public function generate(): void
    {
        $uuid = $this->uuidFactory->create();
        $randomBased = $this->uuidFactory->randomBased()->create();
        $nameBased = $this->uuidFactory->nameBased($namespace)->create($name);
        $timeBased = $this->uuidFactory->timeBased($node)->create();
    }
}
```

**Default versions:**
- Default/time-based: UUIDv7
- Name-based: UUIDv5
- Random-based: UUIDv4

#### Factory Configuration

```yaml
# config/packages/uid.yaml
framework:
    uid:
        default_uuid_version: 6
        name_based_uuid_version: 3
        name_based_uuid_namespace: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
        time_based_uuid_version: 6
        time_based_uuid_node: 121212121212
```

### Converting UUIDs

```php
$uuid = Uuid::fromString('d9e7a184-5d5b-11ea-a62a-3499710062d0');

$uuid->toBinary();   // string(16) "\xd9\xe7\xa1\x84..."
$uuid->toBase32();   // string(26) "6SWYGR8QAV27NACAHMK5RG0RPG"
$uuid->toBase58();   // string(22) "TuetYWNHhmuSQ3xPoVLv9M"
$uuid->toRfc4122();  // string(36) "d9e7a184-5d5b-11ea-a62a-3499710062d0"
$uuid->toHex();      // string(34) "0xd9e7a1845d5b11eaa62a3499710062d0"
$uuid->toString();   // string(36) "d9e7a184-5d5b-11ea-a62a-3499710062d0"
```

### Converting UUID Versions

```php
$uuid = Uuid::v1();
$uuid->toV6(); // Returns UuidV6 instance
$uuid->toV7(); // Returns UuidV7 instance

$uuid = Uuid::v6();
$uuid->toV7(); // Returns UuidV7 instance
```

### Working with UUIDs

```php
use Symfony\Component\Uid\NilUuid;
use Symfony\Component\Uid\UuidV4;

$uuid = Uuid::v4();

// Check if nil (null)
$uuid instanceof NilUuid; // false

// Check type
$uuid instanceof UuidV4; // true

// Get datetime (v1, v6, v7 only)
$uuid = Uuid::v1();
$uuid->getDateTime(); // DateTimeImmutable instance

// Validate UUID
Uuid::isValid($uuid); // true or false

// Equality
$uuid1->equals($uuid2); // true or false

// Comparison (spaceship operator)
$uuid1->compare($uuid2); // int: 0, > 0, or < 0
```

### Validation with Format

```php
// Accept only RFC 4122 format
Uuid::isValid('90067ce4-f083-47d2-a0f4-c47359de0f97', Uuid::FORMAT_RFC_4122);

// Accept multiple formats
Uuid::isValid('3aJ7CNpDMfXPZrCsn4Cgey', Uuid::FORMAT_BASE_32 | Uuid::FORMAT_BASE_58);

// Accept any format
Uuid::isValid($value, Uuid::FORMAT_ALL);
```

**Available format constants:**
- `Uuid::FORMAT_BINARY`
- `Uuid::FORMAT_BASE_32`
- `Uuid::FORMAT_BASE_58`
- `Uuid::FORMAT_RFC_4122`
- `Uuid::FORMAT_RFC_9562` (equivalent to RFC_4122)
- `Uuid::FORMAT_ALL`

### Storing UUIDs in Databases

#### Doctrine Integration

```php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Types\UuidType;
use Symfony\Component\Uid\Uuid;

#[ORM\Entity]
class Product
{
    #[ORM\Column(type: UuidType::NAME)]
    private Uuid $someProperty;
}
```

#### Auto-generating UUID Primary Keys

```php
use Symfony\Bridge\Doctrine\IdGenerator\UuidGenerator;

class User
{
    #[ORM\Id]
    #[ORM\Column(type: UuidType::NAME, unique: true)]
    #[ORM\GeneratedValue(strategy: 'CUSTOM')]
    #[ORM\CustomIdGenerator('doctrine.uuid_generator')]
    private ?Uuid $id;
}
```

> **Warning:** Using UUIDs as primary keys is not recommended for performance. UUIDv6/v7 solve fragmentation but indexes remain slower and larger (128 bits vs 32/64 bits for integers).

#### Querying with UUIDs

```php
use Symfony\Bridge\Doctrine\Types\UuidType;
use Doctrine\DBAL\ParameterType;

class ProductRepository
{
    public function findUserProducts(User $user): array
    {
        return $this->createQueryBuilder('p')
            ->setParameter('user', $user->getUuid(), UuidType::NAME)
            // Alternative:
            // ->setParameter('user', $user->getUuid()->toBinary(), ParameterType::BINARY)
            ->getQuery()
            ->getResult();
    }
}
```

### Testing UUIDs

```php
use Symfony\Component\Uid\Factory\MockUuidFactory;
use Symfony\Component\Uid\UuidV4;

class UserServiceTest
{
    public function testCreateUserId(): void
    {
        $factory = new MockUuidFactory([
            UuidV4::fromString('11111111-1111-4111-8111-111111111111'),
            UuidV4::fromString('22222222-2222-4222-8222-222222222222'),
        ]);

        $service = new UserService($factory);

        $this->assertSame('11111111-1111-4111-8111-111111111111', $service->createUserId());
        $this->assertSame('22222222-2222-4222-8222-222222222222', $service->createUserId());
    }
}
```

> `MockUuidFactory` throws an exception if the sequence is exhausted or UUID types don't match.

## ULIDs (Universally Unique Lexicographically Sortable Identifier)

128-bit numbers represented as 26-character strings: `TTTTTTTTTTRRRRRRRRRRRRRRRR` (T = timestamp, R = random bits)

**Advantages over UUIDs:**
- Lexicographically sortable
- 26-character string (vs 36-character UUID)
- 128-bit compatible with UUID
- No index fragmentation

> **Note:** Multiple ULIDs generated in the same millisecond increment the random portion by one bit for monotonicity.

### Generating ULIDs

```php
use Symfony\Component\Uid\Ulid;

$ulid = new Ulid();  // e.g. 01AN4Z07BY79KA1307SR9X4MV3
```

### Creating ULIDs from Other Formats

```php
$ulid = Ulid::fromString('01E439TP9XJZ9RPFH3T1PYBCR8');
$ulid = Ulid::fromBinary("\x01\x71\x06\x9d\x59\x3d\x97\xd3\x8b\x3e\x23\xd0\x6d\xe5\xb3\x08");
$ulid = Ulid::fromBase32('01E439TP9XJZ9RPFH3T1PYBCR8');
$ulid = Ulid::fromBase58('1BKocMc5BnrVcuq2ti4Eqm');
$ulid = Ulid::fromRfc4122('0171069d-593d-97d3-8b3e-23d06de5b308');
```

### ULID Factory

```php
use Symfony\Component\Uid\Factory\UlidFactory;

class FooService
{
    public function __construct(private UlidFactory $ulidFactory) {}

    public function generate(): void
    {
        $ulid = $this->ulidFactory->create();
    }
}
```

### NilUlid

```php
use Symfony\Component\Uid\NilUlid;

$ulid = new NilUlid();
// Equivalent to: new Ulid('00000000000000000000000000')
```

### Converting ULIDs

```php
$ulid = Ulid::fromString('01E439TP9XJZ9RPFH3T1PYBCR8');

$ulid->toBinary();   // string(16) "\x01\x71\x06\x9d..."
$ulid->toBase32();   // string(26) "01E439TP9XJZ9RPFH3T1PYBCR8"
$ulid->toBase58();   // string(22) "1BKocMc5BnrVcuq2ti4Eqm"
$ulid->toRfc4122();  // string(36) "0171069d-593d-97d3-8b3e-23d06de5b308"
$ulid->toHex();      // string(34) "0x0171069d593d97d38b3e23d06de5b308"
```

### Working with ULIDs

```php
$ulid1 = new Ulid();
$ulid2 = new Ulid();

// Validate ULID
Ulid::isValid($ulidValue); // true or false

// Get datetime
$ulid1->getDateTime(); // DateTimeImmutable instance

// Equality
$ulid1->equals($ulid2); // true or false

// Comparison
$ulid1->compare($ulid2); // int: -1, 0, or 1
```

### Storing ULIDs in Databases

#### Doctrine Integration

```php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Types\UlidType;
use Symfony\Component\Uid\Ulid;

#[ORM\Entity]
class Product
{
    #[ORM\Column(type: UlidType::NAME)]
    private Ulid $someProperty;
}
```

#### Auto-generating ULID Primary Keys

```php
use Symfony\Bridge\Doctrine\IdGenerator\UlidGenerator;

class Product
{
    #[ORM\Id]
    #[ORM\Column(type: UlidType::NAME, unique: true)]
    #[ORM\GeneratedValue(strategy: 'CUSTOM')]
    #[ORM\CustomIdGenerator(class: UlidGenerator::class)]
    private ?Ulid $id;
}
```

#### Querying with ULIDs

```php
use Symfony\Bridge\Doctrine\Types\UlidType;

class ProductRepository
{
    public function findUserProducts(User $user): array
    {
        return $this->createQueryBuilder('p')
            ->setParameter('user', $user->getUlid(), UlidType::NAME)
            // Alternative:
            // ->setParameter('user', $user->getUlid()->toBinary())
            ->getQuery()
            ->getResult();
    }
}
```

## Console Commands

Enable commands via configuration:

```yaml
# config/services.yaml
services:
    Symfony\Component\Uid\Command\GenerateUlidCommand: ~
    Symfony\Component\Uid\Command\GenerateUuidCommand: ~
    Symfony\Component\Uid\Command\InspectUlidCommand: ~
    Symfony\Component\Uid\Command\InspectUuidCommand: ~
```

### Generate UUIDs

```bash
# Generate 1 random-based UUID
php bin/console uuid:generate --random-based

# Generate 1 time-based UUID with specific node
php bin/console uuid:generate --time-based=now --node=fb3502dc-137e-4849-8886-ac90d07f64a7

# Generate 2 UUIDs in base58 format
php bin/console uuid:generate --count=2 --format=base58
```

### Generate ULIDs

```bash
# Generate 1 ULID with current time
php bin/console ulid:generate

# Generate 1 ULID with specific timestamp
php bin/console ulid:generate --time="2021-02-02 14:00:00"

# Generate 2 ULIDs in RFC4122 format
php bin/console ulid:generate --count=2 --format=rfc4122
```

### Inspect UUIDs

```bash
php bin/console uuid:inspect d0a3a023-f515-4fe0-915c-575e63693998
```

Output:
```
 ---------------------- --------------------------------------
  Label                  Value
 ---------------------- --------------------------------------
  Version                4
  Canonical (RFC 4122)   d0a3a023-f515-4fe0-915c-575e63693998
  Base 58                SmHvuofV4GCF7QW543rDD9
  Base 32                6GMEG27X8N9ZG92Q2QBSHPJECR
 ---------------------- --------------------------------------
```

### Inspect ULIDs

```bash
php bin/console ulid:inspect 01F2TTCSYK1PDRH73Z41BN1C4X
```

Output:
```
 --------------------- --------------------------------------
  Label                 Value
 --------------------- --------------------------------------
  Canonical (Base 32)   01F2TTCSYK1PDRH73Z41BN1C4X
  Base 58               1BYGm16jS4kX3VYCysKKq6
  RFC 4122              0178b5a6-67d3-0d9b-889c-7f205750b09d
 --------------------- --------------------------------------
  Timestamp             2021-04-09 08:01:24.947
 --------------------- --------------------------------------
```

## Class Reference

### AbstractUid

Base class for all UID types.

**Methods:**
- `toBinary(): string` - Returns 16-byte binary representation
- `toBase32(): string` - Returns 26-character Base32 string
- `toBase58(): string` - Returns 22-character Base58 string
- `toRfc4122(): string` - Returns RFC 4122 format (36 characters)
- `toHex(): string` - Returns hexadecimal with 0x prefix
- `toString(): string` - Returns canonical string representation
- `equals(self $other): bool` - Checks equality
- `compare(self $other): int` - Compares two UIDs (-1, 0, 1)
- `getDateTime(): DateTimeImmutable` - Returns timestamp (time-based UIDs only)

**Static methods:**
- `fromString(string $uid): static` - Creates from any supported format
- `fromBinary(string $uid): static` - Creates from 16-byte binary
- `fromBase32(string $uid): static` - Creates from Base32 string
- `fromBase58(string $uid): static` - Creates from Base58 string
- `fromRfc4122(string $uid): static` - Creates from RFC 4122 format
- `isValid(string $uid, int $format = self::FORMAT_ALL): bool` - Validates UID

### Uuid

Factory class for creating UUIDs.

**Static factory methods:**
- `Uuid::v1(): UuidV1`
- `Uuid::v3(Uuid $namespace, string $name): UuidV3`
- `Uuid::v4(): UuidV4`
- `Uuid::v5(Uuid $namespace, string $name): UuidV5`
- `Uuid::v6(): UuidV6`
- `Uuid::v7(): UuidV7`
- `Uuid::v8(string $uuid): UuidV8`

**Namespace constants:**
- `Uuid::NAMESPACE_DNS`
- `Uuid::NAMESPACE_URL`
- `Uuid::NAMESPACE_OID`
- `Uuid::NAMESPACE_X500`

**Format constants:**
- `Uuid::FORMAT_BINARY`
- `Uuid::FORMAT_BASE_32`
- `Uuid::FORMAT_BASE_58`
- `Uuid::FORMAT_RFC_4122`
- `Uuid::FORMAT_RFC_9562`
- `Uuid::FORMAT_ALL`

### UuidV1, UuidV6, UuidV7

Time-based UUID classes implementing `TimeBasedUidInterface`.

**Additional methods:**
- `getDateTime(): DateTimeImmutable`
- `toV6(): UuidV6` (UuidV1 only)
- `toV7(): UuidV7` (UuidV1, UuidV6)

### Ulid

ULID implementation.

**Constructor:**
- `new Ulid(?string $ulid = null)` - Creates new ULID (generates if null)

**Static methods:**
- Same as AbstractUid plus:
- `Ulid::generate(?float $time = null): string` - Generates ULID string

### NilUuid and NilUlid

Special null/nil UIDs (all zeros).

```php
use Symfony\Component\Uid\NilUuid;
use Symfony\Component\Uid\NilUlid;

$nilUuid = new NilUuid();  // 00000000-0000-0000-0000-000000000000
$nilUlid = new NilUlid();  // 00000000000000000000000000
```

### UuidFactory

Service for generating UUIDs with dependency injection.

**Methods:**
- `create(?DateTimeInterface $time = null): UuidV6|UuidV7`
- `randomBased(): RandomBasedUuidFactory`
- `timeBased(?Uuid $node = null): TimeBasedUuidFactory`
- `nameBased(Uuid|string $namespace): NameBasedUuidFactory`

### UlidFactory

Service for generating ULIDs with dependency injection.

**Methods:**
- `create(?DateTimeInterface $time = null): Ulid`

### MockUuidFactory

Testing helper for deterministic UUID generation.

```php
use Symfony\Component\Uid\Factory\MockUuidFactory;

$factory = new MockUuidFactory([
    UuidV4::fromString('11111111-1111-4111-8111-111111111111'),
    UuidV4::fromString('22222222-2222-4222-8222-222222222222'),
]);
```

## Doctrine Types

### UuidType

Doctrine type for storing UUIDs.

```php
use Symfony\Bridge\Doctrine\Types\UuidType;

#[ORM\Column(type: UuidType::NAME)]  // 'uuid'
private Uuid $uuid;
```

### UlidType

Doctrine type for storing ULIDs.

```php
use Symfony\Bridge\Doctrine\Types\UlidType;

#[ORM\Column(type: UlidType::NAME)]  // 'ulid'
private Ulid $ulid;
```

## Resources

- **Documentation**: https://symfony.com/doc/7.4/components/uid.html
- **GitHub**: https://github.com/symfony/uid
- **License**: MIT
- **Used by**: 1.1 million projects
