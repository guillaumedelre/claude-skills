---
name: "symfony-7-4-security"
description: "Symfony 7.4 Security bundle reference for authentication, authorization, password hashing, and access control. Use when configuring firewalls, providers, authenticators, voters, role hierarchy, login throttling, or any security-related Symfony code. Triggers on: Security, security.yaml, firewalls, providers, authenticators, AbstractAuthenticator, JsonLoginAuthenticator, FormLoginAuthenticator, LoginLinkAuthenticator, AccessTokenAuthenticator, RememberMeAuthenticator, voters, Voter, IsGranted, AccessDecisionManager, AuthorizationCheckerInterface, UserInterface, PasswordAuthenticatedUserInterface, PasswordHasherInterface, UserPasswordHasher, hash_password, IS_AUTHENTICATED_FULLY, ROLE_USER, role_hierarchy, switch_user, login_throttling, logout, CSRF, programmatic login, AccessToken, OIDC, security events, Security helper, CurrentUser, GrantedBy."
---

# Symfony 7.4 Security Bundle

GitHub: https://github.com/symfony/security-bundle
Docs: https://symfony.com/doc/7.4/security.html

## Installation

```bash
composer require symfony/security-bundle
```

## Quick Reference

### Minimal `security.yaml`

```yaml
security:
    password_hashers:
        Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface: 'auto'

    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email

    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false
        main:
            lazy: true
            provider: app_user_provider
            json_login:
                check_path: /api/login
                username_path: email
                password_path: password
            logout:
                path: /api/logout
            login_throttling:
                max_attempts: 5
                interval: '15 minutes'

    access_control:
        - { path: ^/api/admin, roles: ROLE_ADMIN }
        - { path: ^/api,       roles: PUBLIC_ACCESS }

    role_hierarchy:
        ROLE_ADMIN:       ROLE_USER
        ROLE_SUPER_ADMIN: ROLE_ADMIN
```

### User Entity

```php
use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;
use Symfony\Component\Security\Core\User\UserInterface;

#[ORM\Entity]
class User implements UserInterface, PasswordAuthenticatedUserInterface
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column]
    private ?int $id = null;

    #[ORM\Column(unique: true)]
    private string $email;

    #[ORM\Column]
    private string $password;

    #[ORM\Column]
    private array $roles = [];

    public function getUserIdentifier(): string { return $this->email; }
    public function getRoles(): array { return array_unique([...$this->roles, 'ROLE_USER']); }
    public function getPassword(): string { return $this->password; }
    public function eraseCredentials(): void {}
}
```

### Hashing Passwords

```php
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

public function register(User $user, string $plain, UserPasswordHasherInterface $hasher): void
{
    $user->setPassword($hasher->hashPassword($user, $plain));
}
```

### Custom Authenticator

```php
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Security\Http\Authenticator\AbstractAuthenticator;
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
use Symfony\Component\Security\Http\Authenticator\Passport\Credentials\CustomCredentials;
use Symfony\Component\Security\Http\Authenticator\Passport\Passport;
use Symfony\Component\Security\Http\Authenticator\Passport\SelfValidatingPassport;

final class ApiKeyAuthenticator extends AbstractAuthenticator
{
    public function __construct(private UserRepository $users) {}

    public function supports(Request $request): ?bool
    {
        return $request->headers->has('X-API-Key');
    }

    public function authenticate(Request $request): Passport
    {
        $apiKey = $request->headers->get('X-API-Key');

        return new SelfValidatingPassport(
            new UserBadge($apiKey, fn ($key) => $this->users->findOneByApiKey($key))
        );
    }

    public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response
    {
        return null;
    }

    public function onAuthenticationFailure(Request $request, AuthenticationException $exception): ?Response
    {
        return new JsonResponse(['error' => $exception->getMessage()], 401);
    }
}
```

Wire it:

```yaml
security:
    firewalls:
        main:
            custom_authenticators:
                - App\Security\ApiKeyAuthenticator
```

### Access Token Authenticator (JWT/OIDC)

```yaml
security:
    firewalls:
        api:
            access_token:
                token_handler: App\Security\JwtTokenHandler
                token_extractors:
                    - header
                    - query_string
```

```php
use Symfony\Component\Security\Http\AccessToken\AccessTokenHandlerInterface;

final class JwtTokenHandler implements AccessTokenHandlerInterface
{
    public function getUserBadgeFrom(string $accessToken): UserBadge
    {
        $payload = $this->jwt->decode($accessToken);
        return new UserBadge($payload['sub'], fn ($id) => $this->users->find($id));
    }
}
```

### Voter

```php
use Symfony\Component\Security\Core\Authorization\Voter\Voter;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;

final class PostVoter extends Voter
{
    public const EDIT = 'POST_EDIT';
    public const VIEW = 'POST_VIEW';

    protected function supports(string $attribute, mixed $subject): bool
    {
        return in_array($attribute, [self::EDIT, self::VIEW]) && $subject instanceof Post;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();
        if (!$user instanceof User) return false;

        return match ($attribute) {
            self::EDIT => $subject->getAuthor() === $user,
            self::VIEW => true,
        };
    }
}
```

### #[IsGranted] on Controllers

```php
use Symfony\Component\Security\Http\Attribute\IsGranted;
use Symfony\Component\Security\Http\Attribute\CurrentUser;

#[IsGranted('ROLE_ADMIN')]
class AdminController extends AbstractController
{
    #[Route('/admin/posts/{id}/edit')]
    #[IsGranted('POST_EDIT', 'post')]
    public function edit(Post $post, #[CurrentUser] User $user): Response { /* ... */ }
}
```

### Authorization Checker (in services)

```php
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;

public function __construct(private AuthorizationCheckerInterface $authChecker) {}

public function doSomething(Post $post): void
{
    if (!$this->authChecker->isGranted('POST_EDIT', $post)) {
        throw new AccessDeniedException();
    }
}
```

### Programmatic Login

```php
use Symfony\Bundle\SecurityBundle\Security;

public function __construct(private Security $security) {}

public function impersonate(User $user, Request $request): Response
{
    $this->security->login($user, 'form_login', 'main');
    return $this->redirectToRoute('home');
}
```

### Logout

```yaml
security:
    firewalls:
        main:
            logout:
                path: /logout
                target: /
                invalidate_session: true
                clear_site_data: ['*']
```

### Login Throttling

```yaml
firewalls:
    main:
        login_throttling:
            max_attempts: 5
            interval: '15 minutes'
            lock_factory: limiter.login
```

### Switch User (impersonation)

```yaml
firewalls:
    main:
        switch_user:
            role: ROLE_ALLOWED_TO_SWITCH
            parameter: _switch_user
```

URL: `/?_switch_user=alice@example.com` (exit: `/?_switch_user=_exit`).

## Built-in Attributes

| Symbol | Where |
|--------|-------|
| `IS_AUTHENTICATED_FULLY` | Logged in via primary credential |
| `IS_AUTHENTICATED_REMEMBERED` | Authenticated, possibly via remember-me |
| `IS_AUTHENTICATED` | Any authentication (incl. anonymous-less in 7.x) |
| `PUBLIC_ACCESS` | Bypass `access_control` |
| `IS_IMPERSONATOR` | True when running as switch_user |

## Full Documentation

For voters, authenticators full lifecycle, OIDC token handlers, login link, remember-me, custom user providers, security events, CSRF, and migration tips: see [references/security.md](references/security.md).
