# Symfony 7.4 Workflow Component - Complete Reference

## Overview

The Workflow component provides tools for managing a workflow or finite state machine. It gives you an object-oriented way to define a process or lifecycle that your objects go through.

## Installation

```bash
composer require symfony/workflow
```

## Core Concepts

### Places

Places are the steps or stages in your process. They represent states that an object can be in.

Example places for a blog post: `draft`, `reviewed`, `rejected`, `published`

### Transitions

Transitions describe the actions to get from one place to another. Each transition has:
- A unique name
- An origin place (from)
- A destination place (to)

Example: A `to_review` transition moves a post from `draft` to `reviewed`

### Definition

A definition is a set of places and transitions that creates the complete workflow structure.

### Marking Store

The marking store is responsible for writing the current state(s) to your object. It implements `MarkingStoreInterface`.

## Creating Workflows

### YAML Configuration

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        blog_publishing:
            type: 'workflow' # or 'state_machine'
            audit_trail:
                enabled: true
            marking_store:
                type: 'method'
                property: 'currentPlace'
            supports:
                - App\Entity\BlogPost
            initial_marking: draft
            places:
                - draft
                - reviewed
                - rejected
                - published
            transitions:
                to_review:
                    from: draft
                    to: reviewed
                publish:
                    from: reviewed
                    to: published
                reject:
                    from: reviewed
                    to: rejected
```

### PHP Configuration

```php
// config/packages/workflow.php
use App\Entity\BlogPost;
use Symfony\Config\FrameworkConfig;

return static function (FrameworkConfig $framework): void {
    $blogPublishing = $framework->workflows()->workflow('blog_publishing');
    $blogPublishing
        ->type('workflow')
        ->supports([BlogPost::class])
        ->initialMarking(['draft']);

    $blogPublishing->auditTrail()->enabled(true);
    $blogPublishing->markingStore()
        ->type('method')
        ->property('currentPlace');

    $blogPublishing->place()->name('draft');
    $blogPublishing->place()->name('reviewed');
    $blogPublishing->place()->name('rejected');
    $blogPublishing->place()->name('published');

    $blogPublishing->transition()
        ->name('to_review')
        ->from('draft')
        ->to('reviewed');

    $blogPublishing->transition()
        ->name('publish')
        ->from('reviewed')
        ->to('published');

    $blogPublishing->transition()
        ->name('reject')
        ->from('reviewed')
        ->to('rejected');
};
```

### Programmatic Definition

```php
use Symfony\Component\Workflow\DefinitionBuilder;
use Symfony\Component\Workflow\MarkingStore\MethodMarkingStore;
use Symfony\Component\Workflow\Transition;
use Symfony\Component\Workflow\Workflow;

$definitionBuilder = new DefinitionBuilder();
$definition = $definitionBuilder
    ->addPlaces(['draft', 'reviewed', 'rejected', 'published'])
    ->addTransition(new Transition('to_review', 'draft', 'reviewed'))
    ->addTransition(new Transition('publish', 'reviewed', 'published'))
    ->addTransition(new Transition('reject', 'reviewed', 'rejected'))
    ->build();

$singleState = true; // true = single state, false = multiple states
$property = 'currentState';
$marking = new MethodMarkingStore($singleState, $property);

$workflow = new Workflow($definition, $marking);
```

## Entity Implementation

### Single State (State Machine)

```php
namespace App\Entity;

class BlogPost
{
    private string $currentPlace;
    private string $title;
    private string $content;

    public function getCurrentPlace(): string
    {
        return $this->currentPlace;
    }

    public function setCurrentPlace(string $currentPlace, array $context = []): void
    {
        $this->currentPlace = $currentPlace;
    }
}
```

### Multiple States (Workflow)

For workflows that allow an object to be in multiple places simultaneously:

```php
use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class BlogPost
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private int $id;

    #[ORM\Column(type: Types::JSON)]
    private array $currentPlaces = [];

    public function getCurrentPlaces(): array
    {
        return $this->currentPlaces;
    }

    public function setCurrentPlaces(array $currentPlaces, array $context = []): void
    {
        $this->currentPlaces = $currentPlaces;
    }
}
```

## Using Enums as Places (Symfony 7.4+)

```php
// src/Enumeration/BlogPostStatus.php
namespace App\Enumeration;

enum BlogPostStatus: string
{
    case Draft = 'draft';
    case Reviewed = 'reviewed';
    case Published = 'published';
    case Rejected = 'rejected';
}
```

```yaml
# config/packages/workflow.yaml
framework:
    workflows:
        blog_publishing:
            type: 'workflow'
            marking_store:
                type: 'method'
                property: 'status'
            supports:
                - App\Entity\BlogPost
            initial_marking: !php/enum App\Enumeration\BlogPostStatus::Draft
            places: !php/enum App\Enumeration\BlogPostStatus
            transitions:
                to_review:
                    from: !php/enum App\Enumeration\BlogPostStatus::Draft
                    to: !php/enum App\Enumeration\BlogPostStatus::Reviewed
```

## Workflow vs State Machine

| Aspect | Workflow | State Machine |
|--------|----------|---------------|
| Type | `workflow` | `state_machine` |
| Multiple Places | Yes (object can be in multiple places) | No (single place only) |
| Marking Store | `multiple_state` | `single_state` |
| Property Type | `array` | `string` |
| Use Case | Parallel processes, complex lifecycles | Simple sequential states |

## Using Workflows

### Basic Operations

```php
$blogPost = new BlogPost();

// Initialize workflow (sets initial marking if null)
$workflow->getMarking($blogPost);

// Check if transition is allowed
$workflow->can($blogPost, 'publish'); // false
$workflow->can($blogPost, 'to_review'); // true

// Apply a transition
$workflow->apply($blogPost, 'to_review');

// Get all enabled transitions
$transitions = $workflow->getEnabledTransitions($blogPost);

// Get specific enabled transition
$transition = $workflow->getEnabledTransition($blogPost, 'publish');

// Get current marking
$marking = $workflow->getMarking($blogPost);
$marking->has('reviewed'); // true/false
```

### Injecting Workflows

#### Method 1: Naming Convention

```php
use Symfony\Component\Workflow\WorkflowInterface;

class MyClass
{
    public function __construct(
        // Naming: {workflow_name}Workflow (camelCase)
        private WorkflowInterface $blogPublishingWorkflow,
    ) {
    }
}
```

#### Method 2: Target Attribute

```php
use Symfony\Component\DependencyInjection\Attribute\Target;
use Symfony\Component\Workflow\WorkflowInterface;

class MyClass
{
    public function __construct(
        #[Target('blog_publishing')] private WorkflowInterface $workflow,
    ) {
    }
}
```

#### Method 3: Service Locator

```php
use Symfony\Component\DependencyInjection\Attribute\AutowireLocator;
use Symfony\Component\DependencyInjection\ServiceLocator;

class MyClass
{
    public function __construct(
        #[AutowireLocator('workflow', 'name')]
        private ServiceLocator $workflows,
    ) {
    }

    public function someMethod(): void
    {
        $workflow = $this->workflows->get('user_registration');
    }
}
```

## Workflow Events

Events are dispatched during transitions in the following order:

### 1. Guard Events

Validate if transition should be blocked:
- `workflow.guard`
- `workflow.[workflow name].guard`
- `workflow.[workflow name].guard.[transition name]`

### 2. Leave Events

Subject leaving a place:
- `workflow.leave`
- `workflow.[workflow name].leave`
- `workflow.[workflow name].leave.[place name]`

### 3. Transition Events

Subject going through transition:
- `workflow.transition`
- `workflow.[workflow name].transition`
- `workflow.[workflow name].transition.[transition name]`

### 4. Enter Events

Subject entering a place (before marking update):
- `workflow.enter`
- `workflow.[workflow name].enter`
- `workflow.[workflow name].enter.[place name]`

### 5. Entered Events

Subject entered in places (after marking update):
- `workflow.entered`
- `workflow.[workflow name].entered`
- `workflow.[workflow name].entered.[place name]`

### 6. Completed Events

Object completed transition:
- `workflow.completed`
- `workflow.[workflow name].completed`
- `workflow.[workflow name].completed.[transition name]`

### 7. Announce Events

Triggered for each now-accessible transition:
- `workflow.announce`
- `workflow.[workflow name].announce`
- `workflow.[workflow name].announce.[transition name]`

## Event Subscriber Example

```php
namespace App\EventSubscriber;

use Psr\Log\LoggerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Workflow\Event\Event;
use Symfony\Component\Workflow\Event\LeaveEvent;

class WorkflowLoggerSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private LoggerInterface $logger,
    ) {
    }

    public function onLeave(Event $event): void
    {
        $this->logger->alert(sprintf(
            'Blog post (id: "%s") performed transition "%s" from "%s" to "%s"',
            $event->getSubject()->getId(),
            $event->getTransition()->getName(),
            implode(', ', array_keys($event->getMarking()->getPlaces())),
            implode(', ', $event->getTransition()->getTos())
        ));
    }

    public static function getSubscribedEvents(): array
    {
        return [
            LeaveEvent::getName('blog_publishing') => 'onLeave',
            // Alternative: 'workflow.blog_publishing.leave' => 'onLeave',
        ];
    }
}
```

## Event Listener Attributes (Symfony 7.1+)

```php
use Symfony\Component\Workflow\Attribute\AsAnnounceListener;
use Symfony\Component\Workflow\Attribute\AsCompletedListener;
use Symfony\Component\Workflow\Attribute\AsEnterListener;
use Symfony\Component\Workflow\Attribute\AsEnteredListener;
use Symfony\Component\Workflow\Attribute\AsGuardListener;
use Symfony\Component\Workflow\Attribute\AsLeaveListener;
use Symfony\Component\Workflow\Attribute\AsTransitionListener;

class ArticleWorkflowEventListener
{
    #[AsGuardListener(workflow: 'blog_publishing', transition: 'publish')]
    public function onGuardPublish(GuardEvent $event): void
    {
        // Validate if transition should be blocked
    }

    #[AsLeaveListener(workflow: 'blog_publishing', place: 'draft')]
    public function onLeaveDraft(LeaveEvent $event): void
    {
        // Leaving draft state
    }

    #[AsTransitionListener(workflow: 'blog_publishing', transition: 'publish')]
    public function onPublishTransition(TransitionEvent $event): void
    {
        // During publish transition
    }

    #[AsEnterListener(workflow: 'blog_publishing', place: 'published')]
    public function onEnterPublished(EnterEvent $event): void
    {
        // Entering published state (before marking update)
    }

    #[AsEnteredListener(workflow: 'blog_publishing', place: 'published')]
    public function onEnteredPublished(EnteredEvent $event): void
    {
        // Entered published state (after marking update)
    }

    #[AsCompletedListener(workflow: 'blog_publishing', transition: 'publish')]
    public function onCompleted(CompletedEvent $event): void
    {
        // Transition completed
    }

    #[AsAnnounceListener(workflow: 'blog_publishing', transition: 'archive')]
    public function onAnnounce(AnnounceEvent $event): void
    {
        // Archive transition is now available
    }
}
```

## Guard Events

Special events to block/allow transitions:

### Basic Guard

```php
namespace App\EventSubscriber;

use App\Entity\BlogPost;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Workflow\Event\GuardEvent;

class BlogPostReviewSubscriber implements EventSubscriberInterface
{
    public function guardReview(GuardEvent $event): void
    {
        /** @var BlogPost $post */
        $post = $event->getSubject();

        if (empty($post->getTitle())) {
            $event->setBlocked(true, 'Cannot review without a title.');
        }
    }

    public static function getSubscribedEvents(): array
    {
        return [
            'workflow.blog_publishing.guard.to_review' => ['guardReview'],
        ];
    }
}
```

### Expression Language Guards

```yaml
framework:
    workflows:
        blog_publishing:
            transitions:
                to_review:
                    guard: "is_granted('ROLE_REVIEWER')"
                    from: draft
                    to: reviewed
                publish:
                    guard: "is_authenticated"
                    from: reviewed
                    to: published
                reject:
                    guard: "is_granted('ROLE_ADMIN') and subject.isRejectable()"
                    from: reviewed
                    to: rejected
```

Available expression variables:
- `subject` - The object being transitioned
- `is_granted()` - Security voter check
- `is_authenticated` - Check if user is authenticated

## Transition Blockers

```php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\Workflow\Event\GuardEvent;
use Symfony\Component\Workflow\TransitionBlocker;

class BlogPostPublishSubscriber implements EventSubscriberInterface
{
    public function guardPublish(GuardEvent $event): void
    {
        $eventTransition = $event->getTransition();
        $hourLimit = $event->getMetadata('hour_limit', $eventTransition);

        if (date('H') > $hourLimit) {
            $explanation = $event->getMetadata('explanation', $eventTransition);
            $event->addTransitionBlocker(
                new TransitionBlocker($explanation, '0')
            );
        }
    }

    public static function getSubscribedEvents(): array
    {
        return [
            'workflow.blog_publishing.guard.publish' => ['guardPublish'],
        ];
    }
}
```

### TransitionBlocker Class

```php
use Symfony\Component\Workflow\TransitionBlocker;

// Create a blocker
$blocker = new TransitionBlocker('Reason message', 'error_code');

// Get blocker information
$blocker->getMessage();  // 'Reason message'
$blocker->getCode();     // 'error_code'
$blocker->getParameters(); // Additional parameters

// Built-in blockers
TransitionBlocker::createBlockedByMarking($marking);
TransitionBlocker::createBlockedByExpressionGuard($expression);
TransitionBlocker::createUnknown($transitionName, $fromPlace);
```

## Event Methods

All events inherit from `Event` class:

```php
$event->getMarking();        // Marking instance
$event->getSubject();        // Object dispatching event
$event->getTransition();     // Transition instance
$event->getWorkflowName();   // Workflow name string
$event->getMetadata();       // Event metadata
```

### GuardEvent Additional Methods

```php
$event->isBlocked();                    // bool
$event->setBlocked(bool, string);       // Block with reason
$event->getTransitionBlockerList();     // TransitionBlockerList
$event->addTransitionBlocker(TransitionBlocker);
```

## Choosing Events to Dispatch

### Configure Globally

```yaml
framework:
    workflows:
        blog_publishing:
            # Only dispatch specified events
            events_to_dispatch: ['workflow.leave', 'workflow.completed']

            # Or disable all events
            events_to_dispatch: []
```

### Disable Per Transition

```php
use Symfony\Component\Workflow\Workflow;

$workflow->apply($post, 'to_review', [
    Workflow::DISABLE_ANNOUNCE_EVENT => true,
    Workflow::DISABLE_LEAVE_EVENT => true,
]);
```

Available constants:
- `Workflow::DISABLE_LEAVE_EVENT`
- `Workflow::DISABLE_TRANSITION_EVENT`
- `Workflow::DISABLE_ENTER_EVENT`
- `Workflow::DISABLE_ENTERED_EVENT`
- `Workflow::DISABLE_COMPLETED_EVENT`
- `Workflow::DISABLE_ANNOUNCE_EVENT`

## Custom Marking Store

```php
namespace App\Workflow\MarkingStore;

use Symfony\Component\Workflow\Marking;
use Symfony\Component\Workflow\MarkingStore\MarkingStoreInterface;

final class BlogPostMarkingStore implements MarkingStoreInterface
{
    public function getMarking(object $subject): Marking
    {
        return new Marking([$subject->getCurrentPlace() => 1]);
    }

    public function setMarking(object $subject, Marking $marking, array $context = []): void
    {
        $place = key($marking->getPlaces());
        $subject->setCurrentPlace($place);
    }
}
```

### Register Custom Marking Store

```yaml
framework:
    workflows:
        blog_publishing:
            marking_store:
                service: 'App\Workflow\MarkingStore\BlogPostMarkingStore'
```

## Storing Metadata

### Configuration

```yaml
framework:
    workflows:
        blog_publishing:
            metadata:
                title: 'Blog Publishing Workflow'
            places:
                draft:
                    metadata:
                        max_num_of_words: 500
            transitions:
                to_review:
                    from: draft
                    to: review
                    metadata:
                        priority: 0.5
                publish:
                    from: reviewed
                    to: published
                    metadata:
                        hour_limit: 20
                        explanation: 'You cannot publish after 8 PM.'
```

### Access Metadata

```php
// Workflow metadata
$title = $workflow->getMetadataStore()->getWorkflowMetadata()['title'] ?? 'Default';

// Place metadata
$maxWords = $workflow->getMetadataStore()->getPlaceMetadata('draft')['max_num_of_words'] ?? 500;

// Transition metadata
$aTransition = $workflow->getDefinition()->getTransitions()[0];
$priority = $workflow->getMetadataStore()->getTransitionMetadata($aTransition)['priority'] ?? 0;

// Using getMetadata() shorthand
$title = $workflow->getMetadataStore()->getMetadata('title');
$maxWords = $workflow->getMetadataStore()->getMetadata('max_num_of_words', 'draft');
$priority = $workflow->getMetadataStore()->getMetadata('priority', $aTransition);
```

## Weighted Transitions (Symfony 7.4+)

Allows multiple tokens to flow through places (Petri net semantics):

```yaml
framework:
    workflows:
        make_table:
            type: 'workflow'
            marking_store:
                type: 'method'
                property: 'marking'
            supports:
                - App\Entity\TableProject
            initial_marking: init
            places:
                - init
                - prepare_leg
                - prepare_top
                - stopwatch_running
                - leg_created
                - top_created
                - finished
            transitions:
                start:
                    from: init
                    to:
                        - place: prepare_leg
                          weight: 4
                        - place: prepare_top
                          weight: 1
                        - place: stopwatch_running
                          weight: 1
                build_leg:
                    from: prepare_leg
                    to: leg_created
                join:
                    from:
                        - place: leg_created
                          weight: 4
                        - top_created
                        - stopwatch_running
                    to: finished
```

### Programmatic Arc Definition

```php
use Symfony\Component\Workflow\Arc;
use Symfony\Component\Workflow\Definition;
use Symfony\Component\Workflow\Transition;
use Symfony\Component\Workflow\Workflow;

$definition = new Definition(
    ['init', 'prepare_leg', 'prepare_top', 'stopwatch_running', 'leg_created', 'top_created', 'finished'],
    [
        new Transition('start', 'init', [
            new Arc('prepare_leg', 4),
            new Arc('prepare_top', 1),
            'stopwatch_running',
        ]),
        new Transition('build_leg', 'prepare_leg', 'leg_created'),
        new Transition('build_top', 'prepare_top', 'top_created'),
        new Transition('join', [
            new Arc('leg_created', 4),
            'top_created',
            'stopwatch_running',
        ], 'finished'),
    ]
);

$workflow = new Workflow($definition);
$workflow->apply($subject, 'start');
$workflow->apply($subject, 'build_leg');
```

## Twig Functions

```twig
{# Check if transition is possible #}
{% if workflow_can(post, 'publish') %}
    <a href="...">Publish</a>
{% endif %}

{# Specify workflow name (required if object has multiple workflows) #}
{% if workflow_can(post, 'publish', 'blog_publishing') %}
    <a href="...">Publish</a>
{% endif %}

{# Get all enabled transitions #}
{% for transition in workflow_transitions(post) %}
    <a href="...">{{ transition.name }}</a>
{% endfor %}

{# Get specific transition #}
{% set transition = workflow_transition(post, 'publish') %}

{# Get marked places #}
{% if 'reviewed' in workflow_marked_places(post) %}
    <span class="label">Reviewed</span>
{% endif %}

{# Check if has marked place #}
{% if workflow_has_marked_place(post, 'reviewed') %}
    <p>This post is ready for review.</p>
{% endif %}

{# Get transition blockers #}
{% for blocker in workflow_transition_blockers(post, 'publish') %}
    <span class="error">{{ blocker.message }}</span>
{% endfor %}

{# Get workflow metadata #}
{% set title = workflow_metadata(post, 'title') %}
{% set maxWords = workflow_metadata(post, 'max_num_of_words', 'draft') %}
```

## Debug Commands

```bash
# View all workflow configuration
php bin/console config:dump-reference framework workflows

# Dump workflow visualization (DOT format)
php bin/console workflow:dump blog_publishing

# Dump with marking
php bin/console workflow:dump blog_publishing --marking draft

# Dump as PlantUML
php bin/console workflow:dump blog_publishing --dump-format=puml

# Debug autowiring
php bin/console debug:autowiring workflow
```

## Key Classes Reference

### Workflow Classes

| Class | Description |
|-------|-------------|
| `Workflow` | Main workflow class |
| `StateMachine` | State machine implementation |
| `WorkflowInterface` | Primary interface |
| `Definition` | Workflow definition (places + transitions) |
| `DefinitionBuilder` | Builder for creating definitions |
| `Transition` | Represents a transition |
| `Arc` | Weighted arc for Petri net workflows |
| `Marking` | Represents current state(s) |
| `Registry` | Workflow registry for multiple workflows |

### Marking Store Classes

| Class | Description |
|-------|-------------|
| `MarkingStoreInterface` | Interface for marking stores |
| `MethodMarkingStore` | Uses getter/setter methods |

### Event Classes

| Class | Description |
|-------|-------------|
| `Event` | Base event class |
| `GuardEvent` | For guard validation |
| `LeaveEvent` | When leaving a place |
| `TransitionEvent` | During transition |
| `EnterEvent` | When entering a place (before marking) |
| `EnteredEvent` | After entering a place (after marking) |
| `CompletedEvent` | When transition completes |
| `AnnounceEvent` | When new transition becomes available |

### Blocker Classes

| Class | Description |
|-------|-------------|
| `TransitionBlocker` | Represents a blocking reason |
| `TransitionBlockerList` | Collection of blockers |

### Attribute Classes (Symfony 7.1+)

| Attribute | Description |
|-----------|-------------|
| `#[AsGuardListener]` | Register guard event listener |
| `#[AsLeaveListener]` | Register leave event listener |
| `#[AsTransitionListener]` | Register transition event listener |
| `#[AsEnterListener]` | Register enter event listener |
| `#[AsEnteredListener]` | Register entered event listener |
| `#[AsCompletedListener]` | Register completed event listener |
| `#[AsAnnounceListener]` | Register announce event listener |

## Best Practices

1. **Use state machines for simple sequential processes** where an object can only be in one state at a time.

2. **Use workflows for complex processes** where an object might be in multiple states simultaneously.

3. **Keep guards lightweight** - heavy validation in guards can impact performance since they run on every `can()` check.

4. **Use metadata for configuration** - store business rules in metadata rather than hardcoding in event listeners.

5. **Disable unnecessary events** - if you don't need announce events, disable them for better performance.

6. **Use attributes for event listeners** - cleaner and more maintainable than manual subscriber registration.

7. **Test workflows** - write tests that verify transition rules and guard logic.

## External Resources

- **Documentation**: https://symfony.com/doc/7.4/workflow.html
- **Component Documentation**: https://symfony.com/doc/7.4/components/workflow.html
- **GitHub Repository**: https://github.com/symfony/workflow
- **MIT License**: https://github.com/symfony/workflow/blob/8.1/LICENSE
