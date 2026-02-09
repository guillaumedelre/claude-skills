---
name: "symfony-7-4-intl"
description: "Symfony 7.4 Intl component reference. Triggers on: Intl, internationalization, locales, countries, currencies, languages, timezones, ICU, number formatting, Locale, Scripts, i18n, localization"
---

# Symfony 7.4 Intl Component

## Overview

The Symfony Intl component provides access to localization data from the ICU library. It exposes static API classes for querying languages, scripts, countries, currencies, locales, and timezones with locale-aware name translation.

## Quick Reference

| Class | Purpose | Standards |
|-------|---------|-----------|
| `Languages` | Language names and codes | ISO 639-1 (alpha-2), ISO 639-2 (alpha-3) |
| `Scripts` | Writing system names | Unicode ISO 15924 |
| `Countries` | Country names and codes | ISO 3166-1 alpha-2, alpha-3, numeric |
| `Currencies` | Currency names, symbols, fractions | ISO 4217 |
| `Locales` | Locale names | ICU locale identifiers |
| `Timezones` | Timezone names and offsets | IANA timezone database |

All classes live in `Symfony\Component\Intl` namespace. All `getNames()` / `getName()` methods accept an optional locale parameter for translated output.

### Common Patterns

```php
use Symfony\Component\Intl\Countries;
use Symfony\Component\Intl\Languages;
use Symfony\Component\Intl\Currencies;
use Symfony\Component\Intl\Timezones;

// Get localized name
$name = Countries::getName('FR', 'de'); // 'Frankreich'

// List all entries (sorted by current locale)
$all = Languages::getNames();

// Validate a code
$valid = Currencies::exists('EUR'); // true

// Convert alpha-2 <-> alpha-3
$a3 = Countries::getAlpha3Code('FR'); // 'FRA'
$a2 = Countries::getAlpha2Code('FRA'); // 'FR'
```

### Exception Handling

All lookup methods throw `Symfony\Component\Intl\Exception\MissingResourceException` for invalid codes.

## Full Documentation
- Documentation: https://symfony.com/doc/7.4/components/intl.html
- GitHub: https://github.com/symfony/intl
- Detailed API reference: see `references/intl.md`
