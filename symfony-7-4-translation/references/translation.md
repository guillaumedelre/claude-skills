# Symfony 7.4 Translation - Complete Reference

## Table of Contents

- [Architecture](#architecture)
- [Catalogue Formats](#catalogue-formats)
- [Loading Order and Fallbacks](#loading-order-and-fallbacks)
- [ICU MessageFormat Grammar](#icu-messageformat-grammar)
- [Custom Loaders](#custom-loaders)
- [MessageCatalogue API](#messagecatalogue-api)
- [Translatable Objects](#translatable-objects)
- [Locale in HTTP Request](#locale-in-http-request)
- [LocaleSwitcher](#localeswitcher)
- [Async Providers](#async-providers)
- [Pseudo-localization](#pseudo-localization)
- [Pluralization (Legacy)](#pluralization-legacy)
- [Twig Integration](#twig-integration)
- [Debug Commands](#debug-commands)

---

## Architecture

`Translator` (in `Symfony\Component\Translation`) implements `TranslatorInterface`, `TranslatorBagInterface`, `LocaleAwareInterface`, `WarmableInterface`.

Resources are loaded into a `MessageCatalogue` per locale. Catalogues are cached in `var/cache/<env>/translations/` as compiled PHP.

Key services:
- `translator` (the Translator).
- `translation.loader.xliff`, `translation.loader.yaml`, `translation.loader.php`, `translation.loader.json`, `translation.loader.csv`, `translation.loader.po`, `translation.loader.qt`, `translation.loader.res`, `translation.loader.ini`.
- `translation.reader`, `translation.writer` (for extract/dump commands).
- `translation.locale_switcher`.
- `translation.provider_collection` (async providers).

## Catalogue Formats

| Extension | Loader | Notes |
|-----------|--------|-------|
| `.yaml` / `.yml` | YamlFileLoader | Most common |
| `.xliff` / `.xlf` | XliffFileLoader | Industry standard for translators |
| `.php` | PhpFileLoader | Returns array |
| `.json` | JsonFileLoader | Flat key/value |
| `.csv` | CsvFileLoader | First column = key, second = value |
| `.po` / `.mo` | PoFileLoader / MoFileLoader | Gettext |
| `.qt` | QtFileLoader | Qt TS XML |
| `.ini` | IniFileLoader | Section-less ini |

### File Naming Convention

`{domain}.{locale}.{format}` e.g. `messages.fr.yaml`, `validators.en_GB.xliff`.

### ICU Domain

Append `+intl-icu` to the domain to enable ICU MessageFormat:

```
messages+intl-icu.fr.yaml
```

Files in `messages+intl-icu` take precedence over `messages` for the same key in the same locale.

### XLIFF 1.2 Example

```xml
<?xml version="1.0"?>
<xliff version="1.2" xmlns="urn:oasis:names:tc:xliff:document:1.2">
    <file source-language="en" target-language="fr" datatype="plaintext" original="messages">
        <body>
            <trans-unit id="1">
                <source>welcome</source>
                <target>Bienvenue</target>
                <note>Used on landing page</note>
            </trans-unit>
        </body>
    </file>
</xliff>
```

### XLIFF 2.0

```xml
<xliff xmlns="urn:oasis:names:tc:xliff:document:2.0" version="2.0" srcLang="en" trgLang="fr">
    <file id="messages">
        <unit id="welcome">
            <segment>
                <source>welcome</source>
                <target>Bienvenue</target>
            </segment>
        </unit>
    </file>
</xliff>
```

## Loading Order and Fallbacks

For locale `fr_BE` with fallbacks `[en]`, the translator queries:

1. `messages+intl-icu.fr_BE.*`
2. `messages.fr_BE.*`
3. `messages+intl-icu.fr.*`
4. `messages.fr.*`
5. `messages+intl-icu.en.*`
6. `messages.en.*`

If still missing, returns the key itself.

## ICU MessageFormat Grammar

Full grammar: https://unicode-org.github.io/icu/userguide/format_parse/messages/

### Plural

```
{count, plural,
    =0 {empty}
    =1 {single}
    one {# item}
    few {# items}
    many {# items}
    other {# items}
}
```

CLDR plural categories: `zero`, `one`, `two`, `few`, `many`, `other`. `=N` matches exact value.

### Selectordinal

```
{place, selectordinal,
    one {#st}
    two {#nd}
    few {#rd}
    other {#th}
}
```

### Select

```
{gender, select,
    female {Madame}
    male {Monsieur}
    other {Mx.}
}
```

### Number / Date / Time

```
{amount, number, currency}
{amount, number, percent}
{amount, number, ::compact-short}
{date, date, ::yMMMMd}
{time, time, short}
{when, date, ::yMMMd, hh:mm}
```

### Nesting

```
{count, plural,
    one {{name}'s photo}
    other {{name}'s {count} photos}
}
```

### Quoting

Use single quotes to escape literal `{` `}`:

```
'{not a placeholder}'
```

## Custom Loaders

```php
use Symfony\Component\Translation\Loader\LoaderInterface;
use Symfony\Component\Translation\MessageCatalogue;
use Symfony\Component\Config\Resource\FileResource;

final class TomlLoader implements LoaderInterface
{
    public function load(mixed $resource, string $locale, string $domain = 'messages'): MessageCatalogue
    {
        $messages = $this->parseToml($resource);
        $catalogue = new MessageCatalogue($locale, [$domain => $messages]);
        $catalogue->addResource(new FileResource($resource));
        return $catalogue;
    }
}
```

Tag with `translation.loader`:

```yaml
services:
    App\Translation\TomlLoader:
        tags: [{ name: translation.loader, alias: toml }]
```

## MessageCatalogue API

```php
$catalogue = $translator->getCatalogue('fr');

$catalogue->all();                          // All messages by domain
$catalogue->all('messages');                // ['key' => 'translation', ...]
$catalogue->has('welcome');
$catalogue->get('welcome', 'messages');
$catalogue->getDomains();
$catalogue->getLocale();
$catalogue->getFallbackCatalogue();
$catalogue->add(['greet' => 'Salut'], 'messages');
$catalogue->set('key', 'value', 'domain');
```

`MessageCatalogueInterface` (read) vs `MessageCatalogue` (write).

## Translatable Objects

```php
use Symfony\Contracts\Translation\TranslatableInterface;
use Symfony\Contracts\Translation\TranslatorInterface;

enum OrderStatus: string implements TranslatableInterface
{
    case Pending = 'pending';
    case Shipped = 'shipped';
    case Delivered = 'delivered';

    public function trans(TranslatorInterface $translator, ?string $locale = null): string
    {
        return $translator->trans('order.status.'.$this->value, locale: $locale);
    }
}
```

`TranslatableMessage` (concrete) is the workhorse:

```php
new TranslatableMessage($id, $parameters, $domain);
```

Implements `TranslatableInterface`. The translator detects it and pulls translations lazily.

## Locale in HTTP Request

`LocaleListener` (subscriber on `kernel.request`) sets locale from:

1. `_locale` request attribute (set by router from `{_locale}` placeholder).
2. `Request::setLocale()` if called manually.
3. Falls back to `default_locale` framework setting.

Stickiness across requests:

```yaml
framework:
    session:
        enabled: true
    set_locale_from_accept_language: true   # Use Accept-Language header
```

```php
// Manual stickiness via session
public function changeLocale(Request $request, string $locale): Response
{
    $request->getSession()->set('_locale', $locale);
    return $this->redirectToRoute('home');
}

// Listener:
public function onKernelRequest(RequestEvent $event): void
{
    $request = $event->getRequest();
    if ($locale = $request->getSession()->get('_locale')) {
        $request->setLocale($locale);
    }
}
```

## LocaleSwitcher

```php
use Symfony\Component\Translation\LocaleSwitcher;

class CalendarRenderer
{
    public function __construct(private LocaleSwitcher $switcher) {}

    public function renderInLocale(Calendar $cal, string $locale): string
    {
        return $this->switcher->runWithLocale($locale, fn () => $this->doRender($cal));
    }
}
```

`runWithLocale()` notifies all `LocaleAwareInterface` services for the duration.

## Async Providers

Bridge packages:

```bash
composer require symfony/crowdin-translation-provider
composer require symfony/loco-translation-provider
composer require symfony/lokalise-translation-provider
composer require symfony/phrase-translation-provider
```

Configure:

```yaml
framework:
    translator:
        providers:
            crowdin:
                dsn: '%env(CROWDIN_DSN)%'
                domains: ['messages', 'validators']
                locales: ['en', 'fr', 'de']
            loco:
                dsn: '%env(LOCO_DSN)%'
                domains: ['messages']
                locales: ['en', 'fr']
```

DSN samples:
- `crowdin://API_TOKEN@PROJECT_ID:OPTIONAL_BRANCH`
- `loco://API_KEY@default`
- `lokalise://API_KEY@PROJECT_ID`
- `phrase://USER:TOKEN@default?project=PROJECT_ID`

CLI:

```bash
php bin/console translation:pull crowdin --force --domains=messages
php bin/console translation:push crowdin --force --delete-missing --domains=messages
```

## Pseudo-localization

```yaml
framework:
    translator:
        pseudo_localization:
            accents: true              # ŝômê wôrds
            expansion_factor: 1.4      # Stretch text
            brackets: true             # [text]
            parse_html: true
            localizable_html_attributes: ['title', 'alt']
```

Useful for spotting missing translations and layout overflow.

## Pluralization (Legacy)

Pre-Symfony 4.2 pluralization used `transChoice()`. Symfony 7.x removed `transChoice()`; use ICU MessageFormat (`+intl-icu` domain) instead.

If migrating legacy `|` pipe syntax (`'apple|apples'`), convert to ICU plural blocks during extraction.

## Twig Integration

Twig filters/tags:

```twig
{{ 'welcome'|trans }}
{{ 'hello'|trans({'%name%': name}) }}
{{ 'errors.404'|trans({}, 'security', 'fr') }}

{% trans with {'%name%': name} from 'messages' into 'fr' %}
    hello
{% endtrans %}

{% trans_default_domain 'security' %}
{{ 'login.fail'|trans }}            {# Uses 'security' domain #}
```

For TranslatableMessage:

```twig
{{ message|trans }}
```

## Debug Commands

```bash
php bin/console debug:translation fr [bundle-or-path] [domain]
```

Output legend:
- `xxx` (red): missing in `fr`, missing in fallback.
- `xxx` (yellow): missing in `fr`, present in fallback.
- `xxx` (green): present in `fr`.
- `unused`: present but not referenced in templates/PHP.

```bash
php bin/console translation:extract fr --force --as-tree=4
php bin/console lint:translations
php bin/console lint:yaml translations
php bin/console lint:xliff translations
```
