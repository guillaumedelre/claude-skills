# Symfony 7.4 Serializer - Complete Reference

## Table of Contents

- [Architecture](#architecture)
- [Built-in Normalizers](#built-in-normalizers)
- [Built-in Encoders](#built-in-encoders)
- [Context Keys Reference](#context-keys-reference)
- [Attributes](#attributes)
- [Name Converters](#name-converters)
- [Custom Normalizers](#custom-normalizers)
- [Discriminator Maps](#discriminator-maps)
- [Circular References / Max Depth](#circular-references--max-depth)
- [Partial Denormalization](#partial-denormalization)
- [Type Extractors](#type-extractors)
- [Caching](#caching)
- [Errors](#errors)

---

## Architecture

```
serialize($object, $format, $context)
    -> NormalizerInterface (object -> array)
    -> EncoderInterface (array -> string)

deserialize($string, $type, $format, $context)
    -> DecoderInterface (string -> array)
    -> DenormalizerInterface (array -> object)
```

The `Serializer` service is itself a `SerializerInterface`, `NormalizerInterface`, `DenormalizerInterface`, `EncoderInterface`, `DecoderInterface`. It iterates registered normalizers/encoders by priority and supports.

## Built-in Normalizers

### ObjectNormalizer (default)

Uses `PropertyAccessor` to read/write public properties or get/set methods. Honors `#[SerializedName]`, `#[SerializedPath]`, `#[Groups]`, `#[Ignore]`, `#[Context]`, `#[MaxDepth]`, `#[DiscriminatorMap]`.

### PropertyNormalizer

Reads/writes properties directly (public/protected/private if accessible). Useful for value objects with private state.

### GetSetMethodNormalizer

Strict getter/setter normalizer. Skips properties without `get`/`set` methods.

### DateTimeNormalizer

Context keys:
- `DateTimeNormalizer::FORMAT_KEY` (default `\DateTime::RFC3339`).
- `DateTimeNormalizer::TIMEZONE_KEY` (`\DateTimeZone` or string).
- `DateTimeNormalizer::CAST_KEY` (`'string'`/`'int'`/`'float'`/`'array'` to coerce on denormalization).

### DateIntervalNormalizer

`DateIntervalNormalizer::FORMAT_KEY` defaults to `'P%yY%mM%dDT%hH%iM%sS'`.

### BackedEnumNormalizer

Handles PHP 8.1 backed enums. Throws `NotNormalizableValueException` on invalid value (catch with `COLLECT_DENORMALIZATION_ERRORS`).

### UidNormalizer

`UidNormalizer::NORMALIZATION_FORMAT_KEY`: `'canonical'` (default), `'base58'`, `'base32'`, `'rfc4122'`.

### ArrayDenormalizer

Allows `User[]` or `array<int, User>` as type:

```php
$serializer->deserialize($json, User::class.'[]', 'json');
```

### UnwrappingDenormalizer

```php
$json = '{"data":{"name":"Alice"}}';
$user = $serializer->deserialize($json, User::class, 'json', ['unwrap_path' => '[data]']);
```

### JsonSerializableNormalizer

Calls `JsonSerializable::jsonSerialize()`. Registered with low priority; only kicks in when nothing else handles the object.

### ConstraintViolationListNormalizer

Produces RFC 7807 `application/problem+json` body or simpler legacy:

```php
$serializer->normalize($violations, 'json', ['type' => 'https://example.com/errors']);
```

### ProblemNormalizer

Normalizes `FlattenException` (HTTP errors) into `application/problem+json`.

### MimeMessageNormalizer

Used by Mailer to serialize/deserialize `Email` objects (e.g. for queue transport).

## Built-in Encoders

### JsonEncoder

Context keys:
- `JsonEncoder::OPTIONS` (bitmask of `JSON_*` flags).

```php
$json = $serializer->serialize($obj, 'json', [
    JsonEncoder::OPTIONS => \JSON_UNESCAPED_UNICODE | \JSON_PRETTY_PRINT,
]);
```

### XmlEncoder

Context keys:
- `XmlEncoder::ROOT_NODE_NAME` (default `'response'`).
- `XmlEncoder::ENCODING`.
- `XmlEncoder::FORMAT_OUTPUT`.
- `XmlEncoder::AS_COLLECTION`.

```xml
<response>
    <name>Alice</name>
</response>
```

### CsvEncoder

Context:
- `CsvEncoder::DELIMITER_KEY` (default `,`).
- `CsvEncoder::ENCLOSURE_KEY`.
- `CsvEncoder::ESCAPE_CHAR_KEY`.
- `CsvEncoder::HEADERS_KEY`.
- `CsvEncoder::NO_HEADERS_KEY`.
- `CsvEncoder::AS_COLLECTION_KEY`.
- `CsvEncoder::OUTPUT_UTF8_BOM_KEY`.

### YamlEncoder

Context:
- `YamlEncoder::PRESERVE_EMPTY_OBJECTS`.
- `YamlEncoder::YAML_INLINE`.
- `YamlEncoder::YAML_INDENT`.
- `YamlEncoder::YAML_FLAGS` (bitmask of `Yaml::DUMP_*` / `Yaml::PARSE_*`).

## Context Keys Reference

`AbstractNormalizer::*`:

| Key | Effect |
|-----|--------|
| `GROUPS` | Limit to listed groups |
| `OBJECT_TO_POPULATE` | Populate existing instance instead of `new` |
| `IGNORED_ATTRIBUTES` | Skip listed property names |
| `ATTRIBUTES` | Include only listed |
| `DEFAULT_CONSTRUCTOR_ARGUMENTS` | `['Class' => ['ctorArg' => 'default']]` |
| `CIRCULAR_REFERENCE_LIMIT` | Max revisits before invoking handler |
| `CIRCULAR_REFERENCE_HANDLER` | Callable to substitute |
| `CALLBACKS` | `['propName' => fn($value, $object, $attribute, $format, $context)]` |
| `ALLOW_EXTRA_ATTRIBUTES` | Throw `ExtraAttributesException` if `false` |
| `REQUIRE_ALL_PROPERTIES` | Force ctor arg checks |

`AbstractObjectNormalizer::*`:

| Key | Effect |
|-----|--------|
| `ENABLE_MAX_DEPTH` | Honor `#[MaxDepth]` |
| `MAX_DEPTH_HANDLER` | Callable when limit hit |
| `DEEP_OBJECT_TO_POPULATE` | Recurse `OBJECT_TO_POPULATE` into children |
| `SKIP_NULL_VALUES` | Drop nulls on normalize |
| `SKIP_UNINITIALIZED_VALUES` | Skip typed properties never initialized |
| `PRESERVE_EMPTY_OBJECTS` | `[]` -> `{}` rather than empty array |
| `DISABLE_TYPE_ENFORCEMENT` | Allow type mismatches (legacy behavior) |

`DenormalizerInterface::COLLECT_DENORMALIZATION_ERRORS` collects errors instead of throwing on first.

## Attributes

| Attribute | Targets | Notes |
|-----------|---------|-------|
| `#[Groups]` | property/method/class | Restrict by groups |
| `#[Ignore]` | property/method | Always skip |
| `#[SerializedName]` | property | External name |
| `#[SerializedPath]` | property | Read from nested array path on denormalize |
| `#[Context]` | property | Override context for that property; supports `normalizationContext`/`denormalizationContext`/`groups` |
| `#[MaxDepth]` | property | Honor with `ENABLE_MAX_DEPTH` |
| `#[DiscriminatorMap]` | class | Polymorphic dispatch |

## Name Converters

`MetadataAwareNameConverter` (default): looks at `#[SerializedName]`, then delegates to inner converter.

`CamelCaseToSnakeCaseNameConverter`:

```yaml
framework:
    serializer:
        name_converter: 'serializer.name_converter.camel_case_to_snake_case'
```

Custom:

```php
use Symfony\Component\Serializer\NameConverter\NameConverterInterface;

final class KebabCaseConverter implements NameConverterInterface
{
    public function normalize(string $propertyName): string
    {
        return strtolower(preg_replace('/(?<!^)[A-Z]/', '-$0', $propertyName));
    }
    public function denormalize(string $propertyName): string
    {
        return lcfirst(str_replace('-', '', ucwords($propertyName, '-')));
    }
}
```

Per-property override via `#[SerializedName]` short-circuits the converter.

## Custom Normalizers

```php
use Symfony\Component\Serializer\Normalizer\NormalizerInterface;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;

#[AutoconfigureTag('serializer.normalizer', ['priority' => 200])]
final class MoneyNormalizer implements NormalizerInterface, DenormalizerInterface
{
    public function normalize(mixed $object, ?string $format = null, array $context = []): array
    {
        return ['amount' => $object->amount, 'currency' => $object->currency];
    }

    public function supportsNormalization(mixed $data, ?string $format = null, array $context = []): bool
    {
        return $data instanceof Money;
    }

    public function denormalize(mixed $data, string $type, ?string $format = null, array $context = []): Money
    {
        return new Money($data['amount'], $data['currency']);
    }

    public function supportsDenormalization(mixed $data, string $type, ?string $format = null, array $context = []): bool
    {
        return $type === Money::class;
    }

    public function getSupportedTypes(?string $format): array
    {
        return [Money::class => true];
    }
}
```

`getSupportedTypes()` is required since 7.0; returns `[Type => bool]` where `bool` indicates cacheability (true if `supports*()` is purely type-based).

Inject the inner serializer via `NormalizerAwareInterface` + trait if delegating:

```php
use Symfony\Component\Serializer\Normalizer\NormalizerAwareInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareTrait;

class MyNormalizer implements NormalizerInterface, NormalizerAwareInterface
{
    use NormalizerAwareTrait;
}
```

## Discriminator Maps

```php
#[DiscriminatorMap('type', [
    'circle' => Circle::class,
    'square' => Square::class,
])]
abstract class Shape {}
```

YAML config alternative:

```yaml
# config/serializer.yaml
App\Shape:
    discriminator_map:
        type_property: type
        mapping:
            circle: App\Circle
            square: App\Square
```

The discriminator property is **required** in the input/output. `default_type` (since 6.4) provides fallback:

```php
#[DiscriminatorMap('type', [...], defaultType: 'circle')]
```

## Circular References / Max Depth

```php
$json = $serializer->serialize($author, 'json', [
    AbstractNormalizer::CIRCULAR_REFERENCE_LIMIT => 1,
    AbstractNormalizer::CIRCULAR_REFERENCE_HANDLER => fn (Author $a) => $a->id,
]);
```

```php
class Author
{
    #[MaxDepth(2)]
    public ?Author $parent = null;
}

$json = $serializer->serialize($author, 'json', [
    AbstractObjectNormalizer::ENABLE_MAX_DEPTH => true,
]);
```

## Partial Denormalization

Update existing entity:

```php
$serializer->deserialize($json, User::class, 'json', [
    AbstractNormalizer::OBJECT_TO_POPULATE => $user,
    AbstractObjectNormalizer::DEEP_OBJECT_TO_POPULATE => true,
    'groups' => 'user:write',
]);
```

Validate after with the Validator component.

## Type Extractors

The serializer queries `PropertyTypeExtractorInterface` to discover argument types for denormalization. Register types via:

- PHPDoc on properties (PhpDocExtractor).
- PHP property types (ReflectionExtractor).
- Constructor argument types (ReflectionExtractor).

For collection types, prefer:

```php
class Order
{
    /** @var list<OrderItem> */
    public array $items;
}
```

Or PHP 8.x:

```php
public function __construct(
    /** @var OrderItem[] */
    public readonly array $items,
) {}
```

## Caching

The serializer caches metadata:

```yaml
framework:
    serializer:
        mapping:
            paths: ['%kernel.project_dir%/src/Entity']
        circular_reference_handler: App\Serializer\IdHandler
```

Cache stored in `var/cache/<env>/serialization.php`.

## Errors

| Exception | Cause |
|-----------|-------|
| `NotNormalizableValueException` | Value cannot be denormalized to expected type |
| `MissingConstructorArgumentsException` | Required ctor arg missing in input |
| `ExtraAttributesException` | Unknown attribute when `ALLOW_EXTRA_ATTRIBUTES => false` |
| `PartialDenormalizationException` | Aggregated errors when `COLLECT_DENORMALIZATION_ERRORS => true` |
| `CircularReferenceException` | Limit exceeded with no handler |
| `RuntimeException` | Generic (e.g. discriminator missing) |

Always catch `PartialDenormalizationException` for API endpoints to surface all input issues at once.
