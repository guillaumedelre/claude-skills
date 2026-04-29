---
name: "symfony-7-4-serializer"
description: "Symfony 7.4 Serializer component reference for converting objects to/from arrays/JSON/XML/CSV/YAML and back. Use when serializing/deserializing data, defining normalization groups, customizing property names, handling polymorphic types, or building custom normalizers/encoders. Triggers on: Serializer, SerializerInterface, NormalizerInterface, DenormalizerInterface, Encoder, JsonEncoder, XmlEncoder, CsvEncoder, YamlEncoder, ObjectNormalizer, GetSetMethodNormalizer, PropertyNormalizer, AbstractNormalizer, AbstractObjectNormalizer, BackedEnumNormalizer, DateTimeNormalizer, UidNormalizer, JsonSerializableNormalizer, ArrayDenormalizer, UnwrappingDenormalizer, ProblemNormalizer, ConstraintViolationListNormalizer, FormErrorNormalizer, MimeMessageNormalizer, DataUriNormalizer, DateIntervalNormalizer, custom normalizer, denormalize, serialize, deserialize, #[Groups], #[Ignore], #[SerializedName], #[SerializedPath], #[Context], #[DiscriminatorMap], normalization context, circular references, name converters, MetadataAwareNameConverter, CamelCaseToSnakeCaseNameConverter, JSON-LD, JSON:API."
---

# Symfony 7.4 Serializer Component

GitHub: https://github.com/symfony/serializer
Docs: https://symfony.com/doc/7.4/serializer.html

## Installation

```bash
composer require symfony/serializer-pack    # Recommended (includes annotations + property-info)
# or
composer require symfony/serializer
```

## Quick Reference

### Basic Usage

```php
use Symfony\Component\Serializer\SerializerInterface;

class ProductController
{
    public function __construct(private SerializerInterface $serializer) {}

    public function show(Product $product): Response
    {
        $json = $this->serializer->serialize($product, 'json');
        return new JsonResponse($json, json: true);
    }

    public function create(Request $request): Response
    {
        $product = $this->serializer->deserialize(
            $request->getContent(),
            Product::class,
            'json'
        );
        return new JsonResponse(['id' => $product->id]);
    }
}
```

### Groups

```php
use Symfony\Component\Serializer\Attribute\Groups;

class Product
{
    #[Groups(['product:read', 'product:write'])]
    public string $name;

    #[Groups(['product:read'])]
    public int $stock;

    #[Groups(['product:write'])]
    public string $internalNote;
}

$json = $serializer->serialize($product, 'json', ['groups' => 'product:read']);
$serializer->deserialize($json, Product::class, 'json', ['groups' => ['product:write']]);
```

### Ignore / SerializedName / SerializedPath

```php
use Symfony\Component\Serializer\Attribute\Ignore;
use Symfony\Component\Serializer\Attribute\SerializedName;
use Symfony\Component\Serializer\Attribute\SerializedPath;

class User
{
    #[SerializedName('username')]
    public string $userIdentifier;

    #[Ignore]
    public string $internalToken;

    // From nested JSON: { "profile": { "city": "Paris" } }
    #[SerializedPath('[profile][city]')]
    public string $city;
}
```

### Context Per-property

```php
use Symfony\Component\Serializer\Attribute\Context;
use Symfony\Component\Serializer\Normalizer\DateTimeNormalizer;

class Event
{
    #[Context([DateTimeNormalizer::FORMAT_KEY => 'Y-m-d'])]
    public \DateTimeImmutable $date;

    #[Context(
        normalizationContext: [DateTimeNormalizer::FORMAT_KEY => \DateTime::ATOM],
        denormalizationContext: [DateTimeNormalizer::FORMAT_KEY => 'Y-m-d H:i:s']
    )]
    public \DateTimeImmutable $createdAt;
}
```

### Name Converter

```yaml
framework:
    serializer:
        name_converter: 'serializer.name_converter.camel_case_to_snake_case'
```

Or `MetadataAwareNameConverter` (default in 7.x) which honors `#[SerializedName]` first.

### DiscriminatorMap (Polymorphism)

```php
use Symfony\Component\Serializer\Attribute\DiscriminatorMap;

#[DiscriminatorMap(typeProperty: 'type', mapping: [
    'circle' => Circle::class,
    'square' => Square::class,
])]
abstract class Shape {}

class Circle extends Shape { public float $radius; }
class Square extends Shape { public float $side; }

$shape = $serializer->deserialize('{"type":"circle","radius":3}', Shape::class, 'json');
// $shape instanceof Circle === true
```

### Custom Normalizer

```php
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;

final class MoneyNormalizer implements NormalizerInterface
{
    public function normalize(mixed $object, ?string $format = null, array $context = []): array
    {
        return ['amount' => $object->amount, 'currency' => $object->currency];
    }

    public function supportsNormalization(mixed $data, ?string $format = null, array $context = []): bool
    {
        return $data instanceof Money;
    }

    public function getSupportedTypes(?string $format): array
    {
        return [Money::class => true];   // true = cacheable
    }
}
```

Tag the service with `serializer.normalizer` (auto-tagged with autoconfigure).

Priorities: built-in normalizers run last; custom defaults to priority 0. Use higher priority via `#[AutoconfigureTag('serializer.normalizer', priority: 200)]`.

### Encoders

| Encoder | Format key |
|---------|-----------|
| `JsonEncoder` | `json` |
| `XmlEncoder` | `xml` |
| `YamlEncoder` | `yaml` |
| `CsvEncoder` | `csv` |

Custom encoder:

```php
use Symfony\Component\Serializer\Encoder\EncoderInterface;
use Symfony\Component\Serializer\Encoder\DecoderInterface;

final class TomlEncoder implements EncoderInterface, DecoderInterface
{
    public function encode(mixed $data, string $format, array $context = []): string { /* ... */ }
    public function supportsEncoding(string $format): bool { return $format === 'toml'; }
    public function decode(string $data, string $format, array $context = []): mixed { /* ... */ }
    public function supportsDecoding(string $format): bool { return $format === 'toml'; }
}
```

### Circular References

```php
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;

$serializer->serialize($author, 'json', [
    AbstractNormalizer::CIRCULAR_REFERENCE_HANDLER => fn ($object) => $object->getId(),
]);
```

### Max Depth

```php
class Author
{
    #[MaxDepth(2)]
    public Author $parent;
}

$json = $serializer->serialize($author, 'json', [AbstractObjectNormalizer::ENABLE_MAX_DEPTH => true]);
```

### Partial Denormalization

```php
$user = $serializer->deserialize($json, User::class, 'json', [
    AbstractNormalizer::OBJECT_TO_POPULATE => $existingUser,
    AbstractObjectNormalizer::DEEP_OBJECT_TO_POPULATE => true,
]);
```

### Errors

Catch `PartialDenormalizationException` to collect all violations:

```php
use Symfony\Component\Serializer\Exception\PartialDenormalizationException;

try {
    $serializer->deserialize($json, User::class, 'json', [
        DenormalizerInterface::COLLECT_DENORMALIZATION_ERRORS => true,
    ]);
} catch (PartialDenormalizationException $e) {
    $errors = $e->getErrors();   // NotNormalizableValueException[]
}
```

## Built-in Normalizers (selected)

| Normalizer | Handles |
|------------|---------|
| `ObjectNormalizer` | Generic object normalization (uses property accessor) |
| `GetSetMethodNormalizer` | Public getters/setters |
| `PropertyNormalizer` | Public/protected/private properties |
| `DateTimeNormalizer` | `\DateTimeInterface` |
| `DateTimeZoneNormalizer` | `\DateTimeZone` |
| `DateIntervalNormalizer` | `\DateInterval` |
| `BackedEnumNormalizer` | PHP 8.1 backed enums |
| `UidNormalizer` | `Symfony\Component\Uid\AbstractUid` |
| `JsonSerializableNormalizer` | `\JsonSerializable` (last resort) |
| `DataUriNormalizer` | `data:` URIs |
| `ArrayDenormalizer` | Arrays (`Type[]`) |
| `UnwrappingDenormalizer` | `[unwrap_path]` context |
| `ProblemNormalizer` | RFC 7807 problem details |
| `ConstraintViolationListNormalizer` | Validator violations |
| `FormErrorNormalizer` | Form errors |
| `MimeMessageNormalizer` | Mailer Email |

## Full Documentation

For exhaustive normalizer options, encoder context keys, custom encoders, JSON-LD/JSON:API integration via API Platform, and advanced patterns: see [references/serializer.md](references/serializer.md).
