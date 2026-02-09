# Symfony 7.4 LDAP Component - Full Reference

## Installation

```bash
composer require symfony/ldap
```

Requires PHP's `ldap` extension to be enabled.

## Namespace

`Symfony\Component\Ldap`

## Creating a Connection

The `Ldap` class uses an `AdapterInterface` to communicate with LDAP servers. The built-in adapter is `ExtLdap\Adapter`.

```php
use Symfony\Component\Ldap\Ldap;

// With host and encryption
$ldap = Ldap::create('ext_ldap', [
    'host' => 'my-server',
    'encryption' => 'ssl',
]);

// With connection string
$ldap = Ldap::create('ext_ldap', [
    'connection_string' => 'ldaps://my-server:636',
]);
```

### Connection Options

| Option | Description |
|---|---|
| `host` | IP or hostname of the LDAP server |
| `port` | Port used to access the LDAP server |
| `version` | LDAP protocol version |
| `encryption` | Encryption protocol: `ssl`, `tls`, or `none` (default) |
| `connection_string` | Alternative to `host` and `port` |
| `optReferrals` | Auto-follow referrals returned by LDAP server |
| `options` | Additional server options (see `ConnectionOptions`) |

## Authentication

### Standard Binding

```php
$ldap->bind($dn, $password);
```

**Security Warning:** When LDAP servers allow unauthenticated binds, a blank password will always be valid.

### SASL Binding (Symfony 7.2+)

```php
$ldap->saslBind($dn, $password);
// Additional optional arguments: $mech, $realm, $authcId, etc.
```

### Getting Current User Information (Symfony 7.2+)

```php
$userDn = $ldap->whoami();
```

## Querying

### Basic Query

```php
$query = $ldap->query('dc=symfony,dc=com', '(&(objectclass=person)(ou=Maintainers))');
$results = $query->execute();

foreach ($results as $entry) {
    // Process each entry
}
```

### Load All Results

```php
$results = $query->execute()->toArray();
```

### Query Scope Options

```php
use Symfony\Component\Ldap\Adapter\QueryInterface;

// Subtree search (default)
$query = $ldap->query('...', '...', ['scope' => QueryInterface::SCOPE_SUB]);

// Single object (base) search
$query = $ldap->query('...', '...', ['scope' => QueryInterface::SCOPE_BASE]);

// One level search
$query = $ldap->query('...', '...', ['scope' => QueryInterface::SCOPE_ONE]);
```

| Constant | PHP Function | Description |
|---|---|---|
| `SCOPE_SUB` | `ldap_search` | Subtree (default) |
| `SCOPE_BASE` | `ldap_read` | Single object |
| `SCOPE_ONE` | `ldap_list` | One level |

### Filtering Attributes

```php
$query = $ldap->query('dc=symfony,dc=com', '...', [
    'filter' => ['cn', 'mail'],
]);
```

## Managing Entries

### Entry Manager

```php
$entryManager = $ldap->getEntryManager();
```

### Creating an Entry

```php
use Symfony\Component\Ldap\Entry;

$entry = new Entry('cn=Fabien Potencier,dc=symfony,dc=com', [
    'sn' => ['fabpot'],
    'objectClass' => ['inetOrgPerson'],
]);

$entryManager->add($entry);
```

### Updating an Entry

```php
$entry->setAttribute('email', ['fabpot@symfony.com']);
$entryManager->update($entry);
```

### Accessing Entry Attributes

```php
$phoneNumber = $entry->getAttribute('phoneNumber');

// Case-sensitive check
$exists = $entry->hasAttribute('contractorCompany');

// Case-insensitive check
$exists = $entry->hasAttribute('contractorCompany', false);
```

### Adding/Removing Attribute Values

```php
$entryManager->addAttributeValues($entry, 'telephoneNumber', [
    '+1.111.222.3333',
    '+1.222.333.4444',
]);

$entryManager->removeAttributeValues($entry, 'telephoneNumber', [
    '+1.111.222.3333',
]);
```

### Removing an Entry

```php
$entryManager->remove(new Entry('cn=Test User,dc=symfony,dc=com'));
```

## Batch Operations

Use `applyOperations()` to update multiple attributes atomically:

```php
use Symfony\Component\Ldap\Adapter\ExtLdap\UpdateOperation;

$entryManager->applyOperations($entry->getDn(), [
    new UpdateOperation(LDAP_MODIFY_BATCH_ADD, 'mail', 'new1@example.com'),
    new UpdateOperation(LDAP_MODIFY_BATCH_ADD, 'mail', 'new2@example.com'),
]);
```

### Operation Types

| Constant | Description | Requires Values |
|---|---|---|
| `LDAP_MODIFY_BATCH_ADD` | Add attributes | Yes |
| `LDAP_MODIFY_BATCH_REMOVE` | Remove specific values | Yes |
| `LDAP_MODIFY_BATCH_REMOVE_ALL` | Remove all values | No |
| `LDAP_MODIFY_BATCH_REPLACE` | Replace attribute entirely | Yes |

## Key Classes and Interfaces

| Class/Interface | Purpose |
|---|---|
| `Symfony\Component\Ldap\Ldap` | Main LDAP client, factory method `create()` |
| `Symfony\Component\Ldap\LdapInterface` | Contract: `bind()`, `query()`, `getEntryManager()` |
| `Symfony\Component\Ldap\Entry` | LDAP entry with DN and attributes |
| `Symfony\Component\Ldap\Adapter\AdapterInterface` | Adapter contract |
| `Symfony\Component\Ldap\Adapter\ExtLdap\Adapter` | PHP ldap extension adapter |
| `Symfony\Component\Ldap\Adapter\ExtLdap\ConnectionOptions` | Connection config constants |
| `Symfony\Component\Ldap\Adapter\ExtLdap\EntryManager` | CRUD operations on entries |
| `Symfony\Component\Ldap\Adapter\ExtLdap\UpdateOperation` | Batch update operation |
| `Symfony\Component\Ldap\Adapter\QueryInterface` | Query contract with scope constants |

## Repository Structure

```
symfony/ldap/
├── Adapter/          # LDAP adapter implementations
├── Exception/        # Custom exceptions
├── Security/         # Security-related functionality (auth providers)
├── Entry.php         # LDAP entry representation
├── Ldap.php          # Main LDAP client class
├── LdapInterface.php # Interface definition
```

## Links

- Documentation: https://symfony.com/doc/7.4/components/ldap.html
- GitHub: https://github.com/symfony/ldap
- License: MIT
