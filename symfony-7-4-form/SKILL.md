---
name: "symfony-7-4-form"
description: "Symfony 7.4 Form component reference for building, processing, and validating forms. Use when creating form types, handling form submissions, working with form events, data transformers, form validation, form rendering, or any form-related Symfony code. Triggers on: Form, FormType, FormBuilder, form events, data transformers, form validation, form rendering, AbstractType, FormFactoryInterface, FormBuilderInterface, ChoiceType, EntityType, configureOptions, buildForm, buildView, FormEvents, PRE_SET_DATA, POST_SUBMIT, DataTransformerInterface, CallbackTransformer, TypeTestCase, form_widget, form_row, form themes, validation_groups, DataMapperInterface, FormTypeExtensionInterface."
---

# Symfony 7.4 Form Component

GitHub: https://github.com/symfony/form
Docs: https://symfony.com/doc/7.4/components/form.html

## Quick Reference

### Creating a Form Type

```php
namespace App\Form;

use App\Entity\Task;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\DateType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class TaskType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('task', TextType::class)
            ->add('dueDate', DateType::class);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'data_class' => Task::class,
        ]);
    }
}
```

### Handling Submission in Controller

```php
public function new(Request $request): Response
{
    $task = new Task();
    $form = $this->createForm(TaskType::class, $task);
    $form->handleRequest($request);

    if ($form->isSubmitted() && $form->isValid()) {
        // persist $task ...
        return $this->redirectToRoute('task_success');
    }

    return $this->render('task/new.html.twig', ['form' => $form]);
}
```

### Adding Validation Constraints

```php
use Symfony\Component\Validator\Constraints as Assert;

$builder->add('task', TextType::class, [
    'constraints' => [new Assert\NotBlank(), new Assert\Length(['min' => 3])],
]);
```

### Form Events (Dynamic Fields)

```php
use Symfony\Component\Form\FormEvents;
use Symfony\Component\Form\FormEvent;

$builder->addEventListener(FormEvents::PRE_SET_DATA, function (FormEvent $event): void {
    $data = $event->getData();
    $form = $event->getForm();
    if (!$data || null === $data->getId()) {
        $form->add('name', TextType::class);
    }
});
```

### Data Transformer (Model Transformer)

```php
use Symfony\Component\Form\CallbackTransformer;

$builder->get('tags')
    ->addModelTransformer(new CallbackTransformer(
        fn (?array $tags): string => implode(', ', $tags ?? []),
        fn (string $tags): array => explode(', ', $tags),
    ));
```

### Rendering in Twig

```twig
{{ form_start(form) }}
    {{ form_row(form.task) }}
    {{ form_row(form.dueDate) }}
    <button type="submit">Save</button>
{{ form_end(form) }}
```

### Unit Testing a Form Type

```php
use Symfony\Component\Form\Test\TypeTestCase;

class TaskTypeTest extends TypeTestCase
{
    public function testSubmitValidData(): void
    {
        $formData = ['task' => 'Write tests', 'dueDate' => ['year' => '2026', 'month' => '1', 'day' => '1']];
        $form = $this->factory->create(TaskType::class);
        $form->submit($formData);

        $this->assertTrue($form->isSynchronized());
        $this->assertTrue($form->isValid());
    }
}
```

## Full Documentation

For complete details including all form events, data transformers (model vs view), data mappers, custom form types, form type extensions, dynamic form modification, validation groups, form rendering functions, form themes, and testing patterns, see [references/form.md](references/form.md).
