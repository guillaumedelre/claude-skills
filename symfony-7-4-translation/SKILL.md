---
name: "symfony-7-4-translation"
description: "Symfony 7.4 Translation component reference for internationalization with ICU MessageFormat, XLIFF/YAML/PHP catalogues, async translation providers, and locale management. Use when translating strings, configuring locales, generating translations, or integrating Crowdin/Loco/Lokalise/Phrase. Triggers on: Translation, TranslatorInterface, TranslatorBagInterface, LocaleAwareInterface, trans, translation:extract, translation:pull, translation:push, lint:translations, t() helper, TranslatableInterface, TranslatableMessage, ICU MessageFormat, message catalogues, translation.yaml, fallback locales, _locale route, LocaleSwitcher, default_locale, framework.translator, translation_provider, Crowdin, Loco, Lokalise, PhraseProvider, configureTranslations, XLIFF, YAML messages, intl-icu domain, plural messages, ChoiceMessageFormatter, debug:translation."
---

# Symfony 7.4 Translation Component

GitHub: https://github.com/symfony/translation
Docs: https://symfony.com/doc/7.4/translation.html

## Installation

```bash
composer require symfony/translation
```

## Quick Reference

### Configuration

```yaml
# config/packages/translation.yaml
framework:
    default_locale: en
    translator:
        default_path: '%kernel.project_dir%/translations'
        fallbacks: [en]
        # Async providers (optional)
        providers:
            crowdin:
                dsn: '%env(CROWDIN_DSN)%'
                domains: ['messages']
                locales: ['en', 'fr']
```

### Translation Files

```
translations/
├── messages.en.yaml
├── messages.fr.yaml
├── messages+intl-icu.fr.yaml      # ICU MessageFormat
├── validators.fr.yaml             # Specific domain
└── security.fr.xliff
```

`messages.fr.yaml`:

```yaml
welcome: Bienvenue
hello: 'Bonjour, %name%!'
errors:
    not_found: Élément introuvable
```

ICU domain (`messages+intl-icu.fr.yaml`):

```yaml
items.count: |
    {count, plural,
        =0 {Aucun élément}
        one {Un élément}
        other {# éléments}
    }
```

### Basic Usage

```php
use Symfony\Contracts\Translation\TranslatorInterface;

class GreetingService
{
    public function __construct(private TranslatorInterface $translator) {}

    public function greet(string $name, string $locale = 'en'): string
    {
        return $this->translator->trans('hello', ['%name%' => $name], domain: 'messages', locale: $locale);
    }
}
```

### TranslatableMessage (Lazy Translation)

```php
use Symfony\Component\Translation\TranslatableMessage;
use function Symfony\Component\Translation\t;

$message = new TranslatableMessage('hello', ['%name%' => 'Alice'], 'messages');
// Or shorthand:
$message = t('hello', ['%name%' => 'Alice']);

// Render later (e.g. in Twig or via translator):
$translator->trans($message, locale: 'fr');
```

`t()` is the preferred form in attribute defaults, exceptions, and DTOs because it defers translation to render time.

### In Twig

```twig
{{ 'hello'|trans({'%name%': name}) }}
{{ 'errors.not_found'|trans({}, 'messages', 'fr') }}
{% trans with {'%name%': name} %}hello{% endtrans %}
{{ message|trans }}                {# When message is TranslatableMessage #}
```

### ICU MessageFormat

Pluralization, ordinals, gender, dates - all in a single grammar.

```yaml
# messages+intl-icu.fr.yaml
notifications: |
    {count, plural,
        =0 {Aucune notification}
        one {1 notification}
        other {# notifications}
    }

ordinal: |
    {place, selectordinal,
        one {Vous êtes #er}
        two {Vous êtes #d}
        few {Vous êtes #e}
        other {Vous êtes #e}
    }

greeting: |
    {gender, select,
        female {Bienvenue Madame {name}}
        male {Bienvenue Monsieur {name}}
        other {Bienvenue {name}}
    }
```

```php
$translator->trans('notifications', ['count' => 3], 'messages');     // "3 notifications"
$translator->trans('greeting', ['gender' => 'female', 'name' => 'Alice']);
```

### Domains

```php
$translator->trans('login.fail', [], 'security');
```

Files: `security.en.yaml`, `security.fr.yaml`, etc.

The `messages` domain is the default.

### Fallback Locales

```yaml
framework:
    translator:
        fallbacks: ['en']
```

If `messages.fr.yaml` lacks a key, `messages.en.yaml` is consulted.

### Locale via Route

```yaml
# config/routes.yaml
homepage:
    path: /{_locale}/
    requirements: { _locale: 'en|fr|de' }
    defaults: { _controller: App\Controller\HomeController::index }
```

The request's `_locale` attribute is auto-applied via `LocaleListener`.

### LocaleSwitcher (Programmatic)

```php
use Symfony\Component\Translation\LocaleSwitcher;

class ReportGenerator
{
    public function __construct(private LocaleSwitcher $localeSwitcher) {}

    public function generateInLocale(string $locale): string
    {
        return $this->localeSwitcher->runWithLocale($locale, function () {
            return $this->renderReport();   // Uses $locale during this call
        });
    }
}
```

`$switcher->setLocale('fr')`, `$switcher->reset()` for fine control. The `LocaleAware` services in the container are notified automatically.

### Translating Validation Constraints

Constraint messages are pulled from the `validators` domain. Override:

```yaml
# translations/validators.fr.yaml
This value should not be blank.: Cette valeur ne peut pas être vide.
```

### TranslatableInterface (Custom Objects)

```php
use Symfony\Contracts\Translation\TranslatableInterface;
use Symfony\Contracts\Translation\TranslatorInterface;

final class OrderStatus implements TranslatableInterface
{
    public function __construct(public readonly string $status) {}

    public function trans(TranslatorInterface $translator, ?string $locale = null): string
    {
        return $translator->trans('order.status.'.$this->status, locale: $locale);
    }
}
```

### Extracting Translations

```bash
# From Twig + PHP code
php bin/console translation:extract fr --force --format=yaml

# From specific paths
php bin/console translation:extract fr --domain=messages --force --as-tree=4
```

`--force` writes to translation files; `--as-tree=N` keeps nested YAML up to N levels.

### Async Providers

```bash
php bin/console translation:pull crowdin --force
php bin/console translation:push crowdin --force --delete-missing
```

Providers configured under `framework.translator.providers`. Each requires its bridge package (`symfony/crowdin-translation-provider`, etc.).

### Linting

```bash
php bin/console lint:translations
php bin/console lint:xliff translations
php bin/console lint:yaml translations
```

### Debugging

```bash
php bin/console debug:translation fr [bundle-or-domain]
```

Shows which keys are unused, missing, or fallback-resolved.

## Helper: t() Function

```php
use function Symfony\Component\Translation\t;

throw new \DomainException(t('user.not_found', ['%id%' => $id]));    // Throws TranslatableMessage
```

In Twig: `{{ exception.message|trans }}` translates lazily.

## Full Documentation

For ICU grammar in depth, custom loaders, MessageCatalogue API, providers config, and pseudo-localization: see [references/translation.md](references/translation.md).
