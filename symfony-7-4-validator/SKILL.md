---
name: "symfony-7-4-validator"
description: "Symfony 7.4 Validator component reference for data validation following JSR-303 Bean Validation specification. Use when validating objects, properties, or raw values, defining validation constraints, creating custom validators, working with validation groups, or handling constraint violations. Triggers on: Validator, validation constraints, Assert, ConstraintValidator, validation groups, custom constraints, ValidatorInterface, Constraint, ConstraintViolation, ConstraintViolationList, NotBlank, NotNull, Length, Email, Choice, Valid, Callback, GroupSequence, GroupSequenceProvider, Compound, UniqueEntity."
---

# Symfony 7.4 Validator Component

GitHub: https://github.com/symfony/validator
Docs: https://symfony.com/doc/7.4/components/validator.html

## Quick Reference

### Basic Validation

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Validator\ValidatorInterface;

class Author
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 3, max: 50)]
    private string $name;

    #[Assert\Email]
    private string $email;
}

// In a controller or service
public function validateAuthor(Author $author, ValidatorInterface $validator): void
{
    $errors = $validator->validate($author);

    if (count($errors) > 0) {
        foreach ($errors as $violation) {
            echo $violation->getPropertyPath() . ': ' . $violation->getMessage();
        }
    }
}
```

### Common Constraints

```php
use Symfony\Component\Validator\Constraints as Assert;

class User
{
    #[Assert\NotBlank]
    #[Assert\NotNull]
    private string $username;

    #[Assert\Length(min: 8, max: 255)]
    #[Assert\NotCompromisedPassword]
    private string $password;

    #[Assert\Email(mode: 'strict')]
    private string $email;

    #[Assert\Choice(choices: ['user', 'admin', 'moderator'])]
    private string $role;

    #[Assert\Range(min: 18, max: 120)]
    private int $age;

    #[Assert\Url]
    private ?string $website = null;

    #[Assert\Valid]  // Cascade validation to nested object
    private Address $address;
}
```

### Validation Groups

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

// Validate with specific groups
$errors = $validator->validate($user, null, ['registration']);
```

### Custom Constraint

```php
// src/Validator/ContainsAlphanumeric.php
use Symfony\Component\Validator\Constraint;

#[\Attribute]
class ContainsAlphanumeric extends Constraint
{
    public string $message = 'The string "{{ string }}" contains invalid characters.';
}

// src/Validator/ContainsAlphanumericValidator.php
use Symfony\Component\Validator\ConstraintValidator;

class ContainsAlphanumericValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if (null === $value || '' === $value) {
            return;
        }

        if (!preg_match('/^[a-zA-Z0-9]+$/', $value)) {
            $this->context->buildViolation($constraint->message)
                ->setParameter('{{ string }}', $value)
                ->addViolation();
        }
    }
}
```

### Class Constraint (Validate Entire Object)

```php
use Symfony\Component\Validator\Constraint;

#[\Attribute]
class PasswordMatch extends Constraint
{
    public string $message = 'Passwords do not match.';

    public function getTargets(): string
    {
        return self::CLASS_CONSTRAINT;
    }
}

class PasswordMatchValidator extends ConstraintValidator
{
    public function validate(mixed $value, Constraint $constraint): void
    {
        if ($value->getPassword() !== $value->getConfirmPassword()) {
            $this->context->buildViolation($constraint->message)
                ->atPath('confirmPassword')
                ->addViolation();
        }
    }
}
```

### Callback Constraint

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Context\ExecutionContextInterface;

class Author
{
    #[Assert\Callback]
    public function validate(ExecutionContextInterface $context): void
    {
        if ($this->firstName === $this->lastName) {
            $context->buildViolation('First and last name cannot be identical.')
                ->atPath('firstName')
                ->addViolation();
        }
    }
}
```

### GroupSequence

```php
use Symfony\Component\Validator\Constraints as Assert;

#[Assert\GroupSequence(['User', 'Strict'])]
class User
{
    #[Assert\NotBlank]
    private string $username;

    #[Assert\IsTrue(message: 'Password cannot match username', groups: ['Strict'])]
    public function isPasswordSafe(): bool
    {
        return $this->username !== $this->password;
    }
}
```

### Compound Constraint (Reusable Set)

```php
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
            new Assert\Regex('/[A-Z]+/'),
            new Assert\Regex('/[0-9]+/'),
        ];
    }
}
```

### Validating Raw Values

```php
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Validation;

$validator = Validation::createValidator();

$violations = $validator->validate('invalid-email', [
    new Assert\Email(),
    new Assert\NotBlank(),
]);

// Using validation callables
$validateEmail = Validation::createCallable(new Assert\Email());
$validateEmail('test@example.com');  // Throws ValidationFailedException on error

$isValidEmail = Validation::createIsValidCallable(new Assert\Email());
if (!$isValidEmail('test@example.com')) {
    // Handle invalid email
}
```

## Full Documentation

For complete details including all built-in constraints, constraint options, YAML/XML configuration, testing constraints, translation of error messages, severity levels, and advanced patterns, see [references/validator.md](references/validator.md).
