# Symfony 7.4 Form Component - Complete Reference

## Overview

The Form component provides tools for creating, processing, and reusing HTML forms. It handles form validation, CSRF protection, data binding to objects, and flexible rendering.

**GitHub**: https://github.com/symfony/form
**Documentation**: https://symfony.com/doc/7.4/components/form.html
**License**: MIT

## Installation

```bash
composer require symfony/form
```

For full functionality, also install:

```bash
# HTTP request handling
composer require symfony/http-foundation

# CSRF protection
composer require symfony/security-csrf

# Validation
composer require symfony/validator

# Twig rendering
composer require symfony/twig-bridge

# Translation
composer require symfony/translation
```

## Basic Usage

### Creating a Form Factory

```php
use Symfony\Component\Form\Forms;

$formFactory = Forms::createFormFactory();
```

### Creating Forms

```php
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\Extension\Core\Type\DateType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;

$form = $formFactory->createBuilder()
    ->add('task', TextType::class)
    ->add('dueDate', DateType::class)
    ->add('save', SubmitType::class, ['label' => 'Create Task'])
    ->getForm();
```

### With Symfony Framework

```php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\Extension\Core\Type\DateType;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;

class TaskController extends AbstractController
{
    public function new(Request $request): Response
    {
        $form = $this->createFormBuilder()
            ->add('task', TextType::class)
            ->add('dueDate', DateType::class)
            ->getForm();

        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $data = $form->getData();
            // Process data...
            return $this->redirectToRoute('task_success');
        }

        return $this->render('task/new.html.twig', [
            'form' => $form,
        ]);
    }
}
```

## Form Types

### Creating Custom Form Types

```php
namespace App\Form\Type;

use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\DateType;
use Symfony\Component\Form\Extension\Core\Type\SubmitType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class TaskType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('task', TextType::class)
            ->add('dueDate', DateType::class)
            ->add('save', SubmitType::class);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => Task::class,
        ]);
    }
}
```

### Using the Type

```php
$form = $this->createForm(TaskType::class, $task);
```

### Built-in Field Types

**Text Fields:**
- `TextType` - Single line text input
- `TextareaType` - Multi-line text area
- `EmailType` - Email input with validation
- `IntegerType` - Integer number input
- `MoneyType` - Money/currency input
- `NumberType` - Decimal number input
- `PasswordType` - Password input (masked)
- `PercentType` - Percentage input
- `SearchType` - Search input
- `UrlType` - URL input with validation
- `TelType` - Telephone number input
- `ColorType` - Color picker input

**Choice Fields:**
- `ChoiceType` - Select dropdown, checkboxes, or radio buttons
- `EntityType` - Doctrine entity selection
- `CountryType` - Country selector
- `LanguageType` - Language selector
- `LocaleType` - Locale selector
- `TimezoneType` - Timezone selector
- `CurrencyType` - Currency selector
- `EnumType` - PHP Enum selection

**Date and Time:**
- `DateType` - Date input
- `DateIntervalType` - Date interval input
- `DateTimeType` - Combined date and time
- `TimeType` - Time input
- `BirthdayType` - Birthday selector
- `WeekType` - Week selector

**Other Fields:**
- `CheckboxType` - Single checkbox
- `FileType` - File upload
- `RadioType` - Single radio button
- `CollectionType` - Collection of embedded forms
- `HiddenType` - Hidden field
- `ButtonType` - Generic button
- `ResetType` - Reset button
- `SubmitType` - Submit button

## Form Options

### Common Options

```php
$builder->add('name', TextType::class, [
    'label' => 'Your Name',
    'required' => true,
    'disabled' => false,
    'attr' => ['class' => 'form-control', 'placeholder' => 'Enter name'],
    'label_attr' => ['class' => 'form-label'],
    'help' => 'Please enter your full name',
    'empty_data' => '',
    'mapped' => true,
    'data' => 'default value',
    'constraints' => [
        new Assert\NotBlank(),
        new Assert\Length(['min' => 2, 'max' => 100]),
    ],
]);
```

### Form-Level Options

```php
$form = $this->createFormBuilder($data, [
    'action' => '/submit',
    'method' => 'POST',
    'attr' => ['id' => 'my-form', 'class' => 'form'],
    'csrf_protection' => true,
    'csrf_field_name' => '_token',
    'csrf_token_id' => 'task_form',
]);
```

## Data Binding

### Binding to Objects

```php
class Task
{
    private ?string $task = null;
    private ?\DateTimeInterface $dueDate = null;

    public function getTask(): ?string
    {
        return $this->task;
    }

    public function setTask(?string $task): void
    {
        $this->task = $task;
    }

    public function getDueDate(): ?\DateTimeInterface
    {
        return $this->dueDate;
    }

    public function setDueDate(?\DateTimeInterface $dueDate): void
    {
        $this->dueDate = $dueDate;
    }
}

// Configure data_class
public function configureOptions(OptionsResolver $resolver): void
{
    $resolver->setDefaults([
        'data_class' => Task::class,
    ]);
}

// Usage
$task = new Task();
$form = $this->createForm(TaskType::class, $task);

$form->handleRequest($request);
if ($form->isSubmitted() && $form->isValid()) {
    // $task is now populated with form data
    $entityManager->persist($task);
    $entityManager->flush();
}
```

### Unmapped Fields

```php
$builder->add('termsAccepted', CheckboxType::class, [
    'mapped' => false,
    'constraints' => new Assert\IsTrue([
        'message' => 'You must accept the terms.',
    ]),
]);
```

## Form Events

### Available Events

```php
use Symfony\Component\Form\FormEvents;

FormEvents::PRE_SET_DATA   // Before data is set on the form
FormEvents::POST_SET_DATA  // After data is set on the form
FormEvents::PRE_SUBMIT     // Before data is submitted
FormEvents::SUBMIT         // When data is being normalized
FormEvents::POST_SUBMIT    // After data is submitted and validated
```

### Adding Event Listeners

```php
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;

public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder->addEventListener(FormEvents::PRE_SET_DATA, function (FormEvent $event): void {
        $data = $event->getData();
        $form = $event->getForm();

        // Modify form based on data
        if (!$data || null === $data->getId()) {
            $form->add('name', TextType::class);
        }
    });

    $builder->addEventListener(FormEvents::PRE_SUBMIT, function (FormEvent $event): void {
        $data = $event->getData();
        $form = $event->getForm();

        // Modify form based on submitted data
    });
}
```

### Event Subscribers

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Form\FormEvent;
use Symfony\Component\Form\FormEvents;

class AddNameFieldSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            FormEvents::PRE_SET_DATA => 'preSetData',
        ];
    }

    public function preSetData(FormEvent $event): void
    {
        $data = $event->getData();
        $form = $event->getForm();

        if (!$data || null === $data->getId()) {
            $form->add('name', TextType::class);
        }
    }
}

// Usage in form type
public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder->addEventSubscriber(new AddNameFieldSubscriber());
}
```

## Data Transformers

### Creating a Transformer

```php
use Symfony\Component\Form\DataTransformerInterface;
use Symfony\Component\Form\Exception\TransformationFailedException;

class IssueToNumberTransformer implements DataTransformerInterface
{
    public function __construct(
        private IssueRepository $issueRepository,
    ) {
    }

    /**
     * Transforms an object (Issue) to a string (number).
     */
    public function transform(mixed $value): string
    {
        if (null === $value) {
            return '';
        }

        return $value->getNumber();
    }

    /**
     * Transforms a string (number) to an object (Issue).
     */
    public function reverseTransform(mixed $value): ?Issue
    {
        if (!$value) {
            return null;
        }

        $issue = $this->issueRepository->findOneByNumber($value);

        if (null === $issue) {
            throw new TransformationFailedException(sprintf(
                'An issue with number "%s" does not exist!',
                $value
            ));
        }

        return $issue;
    }
}
```

### Using Transformers

```php
public function buildForm(FormBuilderInterface $builder, array $options): void
{
    $builder->add('issue', TextType::class);

    $builder->get('issue')
        ->addModelTransformer(new IssueToNumberTransformer($this->issueRepository));
}
```

### Callback Transformer

```php
use Symfony\Component\Form\CallbackTransformer;

$builder->add('tags', TextType::class);

$builder->get('tags')
    ->addModelTransformer(new CallbackTransformer(
        fn ($tagsAsArray) => implode(', ', $tagsAsArray ?? []),
        fn ($tagsAsString) => explode(', ', $tagsAsString)
    ));
```

## Validation

### Adding Constraints

```php
use Symfony\Component\Validator\Constraints as Assert;

$builder
    ->add('email', EmailType::class, [
        'constraints' => [
            new Assert\NotBlank(),
            new Assert\Email(),
        ],
    ])
    ->add('password', PasswordType::class, [
        'constraints' => [
            new Assert\NotBlank(),
            new Assert\Length(['min' => 8]),
        ],
    ]);
```

### Validation Groups

```php
public function configureOptions(OptionsResolver $resolver): void
{
    $resolver->setDefaults([
        'validation_groups' => ['registration'],
    ]);
}

// Dynamic validation groups
$resolver->setDefaults([
    'validation_groups' => function (FormInterface $form) {
        $data = $form->getData();
        if ($data->getType() === 'premium') {
            return ['Default', 'premium'];
        }
        return ['Default'];
    },
]);
```

### Accessing Errors

```php
// Get all errors (flat)
$errors = $form->getErrors(true, true);

// Get errors for specific field
$nameErrors = $form->get('name')->getErrors();

// Iterate errors
foreach ($form->getErrors(true) as $error) {
    echo $error->getMessage();
}

// Clear errors
$form->clearErrors();
```

## Embedded Forms

### Embedding Single Forms

```php
// CategoryType
class CategoryType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder->add('name', TextType::class);
    }
}

// TaskType with embedded CategoryType
class TaskType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('description', TextareaType::class)
            ->add('category', CategoryType::class);
    }
}
```

### Collections

```php
use Symfony\Component\Form\Extension\Core\Type\CollectionType;

$builder->add('tags', CollectionType::class, [
    'entry_type' => TagType::class,
    'entry_options' => ['label' => false],
    'allow_add' => true,
    'allow_delete' => true,
    'by_reference' => false,
    'prototype' => true,
    'prototype_name' => '__name__',
]);
```

## Form Rendering

### Twig Functions

```twig
{# Render entire form #}
{{ form(form) }}

{# Render form with custom options #}
{{ form_start(form, {'method': 'GET', 'attr': {'class': 'my-form'}}) }}
{{ form_widget(form) }}
{{ form_end(form) }}

{# Render individual fields #}
{{ form_row(form.name) }}
{{ form_label(form.name) }}
{{ form_widget(form.name) }}
{{ form_errors(form.name) }}
{{ form_help(form.name) }}

{# Render specific field with options #}
{{ form_widget(form.name, {'attr': {'class': 'form-control'}}) }}
```

### Form Themes

```yaml
# config/packages/twig.yaml
twig:
    form_themes: ['bootstrap_5_layout.html.twig']
```

Built-in themes:
- `form_div_layout.html.twig` - Default divs
- `form_table_layout.html.twig` - Table layout
- `bootstrap_3_layout.html.twig` - Bootstrap 3
- `bootstrap_4_layout.html.twig` - Bootstrap 4
- `bootstrap_5_layout.html.twig` - Bootstrap 5
- `tailwind_2_layout.html.twig` - Tailwind CSS

### Custom Form Theme

```twig
{# templates/form/fields.html.twig #}

{% block text_widget %}
    <input type="text" {{ block('widget_attributes') }} value="{{ value }}" class="custom-input">
{% endblock %}

{% block form_row %}
    <div class="form-group">
        {{ form_label(form) }}
        {{ form_widget(form) }}
        {{ form_errors(form) }}
        {{ form_help(form) }}
    </div>
{% endblock %}
```

## Form Type Extensions

### Creating an Extension

```php
namespace App\Form\Extension;

use Symfony\Component\Form\AbstractTypeExtension;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormInterface;
use Symfony\Component\Form\FormView;
use Symfony\Component\OptionsResolver\OptionsResolver;

class TextTypeExtension extends AbstractTypeExtension
{
    public static function getExtendedTypes(): iterable
    {
        return [TextType::class];
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'help_icon' => null,
        ]);
    }

    public function buildView(FormView $view, FormInterface $form, array $options): void
    {
        $view->vars['help_icon'] = $options['help_icon'];
    }
}
```

## Configuration

### Full Factory Setup

```php
use Symfony\Component\Form\Extension\Csrf\CsrfExtension;
use Symfony\Component\Form\Extension\HttpFoundation\HttpFoundationExtension;
use Symfony\Component\Form\Extension\Validator\ValidatorExtension;
use Symfony\Component\Form\Forms;
use Symfony\Component\Security\Csrf\CsrfTokenManager;
use Symfony\Component\Validator\Validation;

$csrfManager = new CsrfTokenManager();
$validator = Validation::createValidator();

$formFactory = Forms::createFormFactoryBuilder()
    ->addExtension(new HttpFoundationExtension())
    ->addExtension(new CsrfExtension($csrfManager))
    ->addExtension(new ValidatorExtension($validator))
    ->getFormFactory();
```

## Key Interfaces

| Interface | Purpose |
|-----------|---------|
| `FormInterface` | Contract for form objects |
| `FormBuilderInterface` | Building forms fluently |
| `FormTypeInterface` | Defining form types |
| `FormTypeExtensionInterface` | Extending existing types |
| `DataTransformerInterface` | Transforming data formats |
| `DataMapperInterface` | Mapping data to/from objects |
| `FormFactoryInterface` | Creating form instances |
| `RequestHandlerInterface` | Processing HTTP requests |

## Key Classes

| Class | Purpose |
|-------|---------|
| `AbstractType` | Base class for custom form types |
| `AbstractTypeExtension` | Base for type extensions |
| `FormBuilder` | Builds form structure |
| `FormFactory` | Creates form instances |
| `FormView` | Represents form for rendering |
| `FormError` | Represents validation error |
| `FormEvent` | Form event data |
| `CallbackTransformer` | Simple data transformer |

## Best Practices

1. **Create Form Type Classes** - Keep forms reusable and testable
2. **Use Data Classes** - Bind forms to objects instead of arrays
3. **Validate at Form Level** - Use constraints in form types
4. **Use Form Events** - For dynamic form modifications
5. **Create Custom Types** - For reusable field combinations
6. **Test Forms** - Use functional tests for form submission
7. **Use CSRF Protection** - Always enable in production

## Testing Forms

```php
use Symfony\Component\Form\Test\TypeTestCase;

class TaskTypeTest extends TypeTestCase
{
    public function testSubmitValidData(): void
    {
        $formData = [
            'task' => 'Write tests',
            'dueDate' => '2024-12-31',
        ];

        $model = new Task();
        $form = $this->factory->create(TaskType::class, $model);

        $expected = new Task();
        $expected->setTask('Write tests');
        $expected->setDueDate(new \DateTime('2024-12-31'));

        $form->submit($formData);

        $this->assertTrue($form->isSynchronized());
        $this->assertEquals($expected, $model);
    }
}
```
