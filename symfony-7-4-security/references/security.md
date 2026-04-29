# Symfony 7.4 Security - Complete Reference

## Table of Contents

- [Configuration Schema](#configuration-schema)
- [User Providers](#user-providers)
- [Password Hashers](#password-hashers)
- [Authenticators](#authenticators)
  - [Built-in Authenticators](#built-in-authenticators)
  - [Custom Authenticator Lifecycle](#custom-authenticator-lifecycle)
  - [Passports and Badges](#passports-and-badges)
- [Access Tokens (OIDC/JWT)](#access-tokens)
- [Voters](#voters)
- [Role Hierarchy](#role-hierarchy)
- [Access Control](#access-control)
- [Login Link / Magic Link](#login-link)
- [Remember Me](#remember-me)
- [Switch User](#switch-user)
- [Login Throttling](#login-throttling)
- [Logout](#logout)
- [CSRF](#csrf)
- [Security Events](#security-events)
- [Programmatic Authentication](#programmatic-authentication)
- [Testing](#testing)

---

## Configuration Schema

```yaml
security:
    enable_authenticator_manager: true   # Default in 7.x; legacy disabled

    password_hashers:
        App\Entity\User: 'auto'           # bcrypt | sodium | pbkdf2 | argon2i | auto
        legacy:
            algorithm: 'bcrypt'
            cost: 13

    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email
        chain:
            chain:
                providers: [app_user_provider, ldap_users]
        memory:
            memory:
                users:
                    admin: { password: '$argon2i$...', roles: ['ROLE_ADMIN'] }
        ldap:
            ldap:
                service: Symfony\Component\Ldap\Ldap
                base_dn: 'dc=example,dc=com'
                search_dn: 'cn=admin,dc=example,dc=com'
                search_password: '%env(LDAP_PASS)%'
                default_roles: ['ROLE_USER']
                uid_key: uid

    role_hierarchy:
        ROLE_ADMIN: [ROLE_USER]

    firewalls:
        main:
            pattern: ^/
            stateless: false
            lazy: true
            provider: app_user_provider
            entry_point: App\Security\ApiKeyAuthenticator
            user_checker: App\Security\UserChecker

            json_login:    { check_path: /api/login, username_path: email, password_path: password }
            form_login:    { login_path: /login, check_path: /login, enable_csrf: true }
            login_link:    { check_route: login_link_check, signature_properties: [email] }
            remember_me:   { secret: '%kernel.secret%', lifetime: 604800, path: / }
            access_token:  { token_handler: App\Security\TokenHandler }
            login_throttling: { max_attempts: 5, interval: '15 minutes' }
            switch_user:   { role: ROLE_ALLOWED_TO_SWITCH }
            logout:        { path: /logout, target: / }

            custom_authenticators:
                - App\Security\ApiKeyAuthenticator

    access_control:
        - { path: ^/admin, roles: ROLE_ADMIN, requires_channel: https, allow_if: "request.headers.get('X-Forwarded-Proto') === 'https'" }
```

## User Providers

| Provider type | Use when |
|---------------|----------|
| `entity` | Backed by Doctrine entity |
| `memory` | Hardcoded users (dev) |
| `ldap` | Active Directory / OpenLDAP |
| `chain` | Try multiple providers in order |
| Custom service | Implement `UserProviderInterface` |

Custom provider:

```php
use Symfony\Component\Security\Core\User\UserProviderInterface;

final class ApiUserProvider implements UserProviderInterface
{
    public function loadUserByIdentifier(string $identifier): UserInterface { /* ... */ }
    public function refreshUser(UserInterface $user): UserInterface { /* ... */ }
    public function supportsClass(string $class): bool
    {
        return $class === User::class;
    }
}
```

```yaml
providers:
    api_users:
        id: App\Security\ApiUserProvider
```

## Password Hashers

`'auto'` selects the best available algorithm and supports rehashing on login:

```php
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

if ($hasher->needsRehash($user)) {
    $user->setPassword($hasher->hashPassword($user, $plain));
    $em->flush();
}
```

Available algorithms: `bcrypt`, `sodium` (Argon2id), `argon2i`, `pbkdf2`. `auto` resolves to `sodium` if libsodium installed, otherwise `bcrypt`.

`migrate_from`:

```yaml
password_hashers:
    App\Entity\User:
        algorithm: 'auto'
        migrate_from: ['bcrypt', 'pbkdf2']
```

## Authenticators

### Built-in Authenticators

| Authenticator | Firewall key |
|---------------|-------------|
| Form login | `form_login` |
| JSON login | `json_login` |
| HTTP Basic | `http_basic` |
| Remember me | `remember_me` |
| Login link | `login_link` |
| Access token | `access_token` |
| X.509 | `x509` |
| LDAP form/HTTP | `form_login_ldap`, `http_basic_ldap` |

### Custom Authenticator Lifecycle

`AbstractAuthenticator` defines five hooks:

1. `supports(Request)` - Return `true` if this authenticator should run, `false` to skip, `null` to defer (no longer recommended; return bool).
2. `authenticate(Request)` - Build a `Passport`. Throw `AuthenticationException` on bad input.
3. `createToken(Passport, string $firewallName)` - Inherited; default produces `UsernamePasswordToken` or `PostAuthenticationToken`.
4. `onAuthenticationSuccess(Request, TokenInterface, string $firewallName)` - Return a `Response` (e.g. redirect) or `null` to continue the request.
5. `onAuthenticationFailure(Request, AuthenticationException)` - Return a `Response` (e.g. JSON error) or `null` to continue (subsequent authenticators may run).

Optional: implement `InteractiveAuthenticatorInterface` if it represents an interactive login (used for `IS_AUTHENTICATED_FULLY`).

### Passports and Badges

```php
use Symfony\Component\Security\Http\Authenticator\Passport\Passport;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Credentials\PasswordCredentials;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\CsrfTokenBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\RememberMeBadge;

return new Passport(
    new UserBadge($identifier, fn ($id) => $userProvider->loadUserByIdentifier($id)),
    new PasswordCredentials($plainPassword),
    [
        new CsrfTokenBadge('authenticate', $csrfToken),
        new RememberMeBadge(),
    ],
);
```

`SelfValidatingPassport` skips credential check (use for tokens already validated, e.g. JWT).

Available badges: `UserBadge`, `PasswordCredentials`, `CustomCredentials`, `CsrfTokenBadge`, `RememberMeBadge`, `PreAuthenticatedUserBadge`.

## Access Tokens

```yaml
firewalls:
    api:
        stateless: true
        access_token:
            token_handler: App\Security\JwtHandler
            token_extractors:
                - header                # Authorization: Bearer
                - query_string          # ?access_token=
                - request_body
                - cookie
```

OIDC built-in handler:

```yaml
access_token:
    token_handler:
        oidc:
            audience: '%env(OIDC_AUDIENCE)%'
            issuers: ['%env(OIDC_ISSUER)%']
            algorithms: ['ES256', 'RS256']
            keyset: '%env(OIDC_JWKS)%'
```

OIDC userinfo handler (calls `/userinfo`):

```yaml
access_token:
    token_handler:
        oidc_user_info:
            base_uri: 'https://provider/userinfo'
            client: secure.client
```

Custom handler:

```php
final class JwtHandler implements AccessTokenHandlerInterface
{
    public function getUserBadgeFrom(string $accessToken): UserBadge
    {
        $payload = $this->jwt->decode($accessToken);
        return new UserBadge((string) $payload->sub);
    }
}
```

## Voters

Extend `Voter` for type-safe voting; implement `VoterInterface` for custom strategies.

```php
final class PostVoter extends Voter
{
    public const EDIT = 'POST_EDIT';

    protected function supports(string $attribute, mixed $subject): bool
    {
        return $attribute === self::EDIT && $subject instanceof Post;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        return $token->getUser() === $subject->getAuthor();
    }
}
```

Strategy (`security.access_decision_manager.strategy`):
- `affirmative` (default): grant if any voter returns `ACCESS_GRANTED`.
- `consensus`: majority.
- `unanimous`: every voter must grant.
- `priority`: first non-abstaining voter decides.

### #[IsGranted] Attribute

```php
#[IsGranted('POST_EDIT', subject: 'post', message: 'Cannot edit', statusCode: 403)]
public function edit(Post $post): Response { /* ... */ }
```

`subject` resolves to a controller argument. `subject` can be an array/expression: `subject: new Expression('args["post"]')`.

## Role Hierarchy

```yaml
role_hierarchy:
    ROLE_EDITOR: [ROLE_USER]
    ROLE_ADMIN:  [ROLE_EDITOR]
    ROLE_SUPER_ADMIN: [ROLE_ADMIN, ROLE_ALLOWED_TO_SWITCH]
```

Roles must start with `ROLE_`. `IS_AUTHENTICATED_*` are not roles; do not use in hierarchy.

## Access Control

Evaluated top-to-bottom; first match wins. Attributes:

```yaml
access_control:
    - { path: ^/admin,
        host: '^admin\.',
        port: 443,
        ips: ['10.0.0.0/8'],
        methods: [POST, PUT],
        route: ^api_,
        roles: ROLE_ADMIN,
        allow_if: "is_granted('ROLE_USER') and 'admin' in roles",
        requires_channel: https }
```

## Login Link

```yaml
firewalls:
    main:
        login_link:
            check_route: login_link_check
            check_post_only: false
            signature_properties: [email, password]
            lifetime: 600
            max_uses: 1
```

```php
public function request(Request $request, LoginLinkHandlerInterface $linkHandler, NotifierInterface $notifier): Response
{
    $user = $this->users->findOneByEmail($request->request->get('email'));
    $details = $linkHandler->createLoginLink($user);
    $notifier->send(new LoginLinkNotification($details, 'Welcome'), $user);
    return $this->json(['ok' => true]);
}
```

## Remember Me

```yaml
firewalls:
    main:
        remember_me:
            secret: '%kernel.secret%'
            lifetime: 604800
            path: /
            secure: true
            httponly: true
            samesite: lax
            always_remember_me: false
            token_provider: doctrine                # or service id
            signature_properties: [password, email]
```

`signature_properties` invalidates cookies if the listed properties change (e.g. password reset).

## Switch User

```yaml
firewalls:
    main:
        switch_user: { role: ROLE_ALLOWED_TO_SWITCH, parameter: _switch_user, target_route: home }
```

Detect impersonation:

```php
$this->isGranted('IS_IMPERSONATOR');
$this->getUser()->getRoles();          // Includes ROLE_PREVIOUS_ADMIN
```

## Login Throttling

```yaml
firewalls:
    main:
        login_throttling:
            max_attempts: 5
            interval: '15 minutes'
            lock_factory: 'limiter.login'   # Optional: rate limiter id
```

Custom rate limiter strategy:

```yaml
framework:
    rate_limiter:
        login:
            policy: token_bucket
            limit: 5
            rate: { interval: '15 minutes' }
```

## Logout

```yaml
firewalls:
    main:
        logout:
            path: /logout
            target: /
            invalidate_session: true
            clear_site_data: ['cookies', 'storage']
            csrf_parameter: _csrf_token
            csrf_token_id: logout
            delete_cookies:
                PHPSESSID: { path: /, domain: example.com }
```

Hook on logout:

```php
use Symfony\Component\Security\Http\Event\LogoutEvent;

#[AsEventListener]
final class LogoutListener
{
    public function __invoke(LogoutEvent $event): void
    {
        // Custom cleanup
    }
}
```

## CSRF

Enable on form login:

```yaml
form_login:
    enable_csrf: true
    csrf_parameter: _csrf_token
    csrf_token_id: authenticate
```

In Twig:

```twig
<input type="hidden" name="_csrf_token" value="{{ csrf_token('authenticate') }}">
```

## Security Events

| Event | When |
|-------|------|
| `LoginSuccessEvent` | After authenticator success |
| `LoginFailureEvent` | After authentication exception |
| `LogoutEvent` | On logout |
| `SwitchUserEvent` | When switch_user activates |
| `InteractiveLoginEvent` | After interactive login |
| `AuthenticationTokenCreatedEvent` | After passport->token |
| `CheckPassportEvent` | Inspect/modify passport before token |

## Programmatic Authentication

```php
use Symfony\Bundle\SecurityBundle\Security;

$this->security->login($user, 'form_login', 'main', [
    new RememberMeBadge(),
]);

$this->security->logout(validateCsrfToken: false);
```

## Testing

```php
class SecuredEndpointTest extends WebTestCase
{
    public function test(): void
    {
        $client = static::createClient();
        $user = self::getContainer()->get(UserRepository::class)->findOneByEmail('admin@example.com');
        $client->loginUser($user);

        $client->request('GET', '/admin');
        self::assertResponseIsSuccessful();
    }
}
```

`loginUser($user, 'main')` for non-default firewall.
