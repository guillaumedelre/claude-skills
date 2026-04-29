---
name: "symfony-7-4-mailer"
description: "Symfony 7.4 Mailer component reference for sending emails via SMTP, sendmail, or third-party APIs (Sendgrid, Mailgun, Amazon SES, Postmark, Brevo, etc.). Use when sending transactional emails, building HTML templates, attachments, embedding images, async dispatch via Messenger, or configuring transports. Triggers on: Mailer, MailerInterface, Email, TemplatedEmail, NotificationEmail, MAILER_DSN, SmtpTransport, SendgridTransport, MailgunTransport, AmazonSesTransport, PostmarkTransport, BrevoTransport, MailerBridgeInterface, Address, Envelope, attachments, attach, embed, htmlTemplate, textTemplate, context, MessageEvent, SentMessageEvent, FailedMessageEvent, async via Messenger, SendEmailMessage, MessageHandlerInterface, mailer:test, failover transport, round_robin transport, signed and encrypted email, S/MIME, DKIM."
---

# Symfony 7.4 Mailer Component

GitHub: https://github.com/symfony/mailer
Docs: https://symfony.com/doc/7.4/mailer.html

## Installation

```bash
composer require symfony/mailer
```

For a third-party provider (pick one):

```bash
composer require symfony/sendgrid-mailer
composer require symfony/mailgun-mailer
composer require symfony/amazon-mailer
composer require symfony/postmark-mailer
composer require symfony/brevo-mailer
```

## Quick Reference

### Configuration

```yaml
# config/packages/mailer.yaml
framework:
    mailer:
        dsn: '%env(MAILER_DSN)%'
        envelope:
            sender: noreply@example.com
        headers:
            From: 'Acme <noreply@example.com>'
```

```dotenv
# .env
MAILER_DSN=smtp://user:pass@smtp.example.com:587
# Other examples
# MAILER_DSN=sendmail://default
# MAILER_DSN=null://null
# MAILER_DSN=sendgrid+api://KEY@default
# MAILER_DSN=mailgun+https://KEY:DOMAIN@default?region=us
# MAILER_DSN=ses+api://ACCESS:SECRET@default?region=eu-west-1
# MAILER_DSN=postmark+api://KEY@default
# MAILER_DSN=brevo+api://KEY@default
```

### Sending a Plain Email

```php
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

class WelcomeMailer
{
    public function __construct(private MailerInterface $mailer) {}

    public function welcome(string $to): void
    {
        $email = (new Email())
            ->from('hello@example.com')
            ->to($to)
            ->cc('audit@example.com')
            ->bcc('archive@example.com')
            ->replyTo('support@example.com')
            ->priority(Email::PRIORITY_HIGH)
            ->subject('Welcome!')
            ->text('Welcome aboard.')
            ->html('<p>Welcome <strong>aboard</strong>.</p>');

        $this->mailer->send($email);
    }
}
```

### TemplatedEmail (Twig)

```php
use Symfony\Bridge\Twig\Mime\TemplatedEmail;

$email = (new TemplatedEmail())
    ->from('hello@example.com')
    ->to(new Address($user->getEmail(), $user->getName()))
    ->subject('Reset your password')
    ->htmlTemplate('emails/reset.html.twig')
    ->textTemplate('emails/reset.txt.twig')
    ->context([
        'user' => $user,
        'resetUrl' => $resetUrl,
    ]);
```

`emails/reset.html.twig` has access to all context vars plus `email` (the `Email` object itself, for setting subject inside template via `{% block subject %}...{% endblock %}` if extending the inky base).

### Attachments

```php
$email
    ->addPart(new DataPart(fopen('/path/file.pdf', 'r'), 'invoice.pdf', 'application/pdf'))
    // Convenience helpers:
    ->attach(fopen('/path/file.pdf', 'r'), 'invoice.pdf', 'application/pdf')
    ->attachFromPath('/path/file.pdf', 'invoice.pdf');
```

### Inline Images (cid:)

```php
$email
    ->embed(fopen('/path/logo.png', 'r'), 'logo')
    ->embedFromPath('/path/logo.png', 'logo')
    ->html('<img src="cid:logo">');
```

In a TemplatedEmail Twig:

```twig
<img src="{{ email.image('@assets/logo.png') }}">
```

### NotificationEmail (pre-styled inky template)

```php
use Symfony\Bridge\Twig\Mime\NotificationEmail;

$email = (new NotificationEmail())
    ->from('alerts@example.com')
    ->to('ops@example.com')
    ->subject('Disk almost full')
    ->markdown("# Storage at 92%\n\nServer **db-01** is critical.")
    ->action('Open dashboard', $dashboardUrl)
    ->importance(NotificationEmail::IMPORTANCE_URGENT);
```

Provides a built-in HTML template (`@email/zurb_2/notification/body.html.twig`).

### Address Helpers

```php
use Symfony\Component\Mime\Address;

$email->from(new Address('hello@example.com', 'Acme Support'));
$email->to('alice@example.com', new Address('bob@example.com', 'Bob'));
```

### Async Sending via Messenger

```yaml
framework:
    messenger:
        transports:
            async: '%env(MESSENGER_TRANSPORT_DSN)%'
        routing:
            'Symfony\Component\Mailer\Messenger\SendEmailMessage': async
```

The Mailer auto-dispatches a `SendEmailMessage` envelope. Run `messenger:consume async` to actually send.

### Failover and Round-robin Transports

```dotenv
MAILER_DSN=failover(smtp://primary:25 sendgrid+api://KEY@default)
MAILER_DSN=roundrobin(smtp://primary:25 smtp://secondary:25)
```

Use `&&` (failover) or `||` (round-robin) syntactic sugar in DSNs (older Symfony versions). 7.x uses parens explicitly above.

### Mailer Events

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Mailer\Event\MessageEvent;
use Symfony\Component\Mailer\Event\SentMessageEvent;
use Symfony\Component\Mailer\Event\FailedMessageEvent;

#[AsEventListener]
final class MailerListener
{
    public function onMessage(MessageEvent $event): void
    {
        $message = $event->getMessage();
        if ($event->isQueued()) { /* synchronous queueing */ }

        $message->getHeaders()->addTextHeader('X-Tenant', 'acme');

        // Override transport at runtime
        if (isDevEnvironment()) {
            $event->setRejected();   // Skip sending
        }
    }

    public function onSent(SentMessageEvent $event): void
    {
        $sent = $event->getMessage();   // SentMessage
        $messageId = $sent->getMessageId();
    }

    public function onFailed(FailedMessageEvent $event): void
    {
        $event->getError();
    }
}
```

### Signing / Encrypting (S/MIME)

```php
use Symfony\Component\Mime\Crypto\SMimeSigner;
use Symfony\Component\Mime\Crypto\SMimeEncrypter;

$signer = new SMimeSigner('/etc/ssl/cert.pem', '/etc/ssl/key.pem', 'passphrase');
$signed = $signer->sign($email);

$encrypter = new SMimeEncrypter('/etc/ssl/recipient.pem');
$encrypted = $encrypter->encrypt($signed);

$mailer->send($encrypted);
```

### DKIM

```php
use Symfony\Component\Mime\Crypto\DkimSigner;

$signer = new DkimSigner('file:///etc/dkim/private.key', 'example.com', 'default');
$signed = $signer->sign($email);
```

### Testing

```bash
php bin/console mailer:test recipient@example.com
```

Disable real sending in tests:

```dotenv
# .env.test
MAILER_DSN=null://null
```

In WebTestCase:

```php
$this->assertEmailCount(1);
$email = $this->getMailerMessage();
$this->assertEmailHtmlBodyContains($email, 'Welcome');
$this->assertEmailHeaderSame($email, 'Subject', 'Welcome!');
```

## Full Documentation

For all transport DSNs, custom transports, MIME message anatomy, deliverability tips, dev environments, and headers manipulation: see [references/mailer.md](references/mailer.md).
