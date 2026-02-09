# Symfony 7.4 Intl Component - Complete Reference

## Installation

```bash
composer require symfony/intl
```

## Languages

```php
use Symfony\Component\Intl\Languages;

// All language names (alpha-2 keys)
$languages = Languages::getNames();        // ['ab' => 'Abkhazian', ...]
$languages = Languages::getNames('de');    // German translations

// All language names (alpha-3 keys)
$languages = Languages::getAlpha3Names();  // ['abk' => 'Abkhazian', ...]

// Single language name
$name = Languages::getName('fr');          // 'French'
$name = Languages::getName('fr', 'de');    // 'Französisch'
$name = Languages::getAlpha3Name('fra');   // 'French'

// Validation
Languages::exists('fr');                   // true
Languages::alpha3CodeExists('fra');        // true

// Code conversion
Languages::getAlpha3Code('fr');            // 'fra'
Languages::getAlpha2Code('fra');           // 'fr'
```

## Scripts

```php
use Symfony\Component\Intl\Scripts;

$scripts = Scripts::getNames();            // ['Adlm' => 'Adlam', ...]
$name = Scripts::getName('Hans');          // 'Simplified'
$name = Scripts::getName('Hans', 'de');    // 'Vereinfacht'
Scripts::exists('Hans');                   // true
```

## Countries

```php
use Symfony\Component\Intl\Countries;

// Names (alpha-2)
$countries = Countries::getNames();        // ['AF' => 'Afghanistan', ...]
$name = Countries::getName('GB');          // 'United Kingdom'
$name = Countries::getName('GB', 'de');    // 'Vereinigtes Königreich'

// Names (alpha-3)
$countries = Countries::getAlpha3Names();  // ['AFG' => 'Afghanistan', ...]
$name = Countries::getAlpha3Name('NOR');   // 'Norway'

// Validation
Countries::exists('FR');                   // true
Countries::alpha3CodeExists('FRA');        // true

// Code conversion
Countries::getAlpha3Code('FR');            // 'FRA'
Countries::getAlpha2Code('FRA');           // 'FR'

// Numeric codes (ISO 3166-1 numeric)
$codes = Countries::getNumericCodes();     // ['AD' => '020', ...]
$num = Countries::getNumericCode('FR');    // '250'
$a2 = Countries::getAlpha2FromNumeric('250'); // 'FR'
Countries::numericCodeExists('250');       // true
```

Set `SYMFONY_INTL_WITH_USER_ASSIGNED` env var to recognize user-assigned codes: `XK`, `XKK`, and `983`.

## Locales

```php
use Symfony\Component\Intl\Locales;

$locales = Locales::getNames();            // ['af' => 'Afrikaans', ...]
$name = Locales::getName('zh_Hans_MO');    // 'Chinese (Simplified, Macau SAR China)'
$name = Locales::getName('zh_Hans_MO', 'de');
Locales::exists('fr_FR');                  // true
```

## Currencies

```php
use Symfony\Component\Intl\Currencies;

// Names and symbols
$currencies = Currencies::getNames();      // ['AFN' => 'Afghan Afghani', ...]
$name = Currencies::getName('INR');        // 'Indian Rupee'
$name = Currencies::getName('INR', 'de');  // 'Indische Rupie'
$symbol = Currencies::getSymbol('INR');    // '₹'
Currencies::exists('EUR');                 // true

// Fraction digits
Currencies::getFractionDigits('INR');      // 2
Currencies::getCashFractionDigits('SEK');  // 0 (different from regular: 2)

// Rounding increments
Currencies::getRoundingIncrement('CAD');       // 0
Currencies::getCashRoundingIncrement('CAD');   // 5

// Country-currency relationships (Symfony 7.4+)
Currencies::forCountry('FR');              // ['EUR']
Currencies::forCountry('ES', legalTender: null, active: true, date: new \DateTimeImmutable('1982-01-01'));
// ['ESP', 'ESB']

Currencies::isValidInCountry('CH', 'CHF'); // true
Currencies::isValidInAnyCountry('USD', legalTender: true, active: true, date: new \DateTimeImmutable('2005-01-01'));
// true
```

## Timezones

```php
use Symfony\Component\Intl\Timezones;

// Names
$timezones = Timezones::getNames();        // ['America/Eirunepe' => 'Acre Time (Eirunepe)', ...]
$name = Timezones::getName('Africa/Nairobi');     // 'East Africa Time (Nairobi)'
$name = Timezones::getName('Africa/Nairobi', 'de'); // 'Ostafrikanische Zeit (Nairobi)'
Timezones::exists('Europe/Paris');         // true

// Country associations
Timezones::forCountryCode('CL');           // ['America/Punta_Arenas', 'America/Santiago', 'Pacific/Easter']
Timezones::getCountryCode('America/Vancouver'); // 'CA'

// Offsets (seconds)
Timezones::getRawOffset('Etc/UTC');                // 0
Timezones::getRawOffset('America/Buenos_Aires');   // -10800
Timezones::getRawOffset('Asia/Katmandu');          // 20700

// DST-aware offset at specific time
Timezones::getRawOffset('Europe/Madrid', strtotime('March 31, 2019'));  // 3600
Timezones::getRawOffset('Europe/Madrid', strtotime('April 1, 2019'));   // 7200

// GMT string for display
Timezones::getGmtOffset('Etc/UTC');                // 'GMT+00:00'
Timezones::getGmtOffset('America/Buenos_Aires');   // 'GMT-03:00'
Timezones::getGmtOffset('Asia/Katmandu');          // 'GMT+05:45'
Timezones::getGmtOffset('Europe/Madrid', strtotime('October 28, 2019'), 'ar');
// 'غرينتش+01:00'
```

## Disk Space Optimization

Compress internal ICU data files (requires zlib):

```bash
php ./vendor/symfony/intl/Resources/bin/compress
```

## Related Symfony Form Types

- `CountryType` - Country select field
- `CurrencyType` - Currency select field
- `LanguageType` - Language select field
- `LocaleType` - Locale select field
- `TimezoneType` - Timezone select field

## Exception Handling

All lookup methods throw `Symfony\Component\Intl\Exception\MissingResourceException` when an invalid code is passed.

```php
use Symfony\Component\Intl\Exception\MissingResourceException;

try {
    $name = Languages::getName('invalid');
} catch (MissingResourceException $e) {
    // handle
}
```
