---
name: "symfony-7-4-ldap"
description: "Symfony 7.4 LDAP component reference. Use this skill when working with LDAP, directory services, LDAP authentication, LDAP queries, Active Directory, or the Symfony Ldap adapter."
---

# Symfony 7.4 LDAP Component

## Overview

The Symfony LDAP component provides a PHP client on top of PHP's `ldap` extension for connecting to, authenticating against, and querying LDAP/Active Directory servers. It is used in Mezzo's **authentication** service for LDAP-based user authentication.

## Quick Reference

### Installation

```bash
composer require symfony/ldap
```

### Create Connection

```php
use Symfony\Component\Ldap\Ldap;

$ldap = Ldap::create('ext_ldap', [
    'host' => 'my-server',
    'encryption' => 'ssl',
]);
// Or with connection string:
$ldap = Ldap::create('ext_ldap', [
    'connection_string' => 'ldaps://my-server:636',
]);
```

### Bind (Authenticate)

```php
$ldap->bind($dn, $password);
```

### Query

```php
$query = $ldap->query('dc=symfony,dc=com', '(&(objectclass=person)(ou=Maintainers))');
$results = $query->execute();
```

### Manage Entries

```php
$entryManager = $ldap->getEntryManager();
$entryManager->add($entry);
$entryManager->update($entry);
$entryManager->remove($entry);
```

## Key Classes

| Class | Purpose |
|---|---|
| `Symfony\Component\Ldap\Ldap` | Main LDAP client |
| `Symfony\Component\Ldap\Entry` | Represents an LDAP entry |
| `Symfony\Component\Ldap\LdapInterface` | Contract for LDAP implementations |
| `Symfony\Component\Ldap\Adapter\ExtLdap\Adapter` | PHP ldap extension adapter |
| `Symfony\Component\Ldap\Adapter\QueryInterface` | Query interface with scope constants |

## Connection Options

| Option | Description |
|---|---|
| `host` | IP or hostname of the LDAP server |
| `port` | Port to access the LDAP server |
| `version` | LDAP protocol version |
| `encryption` | `ssl`, `tls`, or `none` (default) |
| `connection_string` | Full LDAP URI (alternative to host+port) |
| `optReferrals` | Auto-follow referrals |

## Full Documentation
- Official documentation: https://symfony.com/doc/7.4/components/ldap.html
- GitHub repository: https://github.com/symfony/ldap
- Detailed API reference: see [references/ldap.md](references/ldap.md)
