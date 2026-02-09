---
name: "symfony-7-4-mime"
description: "Symfony 7.4 Mime component reference for creating and manipulating MIME messages. Use when working with email messages, MIME types, file attachments, embedded images, or email content. Triggers on: Mime, MIME types, email messages, Email class, multipart, attachments, MimeTypes, Address, TemplatedEmail, RawMessage, Message, Headers, embedded images, DataPart, TextPart, multipart/mixed, multipart/alternative."
---

# Symfony 7.4 Mime Component

GitHub: https://github.com/symfony/mime
Docs: https://symfony.com/doc/7.4/components/mime.html

## Quick Reference

### Creating Email Messages (High-Level API)

```php
use Symfony\Component\Mime\Email;

$email = (new Email())
    ->from('fabien@symfony.com')
    ->to('foo@example.com')
    ->cc('bar@example.com')
    ->bcc('baz@example.com')
    ->replyTo('fabien@symfony.com')
    ->priority(Email::PRIORITY_HIGH)
    ->subject('Important Notification')
    ->text('Lorem ipsum...')
    ->html('<h1>Lorem ipsum</h1> <p>...</p>')
;
```

### File Attachments

```php
$email = (new Email())
    ->from('fabien@symfony.com')
    ->to('foo@example.com')
    ->subject('Document')
    ->text('Please find attached.')
    ->attachFromPath('/path/to/documents/terms.pdf')
    ->attachFromPath('/path/to/documents/privacy.pdf', 'Privacy Policy')
;
```

### Embedded Images

```php
$email = (new Email())
    ->from('fabien@symfony.com')
    ->to('foo@example.com')
    ->subject('Newsletter')
    ->embedFromPath('/path/to/images/logo.png', 'logo')
    ->html('<img src="cid:logo"/> <h1>Welcome!</h1>')
;
```

### Using Address Objects

```php
use Symfony\Component\Mime\Address;

$email = (new Email())
    ->from(new Address('fabien@symfony.com', 'Fabien'))
    ->to(new Address('foo@example.com', 'John Doe'))
    // ...
;
```

### MIME Type Utilities

```php
use Symfony\Component\Mime\MimeTypes;

$mimeTypes = new MimeTypes();

// Get extensions for a MIME type
$exts = $mimeTypes->getExtensions('image/jpeg');
// ['jpeg', 'jpg', 'jpe']

// Get MIME types for an extension
$types = $mimeTypes->getMimeTypes('js');
// ['application/javascript', 'application/x-javascript', 'text/javascript']

// Guess MIME type from file
$mimeType = $mimeTypes->guessMimeType('/path/to/image.gif');
// 'image/gif'
```

### Low-Level Message API

```php
use Symfony\Component\Mime\Header\Headers;
use Symfony\Component\Mime\Message;
use Symfony\Component\Mime\Part\Multipart\AlternativePart;
use Symfony\Component\Mime\Part\TextPart;

$headers = (new Headers())
    ->addMailboxListHeader('From', ['fabien@symfony.com'])
    ->addMailboxListHeader('To', ['foo@example.com'])
    ->addTextHeader('Subject', 'Important Notification')
;

$textContent = new TextPart('Lorem ipsum...');
$htmlContent = new TextPart('<h1>Lorem ipsum</h1>', null, 'html');
$body = new AlternativePart($textContent, $htmlContent);

$email = new Message($headers, $body);
```

## Full Documentation

For complete details including Twig integration, CSS inlining, complex multipart structures (MixedPart, RelatedPart), serialization, custom MIME type guessers, and all header types, see [references/mime.md](references/mime.md).
