# Doctrine Migrations 3.x - Complete Reference

## Table of Contents

- [Configuration Loaders](#configuration-loaders)
- [Storage Backends](#storage-backends)
- [Version Sorting](#version-sorting)
- [Dependency Factory](#dependency-factory)
- [AbstractMigration Hooks](#abstractmigration-hooks)
- [Schema Differ](#schema-differ)
- [Events](#events)
- [Custom Generator / Template](#custom-generator--template)
- [Filtered Migrations](#filtered-migrations)
- [Standalone CLI (Outside Symfony)](#standalone-cli)
- [Doctrine Bundle Integration](#doctrine-bundle-integration)
- [Multiple Connections](#multiple-connections)

---

## Configuration Loaders

The library supports multiple config sources:

- `Doctrine\Migrations\Configuration\Migration\YamlFile`
- `Doctrine\Migrations\Configuration\Migration\PhpFile` (returns `Configuration` instance)
- `Doctrine\Migrations\Configuration\Migration\JsonFile`
- `Doctrine\Migrations\Configuration\Migration\XmlFile`
- `Doctrine\Migrations\Configuration\Migration\ConfigurationArray`
- `Doctrine\Migrations\Configuration\Migration\ExistingConfiguration`

In Symfony, the bundle reads `doctrine_migrations.yaml` and constructs a `Configuration` itself.

### Configuration Object

```php
use Doctrine\Migrations\Configuration\Configuration;
use Doctrine\Migrations\Metadata\Storage\TableMetadataStorageConfiguration;

$config = new Configuration();
$config->addMigrationsDirectory('DoctrineMigrations', __DIR__.'/migrations');
$config->setAllOrNothing(true);
$config->setTransactional(true);
$config->setCheckDatabasePlatform(true);
$config->setCustomTemplate(__DIR__.'/template.tpl');

$storage = new TableMetadataStorageConfiguration();
$storage->setTableName('doctrine_migration_versions');
$storage->setVersionColumnLength(191);
$config->setMetadataStorageConfiguration($storage);
```

## Storage Backends

### TableMetadataStorageConfiguration (default)

Schema:

```sql
CREATE TABLE doctrine_migration_versions (
    version VARCHAR(191) NOT NULL,
    executed_at DATETIME DEFAULT NULL,
    execution_time INTEGER DEFAULT NULL,
    PRIMARY KEY (version)
);
```

Customize column names/lengths:

```yaml
doctrine_migrations:
    storage:
        table_storage:
            table_name: 'migrations_log'
            version_column_name: 'migration_version'
            version_column_length: 255
            executed_at_column_name: 'ran_at'
            execution_time_column_name: 'duration_ms'
```

### Custom Storage

Implement `Doctrine\Migrations\Metadata\Storage\MetadataStorage` and provide via `services` mapping.

## Version Sorting

By default, versions sort by their numeric class suffix (`Version20260101120000`). Override:

```php
use Doctrine\Migrations\Version\AlphabeticalComparator;
use Doctrine\Migrations\Version\Comparator;

final class HierarchicalComparator implements Comparator
{
    public function compare(Version $a, Version $b): int
    {
        // Custom logic
    }
}
```

```yaml
doctrine_migrations:
    services:
        'Doctrine\Migrations\Version\Comparator': App\Migrations\HierarchicalComparator
```

## Dependency Factory

`Doctrine\Migrations\DependencyFactory` is the IoC root. Most services are lazily created and overridable:

```php
$factory = DependencyFactory::fromConnection(
    new ExistingConfiguration($config),
    new ExistingConnection($conn),
    $logger,
);

$factory->setService(
    'Doctrine\Migrations\Version\MigrationFactory',
    new ContainerAwareFactory(/* ... */)
);
```

In Symfony, the bundle wraps this and exposes all services as DI-managed.

## AbstractMigration Hooks

Order of execution per direction:

1. `preUp()` / `preDown()` — pre-flight (abort, skip, warn).
2. `up()` / `down()` — register `addSql()` statements.
3. Statements executed.
4. `postUp()` / `postDown()` — post-processing (e.g. data backfill via DBAL).

```php
public function postUp(Schema $schema): void
{
    $this->connection->executeStatement(
        "UPDATE article SET status = 'published' WHERE published = 1"
    );
}
```

Use `postUp()` for data backfills that are not pure DDL: they execute outside the schema diff context.

### Schema Object in `up()`/`down()`

The `Schema` parameter is **not** the live DB. It's a doctrine `Schema` snapshot of the live DB at migration start. You may inspect it but `addSql()` is the only mechanism to modify the actual DB.

Some teams use it for assertions:

```php
public function up(Schema $schema): void
{
    $this->abortIf(!$schema->hasTable('article'), 'Table missing');
    $this->addSql(/* ... */);
}
```

### Helpers

| Helper | Effect |
|--------|--------|
| `abortIf($cond, $msg)` | Throws if true, halts migration |
| `skipIf($cond, $msg)` | Skips remaining statements (mark version executed) |
| `warnIf($cond, $msg)` | Logs warning, continues |
| `write($msg)` | Logs informational message |
| `throwIrreversibleMigrationException($msg)` | Use in `down()` when revert is impossible |

## Schema Differ

`doctrine:migrations:diff` invokes the `SchemaProvider`:

- `OrmSchemaProvider`: compares ORM metadata to DB.
- `EmptySchemaProvider`: produces a diff against empty schema (initial baseline).

Filter tables to ignore:

```yaml
doctrine:
    dbal:
        schema_filter: ~^(?!session_)~
```

This excludes tables matching the regex from diff/dump.

For ORM-specific filtering, use:

```yaml
doctrine:
    orm:
        mappings:
            App:
                # ...
        controller_resolver:
            auto_mapping: true
```

## Events

The library dispatches Doctrine events:

- `Events::onMigrationsMigrated`
- `Events::onMigrationsMigrating`
- `Events::onMigrationsVersionExecuting`
- `Events::onMigrationsVersionExecuted`
- `Events::onMigrationsVersionSkipped`
- `Events::onMigrationsQueryExecuting`
- `Events::onMigrationsQueryExecuted`

Subscribe via Doctrine event manager:

```php
use Doctrine\Common\EventSubscriber;
use Doctrine\Migrations\Events;
use Doctrine\Migrations\Event\MigrationsEventArgs;

final class MigrationLogger implements EventSubscriber
{
    public function getSubscribedEvents(): array
    {
        return [Events::onMigrationsMigrated];
    }

    public function onMigrationsMigrated(MigrationsEventArgs $args): void
    {
        $this->slack->notify('Migrations applied to '.$args->getConfiguration()->getMetadataStorageConfiguration());
    }
}
```

## Custom Generator / Template

Custom template:

```
<?php

declare(strict_types=1);

namespace <namespace>;

use Doctrine\DBAL\Schema\Schema;
use Doctrine\Migrations\AbstractMigration;

/**
 * <description>
 */
final class Version<version> extends AbstractMigration
{
    public function up(Schema $schema): void
    {
<up>
    }

    public function down(Schema $schema): void
    {
<down>
    }
}
```

Reference at https://github.com/doctrine/migrations/blob/3.8.x/lib/Doctrine/Migrations/Generator/Template.tpl

Custom `Generator`:

```php
use Doctrine\Migrations\Generator\Generator;

final class CompanyGenerator extends Generator
{
    public function generateMigration(string $fqcn, ?string $up = null, ?string $down = null): string
    {
        $code = parent::generateMigration($fqcn, $up, $down);
        return preg_replace('/\@author .*/', '@author '.getenv('USER'), $code);
    }
}
```

```yaml
doctrine_migrations:
    services:
        'Doctrine\Migrations\Generator\Generator': App\Migrations\CompanyGenerator
```

## Filtered Migrations

Skip migrations conditionally (e.g. environment-specific):

```php
public function preUp(Schema $schema): void
{
    $this->skipIf(
        $_ENV['APP_ENV'] === 'test',
        'Skipping seed migration in test environment'
    );
}
```

## Standalone CLI

Without Symfony bundle:

```bash
composer require doctrine/migrations
```

Create `migrations.php`:

```php
return [
    'table_storage' => ['table_name' => 'doctrine_migration_versions'],
    'migrations_paths' => ['DoctrineMigrations' => __DIR__.'/migrations'],
];
```

Create `cli-config.php`:

```php
use Doctrine\Migrations\Configuration\Migration\PhpFile;
use Doctrine\Migrations\Configuration\Connection\ExistingConnection;
use Doctrine\Migrations\DependencyFactory;
use Doctrine\DBAL\DriverManager;

$conn = DriverManager::getConnection(['url' => 'pgsql://user:pass@host/db']);
$config = new PhpFile(__DIR__.'/migrations.php');

return DependencyFactory::fromConnection($config, new ExistingConnection($conn));
```

Run:

```bash
vendor/bin/doctrine-migrations migrate
```

## Doctrine Bundle Integration

The Symfony bundle wires:
- `database_platform` from `doctrine.dbal.connection`.
- Migration paths from bundle config or auto-detected from `migrations/`.
- Logger from Symfony console.
- Connection from `doctrine.orm.entity_manager` (or specific EM).

Use `MakerBundle` for shorter generation:

```bash
php bin/console make:migration
# Equivalent to doctrine:migrations:diff but with friendlier output
```

## Multiple Connections

Per-EM:

```yaml
doctrine:
    dbal:
        connections:
            default: { url: '%env(DATABASE_URL)%' }
            audit:   { url: '%env(AUDIT_DATABASE_URL)%' }
    orm:
        entity_managers:
            default: { connection: default }
            audit:   { connection: audit }

doctrine_migrations:
    migrations_paths:
        'DoctrineMigrations\Default': '%kernel.project_dir%/migrations/default'
        'DoctrineMigrations\Audit':   '%kernel.project_dir%/migrations/audit'
    em: default     # The default EM for migrations commands
```

Generate / migrate per EM:

```bash
php bin/console doctrine:migrations:diff --em=audit
php bin/console doctrine:migrations:migrate --em=audit
```

The bundle re-binds the migration namespace to the chosen EM, so Schema diff queries the right metadata.
