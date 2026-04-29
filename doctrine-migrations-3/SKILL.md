---
name: "doctrine-migrations-3"
description: "Doctrine Migrations 3.x reference for database schema versioning in Symfony/PHP applications via DoctrineMigrationsBundle. Use when generating, editing, executing, or rolling back schema migrations, configuring multiple migration namespaces, or integrating migrations into CI. Triggers on: Doctrine Migrations, DoctrineMigrationsBundle, doctrine:migrations:diff, doctrine:migrations:migrate, doctrine:migrations:generate, doctrine:migrations:execute, doctrine:migrations:status, doctrine:migrations:list, doctrine:migrations:up-to-date, doctrine:migrations:rollup, doctrine:migrations:dump-schema, doctrine:migrations:sync-metadata-storage, doctrine:migrations:current, doctrine:migrations:latest, AbstractMigration, MigrationFactory, ConfigurationLoader, schema diff, addSql, write, throwIrreversibleMigrationException, isTransactional, all_or_nothing, migrations.yaml, multi-namespace migrations, custom template, version provider, dependency factory, MetadataStorageConfiguration."
---

# Doctrine Migrations 3.x

GitHub: https://github.com/doctrine/migrations
Bundle: https://github.com/doctrine/DoctrineMigrationsBundle
Docs: https://www.doctrine-project.org/projects/doctrine-migrations/en/3.8/index.html

## Installation (Symfony)

```bash
composer require doctrine/doctrine-migrations-bundle
```

The recipe creates `migrations/` and `config/packages/doctrine_migrations.yaml`.

## Quick Reference

### Configuration

```yaml
# config/packages/doctrine_migrations.yaml
doctrine_migrations:
    migrations_paths:
        'DoctrineMigrations': '%kernel.project_dir%/migrations'
    enable_profiler: '%kernel.debug%'
    storage:
        table_storage:
            table_name: 'doctrine_migration_versions'
            version_column_name: 'version'
            version_column_length: 191
            executed_at_column_name: 'executed_at'
            execution_time_column_name: 'execution_time'
    transactional: true
    all_or_nothing: false
    check_database_platform: true
    custom_template: null
    organize_migrations: 'BY_YEAR_AND_MONTH'   # null | BY_YEAR | BY_YEAR_AND_MONTH
    services:
        'Doctrine\Migrations\Version\Comparator': App\Migrations\VersionComparator
```

### Generate from Diff

Creates a migration with `addSql()` calls reflecting schema differences between mapped entities and the live DB:

```bash
php bin/console doctrine:migrations:diff
php bin/console doctrine:migrations:diff --namespace='DoctrineMigrations'
php bin/console doctrine:migrations:diff --formatted --line-length=120
```

### Generate Empty Migration

```bash
php bin/console doctrine:migrations:generate
```

### Migration Class Skeleton

```php
declare(strict_types=1);

namespace DoctrineMigrations;

use Doctrine\DBAL\Schema\Schema;
use Doctrine\Migrations\AbstractMigration;

final class Version20260429120000 extends AbstractMigration
{
    public function getDescription(): string
    {
        return 'Add status column to article';
    }

    public function up(Schema $schema): void
    {
        $this->abortIf(
            $this->connection->getDatabasePlatform()->getName() !== 'postgresql',
            'Migration only valid on PostgreSQL'
        );

        $this->addSql('ALTER TABLE article ADD COLUMN status VARCHAR(32) NOT NULL DEFAULT \'draft\'');
        $this->addSql('CREATE INDEX idx_article_status ON article (status)');
    }

    public function down(Schema $schema): void
    {
        $this->addSql('DROP INDEX idx_article_status');
        $this->addSql('ALTER TABLE article DROP COLUMN status');
    }

    public function isTransactional(): bool
    {
        return true;   // Default
    }
}
```

### Run Migrations

```bash
php bin/console doctrine:migrations:migrate                    # All pending
php bin/console doctrine:migrations:migrate prev               # Roll back one
php bin/console doctrine:migrations:migrate first              # Roll back all
php bin/console doctrine:migrations:migrate latest             # Same as default
php bin/console doctrine:migrations:migrate 20260429120000     # Specific version
php bin/console doctrine:migrations:migrate next               # Up one
php bin/console doctrine:migrations:migrate --no-interaction
php bin/console doctrine:migrations:migrate --dry-run
php bin/console doctrine:migrations:migrate --write-sql=/tmp/up.sql
php bin/console doctrine:migrations:migrate --allow-no-migration
```

### Status & Inspection

```bash
php bin/console doctrine:migrations:status                     # Summary
php bin/console doctrine:migrations:list                       # All migrations + executed flag
php bin/console doctrine:migrations:current                    # Current version
php bin/console doctrine:migrations:latest                     # Latest available version
php bin/console doctrine:migrations:up-to-date                 # Exit 0 if no pending
```

### Rollup (Squash History)

After many migrations, baseline the schema (deletes all version history rows and marks the latest as executed):

```bash
php bin/console doctrine:migrations:rollup
```

### Execute Without Recording

```bash
php bin/console doctrine:migrations:execute 20260429120000 --up
php bin/console doctrine:migrations:execute 20260429120000 --down
php bin/console doctrine:migrations:execute 20260429120000 --dry-run
```

### Dump Schema (Skip History)

For new environments, dump the current schema as a starting baseline:

```bash
php bin/console doctrine:migrations:dump-schema
php bin/console doctrine:migrations:sync-metadata-storage
```

### Multiple Namespaces (e.g. per-bundle)

```yaml
doctrine_migrations:
    migrations_paths:
        'DoctrineMigrations\Core': '%kernel.project_dir%/migrations/core'
        'DoctrineMigrations\Audit': '%kernel.project_dir%/migrations/audit'
```

Generate into a specific namespace:

```bash
php bin/console doctrine:migrations:diff --namespace='DoctrineMigrations\Audit'
```

### Multiple Entity Managers

```yaml
doctrine_migrations:
    migrations_paths:
        'DoctrineMigrations\Default': '%kernel.project_dir%/migrations/default'
        'DoctrineMigrations\Audit': '%kernel.project_dir%/migrations/audit'
    em: default
```

Run for a specific EM:

```bash
php bin/console doctrine:migrations:migrate --em=audit
```

## AbstractMigration API

```php
final class Version20260101000000 extends AbstractMigration
{
    // Optional — override behaviors
    public function getDescription(): string { /* ... */ }
    public function isTransactional(): bool { return true; }
    public function preUp(Schema $schema): void {}
    public function up(Schema $schema): void { /* ... */ }
    public function postUp(Schema $schema): void {}
    public function preDown(Schema $schema): void {}
    public function down(Schema $schema): void { /* ... */ }
    public function postDown(Schema $schema): void {}

    // Helpers (inherited)
    protected function addSql(string $sql, array $params = [], array $types = []): void;
    protected function write(string $message): void;
    protected function throwIrreversibleMigrationException(?string $message = null): void;
    protected function abortIf(bool $condition, string $message = '...'): void;
    protected function skipIf(bool $condition, string $message = '...'): void;
    protected function warnIf(bool $condition, string $message = '...'): void;
}
```

### Non-transactional Migrations

PostgreSQL allows DDL inside transactions; MySQL does not for some operations (e.g. `CREATE INDEX CONCURRENTLY` in PG). Disable per-migration:

```php
public function isTransactional(): bool
{
    return false;
}
```

### Irreversible Migrations

```php
public function down(Schema $schema): void
{
    $this->throwIrreversibleMigrationException('Cannot restore deleted data');
}
```

## Custom Template

```yaml
doctrine_migrations:
    custom_template: '%kernel.project_dir%/migrations/Migration.tpl'
```

Template uses `{{ ... }}` placeholders: `version`, `namespace`, `description`, `migrationsPath`. See https://github.com/doctrine/migrations/blob/3.8.x/lib/Doctrine/Migrations/Generator/Template.tpl

## Custom MigrationFactory (DI)

```php
use Doctrine\Migrations\Version\MigrationFactory;
use Doctrine\Migrations\AbstractMigration;
use Doctrine\DBAL\Connection;

final class ContainerAwareFactory implements MigrationFactory
{
    public function __construct(
        private MigrationFactory $inner,
        private ContainerInterface $container,
    ) {}

    public function createVersion(string $migrationClassName): AbstractMigration
    {
        $migration = $this->inner->createVersion($migrationClassName);
        if ($migration instanceof ContainerAwareMigration) {
            $migration->setContainer($this->container);
        }
        return $migration;
    }
}
```

```yaml
doctrine_migrations:
    services:
        'Doctrine\Migrations\Version\MigrationFactory': App\Migrations\ContainerAwareFactory
```

## CI / Deploy

Typical pipeline step:

```bash
# Fail if pending migrations
php bin/console doctrine:migrations:up-to-date --fail-on-unregistered

# Apply
php bin/console doctrine:migrations:migrate --no-interaction --allow-no-migration
```

`--allow-no-migration` returns success if no migrations exist (useful in fresh repos).

## Full Documentation

For storage configuration, version comparators, custom configuration loaders, advanced commands, and event handling: see [references/migrations.md](references/migrations.md).
