---
name: "symfony-7-4-workflow"
description: "Symfony 7.4 Workflow component reference for managing workflows and finite state machines. Use when working with Workflow, state machine, transitions, places, marking, WorkflowInterface, GuardEvent, TransitionBlocker, MarkingStore, Definition, workflow events, or any workflow-related Symfony code. Triggers on: Workflow, state machine, transitions, places, marking, WorkflowInterface, GuardEvent, TransitionBlocker, MarkingStore, Definition, DefinitionBuilder, workflow.guard, workflow.transition, workflow.enter, workflow.leave, workflow.entered, workflow.completed, workflow.announce."
---

# Symfony 7.4 Workflow Component

## Overview

The Workflow component provides tools for managing a workflow or finite state machine, giving you an object-oriented way to define a process or lifecycle that your objects go through.

## Quick Reference

### Installation

```bash
composer require symfony/workflow
```

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Places** | States/stages in your process (e.g., draft, reviewed, published) |
| **Transitions** | Actions to move between places (e.g., to_review, publish) |
| **Marking** | Current state(s) of an object |
| **Marking Store** | Persists the current state to your object |
| **Definition** | Complete workflow structure (places + transitions) |

### Workflow vs State Machine

| Aspect | Workflow | State Machine |
|--------|----------|---------------|
| Type | `workflow` | `state_machine` |
| Multiple Places | Yes (object can be in multiple places) | No (single place only) |
| Marking Store | `multiple_state` | `single_state` |
| Property Type | `array` | `string` |

## Configuration

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

## Basic Usage

### Entity Implementation

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

### Using the Workflow

```php
use Symfony\Component\Workflow\WorkflowInterface;

class BlogPostController
{
    public function __construct(
        private WorkflowInterface $blogPublishingWorkflow,
    ) {
    }

    public function review(BlogPost $post): void
    {
        // Check if transition is possible
        if ($this->blogPublishingWorkflow->can($post, 'to_review')) {
            // Apply transition
            $this->blogPublishingWorkflow->apply($post, 'to_review');
        }

        // Get all enabled transitions
        $transitions = $this->blogPublishingWorkflow->getEnabledTransitions($post);

        // Get current marking
        $marking = $this->blogPublishingWorkflow->getMarking($post);
    }
}
```

### Injecting with Target Attribute

```php
use Symfony\Component\DependencyInjection\Attribute\Target;
use Symfony\Component\Workflow\WorkflowInterface;

class MyService
{
    public function __construct(
        #[Target('blog_publishing')] private WorkflowInterface $workflow,
    ) {
    }
}
```

## Guard Events and Transition Blockers

### Guard Event Subscriber

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

### Using TransitionBlocker

```php
use Symfony\Component\Workflow\Event\GuardEvent;
use Symfony\Component\Workflow\TransitionBlocker;

public function guardPublish(GuardEvent $event): void
{
    $hourLimit = $event->getMetadata('hour_limit', $event->getTransition());

    if (date('H') > $hourLimit) {
        $event->addTransitionBlocker(
            new TransitionBlocker('Cannot publish after 8 PM.', '0')
        );
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
                    guard: "is_authenticated and subject.isReady()"
                    from: reviewed
                    to: published
```

## Event Listener Attributes

```php
use Symfony\Component\Workflow\Attribute\AsTransitionListener;
use Symfony\Component\Workflow\Attribute\AsGuardListener;
use Symfony\Component\Workflow\Attribute\AsEnteredListener;

class ArticleWorkflowListener
{
    #[AsGuardListener(workflow: 'blog_publishing', transition: 'publish')]
    public function onGuardPublish(GuardEvent $event): void
    {
        // Validate transition
    }

    #[AsTransitionListener(workflow: 'blog_publishing', transition: 'publish')]
    public function onTransition(TransitionEvent $event): void
    {
        // During transition
    }

    #[AsEnteredListener(workflow: 'blog_publishing', place: 'published')]
    public function onEnteredPublished(EnteredEvent $event): void
    {
        // After entering published state
    }
}
```

## Twig Functions

```twig
{# Check if transition is possible #}
{% if workflow_can(post, 'publish') %}
    <a href="...">Publish</a>
{% endif %}

{# Get all enabled transitions #}
{% for transition in workflow_transitions(post) %}
    <a href="...">{{ transition.name }}</a>
{% endfor %}

{# Check marked places #}
{% if workflow_has_marked_place(post, 'reviewed') %}
    <span>Ready for review</span>
{% endif %}

{# Get transition blockers #}
{% for blocker in workflow_transition_blockers(post, 'publish') %}
    <span class="error">{{ blocker.message }}</span>
{% endfor %}
```

## Debug Commands

```bash
# Dump workflow visualization
php bin/console workflow:dump blog_publishing

# Debug autowiring
php bin/console debug:autowiring workflow
```

## Full Documentation
- Docs: https://symfony.com/doc/7.4/workflow.html
- **Component Docs**: https://symfony.com/doc/7.4/components/workflow.html
- GitHub: https://github.com/symfony/workflow
- **Full Reference**: See `references/workflow.md` for complete API documentation
