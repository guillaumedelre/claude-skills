# Symfony 7.4 Mime - Complete Reference

## Table of Contents

- [Installation](#installation)
- [Email Class (High-Level API)](#email-class-high-level-api)
- [Address Class](#address-class)
- [File Attachments](#file-attachments)
- [Embedded Images](#embedded-images)
- [Message Class (Low-Level API)](#message-class-low-level-api)
- [Message Parts](#message-parts)
- [Headers](#headers)
- [Twig Integration](#twig-integration)
- [Serializing Messages](#serializing-messages)
- [MIME Types Utilities](#mime-types-utilities)
- [Custom MIME Type Guesser](#custom-mime-type-guesser)

---

## Installation

```bash
composer require symfony/mime
```

## Email Class (High-Level API)

The `Email` class provides a chainable interface for composing email messages:

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

**Priority Constants:**
- `Email::PRIORITY_HIGHEST` (1)
- `Email::PRIORITY_HIGH` (2)
- `Email::PRIORITY_NORMAL` (3)
- `Email::PRIORITY_LOW` (4)
- `Email::PRIORITY_LOWEST` (5)

**Note:** The Mime component only creates messages. Use the Mailer component to send them.

## Address Class

Use `Address` objects for more control over sender/recipient information:

```php
use Symfony\Component\Mime\Address;

$email = (new Email())
    ->from(new Address('fabien@symfony.com', 'Fabien Potencier'))
    ->to(
        new Address('foo@example.com', 'John Doe'),
        new Address('bar@example.com', 'Jane Doe')
    )
    ->replyTo(new Address('support@example.com', 'Support Team'))
;
```

### Multiple Recipients

```php
$email
    ->to('first@example.com', 'second@example.com')
    ->addTo('third@example.com')
    ->cc('cc1@example.com', 'cc2@example.com')
    ->addCc('cc3@example.com')
;
```

## File Attachments

### Attaching from File Path

```php
$email = (new Email())
    ->from('sender@example.com')
    ->to('recipient@example.com')
    ->subject('Document Attached')
    ->text('Please find the document attached.')
    ->attachFromPath('/path/to/documents/terms.pdf')
    ->attachFromPath('/path/to/documents/privacy.pdf', 'Privacy Policy.pdf')
    ->attachFromPath('/path/to/file.txt', 'custom-name.txt', 'text/plain')
;
```

### Attaching from Content

```php
$email->attach($pdfContent, 'document.pdf', 'application/pdf');
```

## Embedded Images

Embed images using Content-ID (cid) references:

```php
$email = (new Email())
    ->from('sender@example.com')
    ->to('recipient@example.com')
    ->subject('Newsletter')
    ->embedFromPath('/path/to/images/logo.png', 'logo')
    ->embedFromPath('/path/to/images/banner.jpg', 'banner')
    ->html('
        <img src="cid:logo"/>
        <h1>Welcome to our Newsletter!</h1>
        <img src="cid:banner"/>
    ')
;
```

### Embedding from Content

```php
$email->embed($imageContent, 'logo', 'image/png');
```

## Message Class (Low-Level API)

For absolute control over email structure, use the `Message` class:

### MIME Structure Hierarchy

```
multipart/mixed
├── multipart/related
│   ├── multipart/alternative
│   │   ├── text/plain
│   │   └── text/html
│   └── image/png (embedded)
└── application/pdf (attachment)
```

### Basic Message Creation

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
$htmlContent = new TextPart('<h1>Lorem ipsum</h1> <p>...</p>', null, 'html');
$body = new AlternativePart($textContent, $htmlContent);

$email = new Message($headers, $body);
```

### Complex Message with Embedded Images and Attachments

```php
use Symfony\Component\Mime\Part\DataPart;
use Symfony\Component\Mime\Part\Multipart\AlternativePart;
use Symfony\Component\Mime\Part\Multipart\MixedPart;
use Symfony\Component\Mime\Part\Multipart\RelatedPart;
use Symfony\Component\Mime\Part\TextPart;

// Create embedded image
$embeddedImage = new DataPart(fopen('/path/to/images/logo.png', 'r'), null, 'image/png');
$imageCid = $embeddedImage->getContentId();

// Create attachment
$attachedFile = new DataPart(fopen('/path/to/documents/terms.pdf', 'r'), 'Terms.pdf', 'application/pdf');

// Create text parts
$textContent = new TextPart('Lorem ipsum...');
$htmlContent = new TextPart(sprintf(
    '<img src="cid:%s"/> <h1>Lorem ipsum</h1> <p>...</p>',
    $imageCid
), null, 'html');

// Assemble the structure
$bodyContent = new AlternativePart($textContent, $htmlContent);
$bodyWithImages = new RelatedPart($bodyContent, $embeddedImage);
$fullBody = new MixedPart($bodyWithImages, $attachedFile);

$email = new Message($headers, $fullBody);
```

## Message Parts

### TextPart

```php
use Symfony\Component\Mime\Part\TextPart;

// Plain text
$textPart = new TextPart('Hello World');

// HTML
$htmlPart = new TextPart('<h1>Hello</h1>', null, 'html');

// With specific charset
$textPart = new TextPart('Content', 'utf-8', 'plain');
```

### DataPart

```php
use Symfony\Component\Mime\Part\DataPart;

// From file handle
$part = new DataPart(fopen('/path/to/file.pdf', 'r'), 'filename.pdf', 'application/pdf');

// From path
$part = DataPart::fromPath('/path/to/file.pdf');
$part = DataPart::fromPath('/path/to/file.pdf', 'custom-name.pdf', 'application/pdf');
```

### Multipart Types

| Class | Purpose |
|-------|---------|
| `AlternativePart` | Multiple representations of same content (text + HTML) |
| `MixedPart` | Message body + attachments |
| `RelatedPart` | Content with inline/embedded resources |
| `DigestPart` | Collection of messages |
| `FormDataPart` | Form data (for HTTP, not email) |

## Headers

### Creating Headers

```php
use Symfony\Component\Mime\Header\Headers;

$headers = new Headers();

// Mailbox headers (From, To, Cc, Bcc, Reply-To)
$headers->addMailboxListHeader('From', ['sender@example.com']);
$headers->addMailboxListHeader('To', [
    'recipient@example.com',
    'Another <another@example.com>'
]);

// Text headers
$headers->addTextHeader('Subject', 'My Subject');
$headers->addTextHeader('X-Custom-Header', 'Custom Value');

// Date header
$headers->addDateHeader('Date', new \DateTimeImmutable());

// ID headers
$headers->addIdHeader('Message-ID', 'unique-id@example.com');

// Parameterized headers
$headers->addParameterizedHeader('Content-Type', 'text/plain', [
    'charset' => 'utf-8'
]);
```

### Getting and Modifying Headers

```php
$headers->get('Subject')->setBody('New Subject');
$headers->remove('X-Custom-Header');
$headers->has('Reply-To'); // bool
```

## Twig Integration

### TemplatedEmail

```php
use Symfony\Bridge\Twig\Mime\TemplatedEmail;

$email = (new TemplatedEmail())
    ->from('sender@example.com')
    ->to('recipient@example.com')
    ->subject('Welcome!')
    ->htmlTemplate('emails/welcome.html.twig')
    ->context([
        'username' => 'John',
        'expiration_date' => new \DateTimeImmutable('+7 days'),
    ])
;
```

### Template Example (emails/welcome.html.twig)

```twig
<h1>Welcome {{ username }}!</h1>
<p>Your account expires on {{ expiration_date|date('Y-m-d') }}.</p>
```

### Rendering with BodyRenderer

```php
use Symfony\Bridge\Twig\Mime\BodyRenderer;
use Twig\Environment;
use Twig\Loader\FilesystemLoader;

$loader = new FilesystemLoader(__DIR__.'/templates');
$twig = new Environment($loader);

$renderer = new BodyRenderer($twig);
$renderer->render($email);
```

### CSS Inlining

Install the extension:

```bash
composer require twig/cssinliner-extra
```

Enable it:

```php
use Twig\Extra\CssInliner\CssInlinerExtension;

$twig->addExtension(new CssInlinerExtension());
```

Use in templates:

```twig
{% apply inline_css %}
<style>
    h1 { color: blue; }
</style>
<h1>Hello</h1>
{% endapply %}
```

Or with external stylesheets:

```twig
{% apply inline_css(source('@styles/email.css')) %}
<h1>Hello</h1>
{% endapply %}
```

## Serializing Messages

Email messages can be serialized for storage or queuing:

```php
$email = (new Email())
    ->from('fabien@symfony.com')
    ->to('recipient@example.com')
    ->subject('Test')
    ->text('Content')
;

$serializedEmail = serialize($email);

// Later, recreate the message
$email = unserialize($serializedEmail);
```

### Using RawMessage

```php
use Symfony\Component\Mime\RawMessage;

// Create from raw MIME content
$rawContent = "From: sender@example.com\r\nTo: recipient@example.com\r\n...";
$message = new RawMessage($rawContent);
```

## MIME Types Utilities

### Getting Extensions and MIME Types

```php
use Symfony\Component\Mime\MimeTypes;

$mimeTypes = new MimeTypes();

// Get extensions for a MIME type
$exts = $mimeTypes->getExtensions('application/javascript');
// ['js', 'jsm', 'mjs']

$exts = $mimeTypes->getExtensions('image/jpeg');
// ['jpeg', 'jpg', 'jpe']

// Get MIME types for an extension
$types = $mimeTypes->getMimeTypes('js');
// ['application/javascript', 'application/x-javascript', 'text/javascript']

$types = $mimeTypes->getMimeTypes('apk');
// ['application/vnd.android.package-archive']
```

### Guessing MIME Type from File

```php
use Symfony\Component\Mime\MimeTypes;

$mimeTypes = new MimeTypes();

// Guesses based on file content (not extension)
$mimeType = $mimeTypes->guessMimeType('/path/to/image.gif');
// 'image/gif'

$mimeType = $mimeTypes->guessMimeType('/path/to/document.pdf');
// 'application/pdf'
```

**Note:** Install the PHP `fileinfo` extension for best performance.

### Singleton Access

```php
$mimeTypes = MimeTypes::getDefault();
```

## Custom MIME Type Guesser

Implement `MimeTypeGuesserInterface` for custom guessing logic:

```php
namespace App\Mime;

use Symfony\Component\Mime\MimeTypeGuesserInterface;

class CustomMimeTypeGuesser implements MimeTypeGuesserInterface
{
    public function isGuesserSupported(): bool
    {
        // Return false to disable this guesser
        return true;
    }

    public function guessMimeType(string $path): ?string
    {
        // Custom detection logic
        $extension = pathinfo($path, PATHINFO_EXTENSION);

        if ($extension === 'custom') {
            return 'application/x-custom';
        }

        // Return null to let other guessers try
        return null;
    }
}
```

### Registering Custom Guesser

In Symfony applications, tag as `mime.mime_type_guesser`:

```yaml
# config/services.yaml
services:
    App\Mime\CustomMimeTypeGuesser:
        tags: ['mime.mime_type_guesser']
```

Or register manually:

```php
$mimeTypes = new MimeTypes();
$mimeTypes->registerGuesser(new CustomMimeTypeGuesser());
```

## Key Classes Summary

| Class | Purpose |
|-------|---------|
| `Email` | High-level email creation with chainable methods |
| `Message` | Low-level message with full control |
| `RawMessage` | Message from raw MIME content |
| `TemplatedEmail` | Email with Twig template support |
| `Address` | Email address with optional name |
| `Headers` | Email header collection |
| `TextPart` | Text or HTML content part |
| `DataPart` | Binary data (attachments, embedded files) |
| `AlternativePart` | Alternative representations (text + HTML) |
| `MixedPart` | Body + attachments |
| `RelatedPart` | Content + embedded resources |
| `MimeTypes` | MIME type utilities and guessing |
