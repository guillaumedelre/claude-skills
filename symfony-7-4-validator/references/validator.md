# Symfony 7.4 Validator - Complete Reference

## Table of Contents

- [Installation](#installation)
- [Basic Usage](#basic-usage)
- [Constraint Configuration](#constraint-configuration)
- [Constraint Targets](#constraint-targets)
- [Built-in Constraints](#built-in-constraints)
- [Validation Groups](#validation-groups)
- [Group Sequences](#group-sequences)
- [Custom Constraints](#custom-constraints)
- [Compound Constraints](#compound-constraints)
- [Constraint Violations](#constraint-violations)
- [Testing Constraints](#testing-constraints)
- [Extending Validation](#extending-validation)
- [Debugging](#debugging)

---

## Installation

```bash
composer require symfony/validator
```

## Basic Usage

### Creating a Validator Instance

```php
use Symfony\Component\Validator\Validation;

$validator = Validation::createValidator();
```

### Using ValidatorInterface (Recommended in Symfony Applications)

```php
use Symfony\Component\Validator\Validator\ValidatorInterface;

class AuthorController
{
    public function create(ValidatorInterface $validator): Response
    {
        $author = new Author();
        $author->setName('');

        $errors = $validator->validate($author);

        if (count($errors) > 0) {
            $errorsString = (string) $errors;
            return new Response($errorsString, 400);
        }

        return new Response('Valid author!');
    }
}
```

### Validating Raw Values

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Validation;

$validator = Validation::createValidator();

$violations = $validator->validate('Bernhard', [
    new Assert\Length(min: 10),
    new Assert\NotBlank(),
]);

if (0 !== count($violations)) {
    foreach ($violations as $violation) {
        echo $violation->getMessage() . "\n";
    }
}
```

### Validation Callables

```php
use Symfony\Component\Validator\Validation;
use Symfony\Component\Validator\Constraints as Assert;

// Throws ValidationFailedException on constraint violations
$validateEmail = Validation::createCallable(new Assert\Email());
try {
    $validateEmail('invalid-email');
} catch (ValidationFailedException $e) {
    $violations = $e->getViolations();
}

// Returns false on constraint violations
$isValidEmail = Validation::createIsValidCallable(new Assert\Email());
if (!$isValidEmail($value)) {
    // Handle validation error
}
```

## Constraint Configuration

Constraints can be configured using **Attributes** (recommended), **YAML**, **XML**, or **PHP**.

### Attributes (Recommended)

```php
use Symfony\Component\Validator\Constraints as Assert;

class Author
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 3, max: 100)]
    private string $name;

    #[Assert\Choice(
        choices: ['fiction', 'non-fiction'],
        message: 'Choose a valid genre.',
    )]
    private string $genre;
}
```

### YAML

```yaml
# config/validator/validation.yaml
App\Entity\Author:
    properties:
        name:
            - NotBlank: ~
            - Length: { min: 3, max: 100 }
        genre:
            - Choice:
                choices: [fiction, non-fiction]
                message: Choose a valid genre.
```

### XML

```xml
<!-- config/validator/validation.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<constraint-mapping xmlns="http://symfony.com/schema/dic/constraint-mapping">
    <class name="App\Entity\Author">
        <property name="name">
            <constraint name="NotBlank"/>
            <constraint name="Length">
                <option name="min">3</option>
                <option name="max">100</option>
            </constraint>
        </property>
        <property name="genre">
            <constraint name="Choice">
                <option name="choices">
                    <value>fiction</value>
                    <value>non-fiction</value>
                </option>
                <option name="message">Choose a valid genre.</option>
            </constraint>
        </property>
    </class>
</constraint-mapping>
```

### PHP (loadValidatorMetadata)

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Mapping\ClassMetadata;

class Author
{
    private string $name;
    private string $genre;

    public static function loadValidatorMetadata(ClassMetadata $metadata): void
    {
        $metadata->addPropertyConstraint('name', new Assert\NotBlank());
        $metadata->addPropertyConstraint('name', new Assert\Length(min: 3, max: 100));
        $metadata->addPropertyConstraint('genre', new Assert\Choice(
            choices: ['fiction', 'non-fiction'],
            message: 'Choose a valid genre.',
        ));
    }
}
```

## Constraint Targets

### Properties

```php
class Author
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 3)]
    private string $firstName;

    #[Assert\Email]
    private string $email;
}
```

### Getters

Apply constraints to method return values (methods must start with `get`, `is`, or `has`):

```php
class Author
{
    private string $firstName;
    private string $password;

    #[Assert\IsTrue(message: 'The password cannot match your first name')]
    public function isPasswordSafe(): bool
    {
        return $this->firstName !== $this->password;
    }
}
```

**YAML getter configuration:**

```yaml
App\Entity\Author:
    getters:
        passwordSafe:
            - 'IsTrue': { message: 'The password cannot match your first name' }
```

### Classes

Class-level constraints validate entire objects:

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Context\ExecutionContextInterface;

#[Assert\Callback('validateAuthor')]
class Author
{
    public function validateAuthor(ExecutionContextInterface $context): void
    {
        // Custom validation logic
    }
}
```

## Built-in Constraints

### Basic Constraints

| Constraint | Description |
|------------|-------------|
| `Blank` | Value must be blank (empty string or null) |
| `NotBlank` | Value must not be blank |
| `NotNull` | Value must not be null |
| `IsNull` | Value must be null |
| `IsTrue` | Value must be true |
| `IsFalse` | Value must be false |
| `Type` | Value must be of a specific PHP type |

### String Constraints

| Constraint | Description |
|------------|-------------|
| `Email` | Valid email address |
| `Length` | String length within range |
| `Url` | Valid URL |
| `Regex` | Matches regex pattern |
| `Ip` | Valid IP address |
| `Cidr` | Valid CIDR notation |
| `Uuid` | Valid UUID format |
| `Ulid` | Valid ULID format |
| `Hostname` | Valid hostname |
| `Json` | Valid JSON string |
| `Yaml` | Valid YAML string |
| `Charset` | Valid character encoding |
| `CssColor` | Valid CSS color |
| `MacAddress` | Valid MAC address |
| `WordCount` | Word count within range |
| `NoSuspiciousCharacters` | No suspicious Unicode characters |
| `PasswordStrength` | Password meets strength requirements |
| `NotCompromisedPassword` | Password not in known breaches |
| `UserPassword` | Matches current user's password |
| `ExpressionSyntax` | Valid expression syntax |
| `Twig` | Valid Twig syntax |

### Comparison Constraints

| Constraint | Description |
|------------|-------------|
| `EqualTo` | Value equals specified value |
| `NotEqualTo` | Value does not equal specified value |
| `IdenticalTo` | Value is identical to specified value |
| `NotIdenticalTo` | Value is not identical to specified value |
| `LessThan` | Value is less than specified value |
| `LessThanOrEqual` | Value is less than or equal to specified value |
| `GreaterThan` | Value is greater than specified value |
| `GreaterThanOrEqual` | Value is greater than or equal to specified value |
| `Range` | Value is within a range |
| `DivisibleBy` | Value is divisible by specified number |
| `Unique` | All elements in collection are unique |

### Number Constraints

| Constraint | Description |
|------------|-------------|
| `Positive` | Value must be positive |
| `PositiveOrZero` | Value must be positive or zero |
| `Negative` | Value must be negative |
| `NegativeOrZero` | Value must be negative or zero |

### Date Constraints

| Constraint | Description |
|------------|-------------|
| `Date` | Valid date (Y-m-d format) |
| `DateTime` | Valid datetime |
| `Time` | Valid time |
| `Timezone` | Valid timezone identifier |
| `Week` | Valid week format |

### Choice Constraints

| Constraint | Description |
|------------|-------------|
| `Choice` | Value is in allowed choices |
| `Country` | Valid country code |
| `Language` | Valid language code |
| `Locale` | Valid locale identifier |

### File Constraints

| Constraint | Description |
|------------|-------------|
| `File` | Valid file |
| `Image` | Valid image file |
| `Video` | Valid video file |

### Financial Constraints

| Constraint | Description |
|------------|-------------|
| `Iban` | Valid IBAN number |
| `Bic` | Valid BIC/SWIFT code |
| `CardScheme` | Valid credit card number |
| `Currency` | Valid currency code |
| `Luhn` | Passes Luhn algorithm |
| `Isbn` | Valid ISBN number |
| `Issn` | Valid ISSN number |
| `Isin` | Valid ISIN number |

### Collection/Object Constraints

| Constraint | Description |
|------------|-------------|
| `Count` | Collection has specific count |
| `All` | Apply constraints to all elements |
| `Collection` | Validate array/object structure |
| `Valid` | Cascade validation to nested objects |
| `Cascade` | Cascade validation without requiring #[Valid] |
| `Traverse` | Traverse and validate iterable |
| `AtLeastOneOf` | At least one constraint must pass |
| `Sequentially` | Apply constraints sequentially |

### Other Constraints

| Constraint | Description |
|------------|-------------|
| `Callback` | Custom validation via callback |
| `Expression` | Expression-based validation |
| `When` | Conditional constraint application |
| `Compound` | Create reusable constraint sets |
| `GroupSequence` | Define validation group order |

### Doctrine Constraints

| Constraint | Description |
|------------|-------------|
| `UniqueEntity` | Entity property is unique in database |
| `EnableAutoMapping` | Enable automatic Doctrine mapping |
| `DisableAutoMapping` | Disable automatic Doctrine mapping |

## Validation Groups

### Defining Groups

```php
use Symfony\Component\Validator\Constraints as Assert;

class User
{
    #[Assert\NotBlank(groups: ['registration'])]
    #[Assert\Email(groups: ['registration', 'profile'])]
    private string $email;

    #[Assert\NotBlank(groups: ['registration'])]
    #[Assert\Length(min: 8, groups: ['registration'])]
    private string $password;

    #[Assert\Length(min: 2)]  // Default group
    private string $city;
}
```

### Applying Groups During Validation

```php
// Validate only 'registration' group
$errors = $validator->validate($user, null, ['registration']);

// Validate multiple groups
$errors = $validator->validate($user, null, ['registration', 'profile']);

// Validate Default group (no groups specified)
$errors = $validator->validate($user);
```

### Built-in Groups

| Group | Description |
|-------|-------------|
| `Default` | Constraints with no explicit group |
| `{ClassName}` | Equivalent to Default for that class (e.g., `User`) |

### YAML Groups

```yaml
App\Entity\User:
    properties:
        email:
            - NotBlank: { groups: [registration] }
            - Email: { groups: [registration, profile] }
        password:
            - NotBlank: { groups: [registration] }
            - Length: { min: 8, groups: [registration] }
        city:
            - Length: { min: 2 }
```

## Group Sequences

Group sequences validate groups in order, stopping at the first failure.

### Static GroupSequence

```php
use Symfony\Component\Validator\Constraints as Assert;

#[Assert\GroupSequence(['User', 'Strict'])]
class User
{
    #[Assert\NotBlank]
    private string $username;

    #[Assert\NotBlank]
    private string $password;

    #[Assert\IsTrue(message: 'Password cannot match username', groups: ['Strict'])]
    public function isPasswordSafe(): bool
    {
        return $this->username !== $this->password;
    }
}
```

### Dynamic GroupSequenceProvider

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\GroupSequenceProviderInterface;

#[Assert\GroupSequenceProvider]
class User implements GroupSequenceProviderInterface
{
    #[Assert\NotBlank]
    private string $name;

    #[Assert\CardScheme(schemes: [Assert\CardScheme::VISA], groups: ['Premium'])]
    private string $creditCard;

    private bool $isPremium = false;

    public function getGroupSequence(): array
    {
        if ($this->isPremium) {
            return ['User', 'Premium'];
        }

        return ['User'];
    }
}
```

### External Group Provider

```php
use Symfony\Component\Validator\GroupProviderInterface;

class UserGroupProvider implements GroupProviderInterface
{
    public function getGroupSequence(object $object): array
    {
        if ($object->isPremium()) {
            return ['User', 'Premium'];
        }

        return ['User'];
    }
}

// In entity
#[Assert\GroupSequenceProvider(provider: UserGroupProvider::class)]
class User { }
```

### Nested Group Sequences

```php
public function getGroupSequence(): array
{
    // Sequential: stops at first failure
    return ['User', 'Premium', 'Api'];

    // Grouped: validates all in sub-array before moving on
    return [['User', 'Premium'], 'Api'];
}
```

## Custom Constraints

### Constraint Class

```php
// src/Validator/ContainsAlphanumeric.php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;

#[\Attribute]
class ContainsAlphanumeric extends Constraint
{
    public string $message = 'The string "{{ string }}" contains invalid characters.';
    public string $mode = 'strict';

    public function __construct(
        ?string $mode = null,
        ?string $message = null,
        ?array $groups = null,
        mixed $payload = null,
    ) {
        parent::__construct(null, $groups, $payload);

        $this->mode = $mode ?? $this->mode;
        $this->message = $message ?? $this->message;
    }
}
```

### Validator Class

```php
// src/Validator/ContainsAlphanumericValidator.php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;
use Symfony\Component\Validator\ConstraintValidator;
use Symfony\Component\Validator\Exception\UnexpectedTypeException;
use Symfony\Component\Validator\Exception\UnexpectedValueException;

class ContainsAlphanumericValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if (!$constraint instanceof ContainsAlphanumeric) {
            throw new UnexpectedTypeException($constraint, ContainsAlphanumeric::class);
        }

        // Ignore null and empty values (let other constraints handle them)
        if (null === $value || '' === $value) {
            return;
        }

        if (!is_string($value)) {
            throw new UnexpectedValueException($value, 'string');
        }

        $pattern = 'strict' === $constraint->mode
            ? '/^[a-zA-Z0-9]+$/'
            : '/^[a-zA-Z0-9\s]+$/';

        if (!preg_match($pattern, $value)) {
            $this->context->buildViolation($constraint->message)
                ->setParameter('{{ string }}', $value)
                ->addViolation();
        }
    }
}
```

### Using #[HasNamedArguments]

```php
use Symfony\Component\Validator\Attribute\HasNamedArguments;
use Symfony\Component\Validator\Constraint;

#[\Attribute]
class ContainsAlphanumeric extends Constraint
{
    public string $message = 'Invalid characters.';

    #[HasNamedArguments]
    public function __construct(
        public string $mode,  // Required option
        ?string $message = null,
        ?array $groups = null,
        mixed $payload = null,
    ) {
        parent::__construct(null, $groups, $payload);
        $this->message = $message ?? $this->message;
    }
}
```

### Class Constraint (Validate Entire Object)

```php
// src/Validator/ConfirmedPaymentReceipt.php
namespace App\Validator;

use Symfony\Component\Validator\Constraint;

#[\Attribute]
class ConfirmedPaymentReceipt extends Constraint
{
    public string $message = 'Email addresses do not match.';

    public function getTargets(): string
    {
        return self::CLASS_CONSTRAINT;
    }
}

// src/Validator/ConfirmedPaymentReceiptValidator.php
class ConfirmedPaymentReceiptValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if (!$value instanceof PaymentReceipt) {
            throw new UnexpectedValueException($value, PaymentReceipt::class);
        }

        if (!$constraint instanceof ConfirmedPaymentReceipt) {
            throw new UnexpectedTypeException($constraint, ConfirmedPaymentReceipt::class);
        }

        $receiptEmail = $value->getPayload()['email'] ?? null;
        $userEmail = $value->getUser()->getEmail();

        if ($userEmail !== $receiptEmail) {
            $this->context->buildViolation($constraint->message)
                ->atPath('user.email')
                ->addViolation();
        }
    }
}
```

### Validator with Dependencies

```php
class UniqueUsernameValidator extends ConstraintValidator
{
    public function __construct(
        private UserRepository $userRepository,
    ) {}

    public function validate(mixed $value, Constraint $constraint): void
    {
        if ($this->userRepository->findByUsername($value)) {
            $this->context->buildViolation($constraint->message)
                ->setParameter('{{ username }}', $value)
                ->addViolation();
        }
    }
}
```

## Compound Constraints

Create reusable constraint sets:

```php
// src/Validator/PasswordRequirements.php
namespace App\Validator;

use Symfony\Component\Validator\Constraints as Assert;

#[\Attribute]
class PasswordRequirements extends Assert\Compound
{
    protected function getConstraints(array $options): array
    {
        return [
            new Assert\NotBlank(),
            new Assert\Length(min: 8, max: 255),
            new Assert\NotCompromisedPassword(),
            new Assert\Regex(
                pattern: '/[A-Z]+/',
                message: 'Password must contain at least one uppercase letter.',
            ),
            new Assert\Regex(
                pattern: '/[0-9]+/',
                message: 'Password must contain at least one number.',
            ),
        ];
    }
}

// Usage
class User
{
    #[PasswordRequirements]
    private string $password;
}
```

## Constraint Violations

### ConstraintViolationListInterface

```php
$violations = $validator->validate($object);

// Check if there are violations
if (count($violations) > 0) {
    foreach ($violations as $violation) {
        $propertyPath = $violation->getPropertyPath();
        $message = $violation->getMessage();
        $invalidValue = $violation->getInvalidValue();
        $code = $violation->getCode();
        $constraint = $violation->getConstraint();
    }
}

// Convert to string
$errorsString = (string) $violations;
```

### Filtering Violations by Code

```php
use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

$violations = $validator->validate($entity);

// Find specific error codes
$uniqueViolations = $violations->findByCodes(UniqueEntity::NOT_UNIQUE_ERROR);
if (0 !== count($uniqueViolations)) {
    // Handle duplicate entity
}
```

### Building Violations in Validators

```php
$this->context->buildViolation($constraint->message)
    ->setParameter('{{ value }}', $value)
    ->setParameter('{{ limit }}', $constraint->limit)
    ->atPath('propertyName')      // Set property path
    ->setCode(MyConstraint::CODE) // Set error code
    ->setCause($cause)            // Set cause
    ->setInvalidValue($value)     // Override invalid value
    ->setPlural($count)           // For pluralization
    ->addViolation();
```

## Testing Constraints

### Testing Atomic Constraints

```php
namespace App\Tests\Validator;

use App\Validator\ContainsAlphanumeric;
use App\Validator\ContainsAlphanumericValidator;
use PHPUnit\Framework\Attributes\DataProvider;
use Symfony\Component\Validator\ConstraintValidatorInterface;
use Symfony\Component\Validator\Test\ConstraintValidatorTestCase;

class ContainsAlphanumericValidatorTest extends ConstraintValidatorTestCase
{
    protected function createValidator(): ConstraintValidatorInterface
    {
        return new ContainsAlphanumericValidator();
    }

    public function testNullIsValid(): void
    {
        $this->validator->validate(null, new ContainsAlphanumeric());
        $this->assertNoViolation();
    }

    public function testEmptyStringIsValid(): void
    {
        $this->validator->validate('', new ContainsAlphanumeric());
        $this->assertNoViolation();
    }

    public function testValidAlphanumeric(): void
    {
        $this->validator->validate('abc123', new ContainsAlphanumeric());
        $this->assertNoViolation();
    }

    #[DataProvider('provideInvalidValues')]
    public function testInvalidValues(string $value): void
    {
        $constraint = new ContainsAlphanumeric(message: 'myMessage');

        $this->validator->validate($value, $constraint);

        $this->buildViolation('myMessage')
            ->setParameter('{{ string }}', $value)
            ->assertRaised();
    }

    public static function provideInvalidValues(): \Generator
    {
        yield ['abc@123'];
        yield ['hello world'];
        yield ['test!'];
    }
}
```

### Testing Compound Constraints

```php
namespace App\Tests\Validator;

use App\Validator\PasswordRequirements;
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Test\CompoundConstraintTestCase;

/**
 * @extends CompoundConstraintTestCase<PasswordRequirements>
 */
class PasswordRequirementsTest extends CompoundConstraintTestCase
{
    public function createCompound(): Assert\Compound
    {
        return new PasswordRequirements();
    }

    public function testWeakPassword(): void
    {
        $this->validateValue('password');

        $this->assertViolationsRaisedByCompound([
            new Assert\NotCompromisedPassword(),
            new Assert\Regex('/[A-Z]+/'),
            new Assert\Regex('/[0-9]+/'),
        ]);
    }

    public function testValidPassword(): void
    {
        $this->validateValue('Str0ngP@ssword!');
        $this->assertNoViolation();
    }
}
```

## Extending Validation

### Adding Constraints to Third-Party Classes

```php
use Symfony\Component\Validator\Attribute\ExtendsValidationFor;
use Symfony\Component\Validator\Constraints as Assert;

#[ExtendsValidationFor(ThirdPartyProduct::class)]
abstract class ProductValidation
{
    #[Assert\NotBlank(groups: ['my_app'])]
    #[Assert\Length(min: 10, groups: ['my_app'])]
    public string $name = '';

    #[Assert\Positive(groups: ['my_app'])]
    public int $price = 0;
}
```

### Constraints in Form Classes

```php
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Validator\Constraints as Assert;

class AuthorType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('name', TextType::class, [
                'constraints' => [
                    new Assert\NotBlank(),
                    new Assert\Length(min: 3),
                ],
            ])
            ->add('email', TextType::class, [
                'constraints' => [
                    new Assert\Email(),
                ],
            ])
        ;
    }
}
```

## Debugging

### Debug Command

```bash
# Show constraints for a specific class
php bin/console debug:validator 'App\Entity\Author'

# Show constraints for all entities in a directory
php bin/console debug:validator src/Entity

# Show constraints with groups
php bin/console debug:validator 'App\Entity\User' --show-all
```

### Rendering Validation Errors in Templates

```twig
{# templates/author/validation.html.twig #}
{% if errors|length > 0 %}
    <ul class="errors">
    {% for error in errors %}
        <li>{{ error.propertyPath }}: {{ error.message }}</li>
    {% endfor %}
    </ul>
{% endif %}
```

### Form Error Rendering

```twig
{{ form_errors(form) }}

{# Or for a specific field #}
{{ form_errors(form.name) }}
```
