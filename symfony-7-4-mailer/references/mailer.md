# Symfony 7.4 Mailer - Complete Reference

## Table of Contents

- [DSN Reference](#dsn-reference)
- [Transports](#transports)
- [Email Anatomy](#email-anatomy)
- [Headers](#headers)
- [Envelope vs From](#envelope-vs-from)
- [Async Dispatch](#async-dispatch)
- [Twig Bridge](#twig-bridge)
- [Inky / CSS Inliner](#inky--css-inliner)
- [Signing and Encryption](#signing-and-encryption)
- [Mailer Events](#mailer-events)
- [Custom Transport](#custom-transport)
- [Testing](#testing)
- [Deliverability Tips](#deliverability-tips)

---

## DSN Reference

| Provider | DSN |
|----------|-----|
| SMTP | `smtp://USER:PASS@HOST:PORT?encryption=tls&auth_mode=login` |
| Sendmail (binary) | `sendmail://default?command=%2Fusr%2Fsbin%2Fsendmail+-bs` |
| Native (php.ini) | `native://default` |
| Null (discard) | `null://null` |
| Sendgrid API | `sendgrid+api://KEY@default` |
| Sendgrid SMTP | `sendgrid+smtp://KEY@default` |
| Mailgun API | `mailgun+api://KEY:DOMAIN@default?region=eu` |
| Mailgun SMTP | `mailgun+smtp://USER:PASS@default` |
| Amazon SES API | `ses+api://ACCESS:SECRET@default?region=eu-west-1` |
| Amazon SES SMTP | `ses+smtp://USER:PASS@default?region=eu-west-1` |
| Postmark API | `postmark+api://KEY@default` |
| Brevo API | `brevo+api://KEY@default` |
| Resend API | `resend+api://KEY@default` |
| Mailjet API | `mailjet+api://KEY:SECRET@default` |
| Mailpace API | `mailpace+api://KEY@default` |
| Scaleway API | `scaleway+api://KEY:SECRET@default?region=fr-par` |
| Mandrill API | `mandrill+api://KEY@default` |

### Failover / Round-robin

```dotenv
MAILER_DSN=failover(sendgrid+api://KEY@default postmark+api://KEY@default)
MAILER_DSN=roundrobin(smtp://a smtp://b)
```

URL-encode special characters in passwords (e.g. `@` → `%40`).

## Transports

Each transport class is in a separate package (e.g. `Symfony\Component\Mailer\Bridge\Sendgrid\Transport\SendgridApiTransport`). Auto-registered via Symfony Flex when the bridge package is installed.

### SmtpTransport

Options:
- `encryption` (`tls`, `ssl`).
- `auth_mode` (`login`, `cram-md5`, `plain`).
- `local_domain` (HELO/EHLO domain).
- `restart_threshold`, `ping_threshold` (long-lived workers).
- `connection_timeout` (default 30s).

### NativeTransport

Uses `php.ini` `sendmail_path`. Limited; lacks DSN options.

### NullTransport

Discards all messages. Useful for tests.

## Email Anatomy

```php
$email = (new Email())
    ->from(...)
    ->sender(...)            // Single Sender header
    ->to(..., ..., ...)
    ->cc(...)
    ->bcc(...)
    ->replyTo(...)
    ->returnPath(...)        // Bounce address (envelope)
    ->priority(Email::PRIORITY_HIGHEST)   // 1-5
    ->subject(...)
    ->date(new \DateTimeImmutable())
    ->text('plain body')
    ->html('<p>html</p>')
    ->attach('binary or stream', 'name.pdf', 'application/pdf')
    ->attachFromPath('/path/file.pdf', 'invoice.pdf', 'application/pdf')
    ->embed(fopen('/logo.png', 'r'), 'logo')
    ->embedFromPath('/logo.png', 'logo');
```

`Email` extends `Message`. The `Message` class is the lower-level MIME container.

### Body Types

- `text(string $body, ?string $charset = 'utf-8')`
- `html(string $body, ?string $charset = 'utf-8')`
- Both produce `multipart/alternative`. Without `text()`, only HTML is sent.

## Headers

```php
$email
    ->getHeaders()
    ->addTextHeader('X-Tenant', 'acme')
    ->addParameterizedHeader('Disposition-Notification-To', 'sender@example.com', ['custom' => 'value'])
    ->addIdHeader('Message-ID', 'msg.123@example.com')
    ->addPathHeader('Return-Path', 'bounces@example.com')
    ->addDateHeader('Date', new \DateTimeImmutable())
    ->addMailboxListHeader('Resent-To', ['carbon@example.com']);
```

Replace via `$headers->remove('X-Tenant')` then `addTextHeader`.

## Envelope vs From

The envelope (SMTP `MAIL FROM` and `RCPT TO`) defaults to `from()` and `to()/cc()/bcc()`. Override:

```php
use Symfony\Component\Mailer\Envelope;
use Symfony\Component\Mailer\SentMessage;

$mailer->send($email, new Envelope(
    new Address('bounces@example.com'),    // sender
    [new Address('actual@recipient.tld')], // recipients
));
```

Or globally:

```yaml
framework:
    mailer:
        envelope:
            sender: bounces@example.com
            recipients: ['*@example.com']   # Catch-all override (dev/test)
```

## Async Dispatch

### Setup

```yaml
framework:
    messenger:
        transports:
            mailer: '%env(MESSENGER_TRANSPORT_DSN)%'
        routing:
            'Symfony\Component\Mailer\Messenger\SendEmailMessage': mailer
```

### Pre-rendered or lazy

By default the email is rendered (templated → MIME) at send time, then serialized to messenger.

For TemplatedEmail with attachments, ensure no resources/streams are passed (they cannot be serialized). Use `attachFromPath()`/`embedFromPath()` instead.

### Worker

```bash
php bin/console messenger:consume mailer --time-limit=3600 --memory-limit=128M
```

## Twig Bridge

`composer require symfony/twig-bundle twig/cssinliner-extra twig/inky-extra`

```php
$email = (new TemplatedEmail())
    ->subject('Hello')
    ->htmlTemplate('emails/welcome.html.twig')
    ->textTemplate('emails/welcome.txt.twig')
    ->context(['name' => 'Alice']);
```

`emails/welcome.html.twig`:

```twig
<h1>Hello {{ name }}</h1>
<img src="{{ email.image('@public/logo.png', 'logo') }}">
<a href="{{ email.attach('@public/terms.pdf', 'Terms.pdf') }}">Terms</a>
```

`email.image()` and `email.attach()` integrate seamlessly with the embed/attach API.

## Inky / CSS Inliner

CSS inliner (run after Twig render):

```yaml
twig:
    paths:
        '%kernel.project_dir%/templates/email': email
```

Inlines styles automatically when `cssinliner-extra` is installed.

Inky responsive layout:

```twig
{% extends '@email/zurb_2/foundation.html.twig' %}
{% block content %}
    <row><columns>Hello!</columns></row>
{% endblock %}
```

## Signing and Encryption

### S/MIME Signing

```php
$signer = new SMimeSigner('/etc/cert.pem', '/etc/key.pem', 'passphrase', extraCerts: '/etc/chain.pem');
$signed = $signer->sign($email);
```

### S/MIME Encryption

```php
$encrypter = new SMimeEncrypter('/etc/recipient.pem', cipher: \OPENSSL_CIPHER_AES_256_CBC);
$encrypted = $encrypter->encrypt($signed);
```

### DKIM

```php
$signer = new DkimSigner('file:///etc/dkim/private.key', 'example.com', 'default', [
    'algorithm' => DkimSigner::ALGO_ED25519,
    'headers_to_ignore' => ['Return-Path'],
]);
$signed = $signer->sign($email);
```

Always sign last (after S/MIME if used together).

## Mailer Events

| Event | Stage | Methods |
|-------|-------|---------|
| `MessageEvent` | Before sending | `getMessage()`, `getEnvelope()`, `setMessage()`, `setEnvelope()`, `setRejected()`, `isQueued()`, `getTransport()` |
| `SentMessageEvent` | After successful send | `getMessage()` returns `SentMessage` (with `getMessageId()`, `getDebug()`) |
| `FailedMessageEvent` | After failure | `getMessage()`, `getError()` |
| `MessageEvents` (collector) | Profiler | Aggregates for dev profiler |

## Custom Transport

```php
use Symfony\Component\Mailer\Transport\AbstractTransport;
use Symfony\Component\Mailer\SentMessage;
use Symfony\Component\Mime\RawMessage;

final class FaxTransport extends AbstractTransport
{
    protected function doSend(SentMessage $message): void
    {
        $envelope = $message->getEnvelope();
        $raw = $message->toString();
        $this->faxClient->dispatch($raw, $envelope->getRecipients()[0]->getAddress());
    }

    public function __toString(): string
    {
        return 'fax://default';
    }
}
```

Register a `TransportFactoryInterface`:

```php
final class FaxTransportFactory extends AbstractTransportFactory
{
    public function create(Dsn $dsn): TransportInterface
    {
        if ($dsn->getScheme() === 'fax') {
            return new FaxTransport(/* ... */);
        }
        throw new UnsupportedSchemeException($dsn, 'fax', $this->getSupportedSchemes());
    }

    protected function getSupportedSchemes(): array { return ['fax']; }
}
```

Tag with `mailer.transport_factory` (auto-tagged via autoconfigure).

## Testing

WebTestCase assertions (all available with `MailerAssertionsTrait`):

```php
$this->assertEmailCount(1);                   // Total sent
$this->assertQueuedEmailCount(0);             // Queued via messenger
$email = $this->getMailerMessage(0);          // Get nth email
$this->assertEmailIsQueued($email);
$this->assertEmailHasHeader($email, 'X-Tenant');
$this->assertEmailHeaderSame($email, 'Subject', 'Hello');
$this->assertEmailTextBodyContains($email, 'Welcome');
$this->assertEmailHtmlBodyContains($email, 'Welcome');
$this->assertEmailAttachmentCount($email, 1);
```

## Deliverability Tips

- Set both `text()` and `html()`; many filters penalize HTML-only.
- Configure DKIM + SPF + DMARC at DNS level.
- Use a dedicated `Return-Path` (envelope sender) for bounce handling.
- Keep `From` consistent (RFC 5322); use `Reply-To` for replies.
- Add `List-Unsubscribe` header for marketing emails.
- Use `mailer:test` to validate SMTP credentials early.
- Avoid sending massive attachments; link to a hosted file instead.
- Throttle via `RateLimiter` if your provider rate-limits per second.
