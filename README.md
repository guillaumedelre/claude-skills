# 🎯 Claude Skills - Symfony & PHP Ecosystem

> Collection de skills spécialisés pour [Claude Code](https://claude.com/code) - Documentation experte pour Symfony 7.4, API Platform 4.2, Doctrine ORM 3 et FrankenPHP 1.x

[![Symfony 7.4](https://img.shields.io/badge/Symfony-7.4-000000?style=flat&logo=symfony)](https://symfony.com)
[![API Platform 4.2](https://img.shields.io/badge/API%20Platform-4.2-38A3A5?style=flat)](https://api-platform.com)
[![Doctrine ORM 3](https://img.shields.io/badge/Doctrine%20ORM-3-FC6A31?style=flat)](https://www.doctrine-project.org)
[![PHP 8.3+](https://img.shields.io/badge/PHP-8.3+-777BB4?style=flat&logo=php)](https://www.php.net)

## 📖 À propos

Ce dépôt contient une collection de **skills Claude Code** - des packages de documentation spécialisés qui permettent à Claude de fournir des conseils experts sur des bibliothèques et frameworks spécifiques.

### Qu'est-ce qu'un skill ?

Un **skill** est un fichier de documentation structuré (`SKILL.md`) qui se charge automatiquement quand vous travaillez sur du code utilisant une bibliothèque ou un framework particulier. Chaque skill fournit :

- ✨ **Exemples de code** pour les cas d'usage courants
- 📚 **Documentation de référence** détaillée
- 🎯 **Meilleures pratiques** et patterns modernes
- 🧪 **Exemples de tests** et patterns de test
- 🔍 **Déclenchement automatique** basé sur les mots-clés du contexte

## 🗂️ Skills disponibles

### Symfony 7.4 Components (38 skills)

<details>
<summary><b>Core Components</b></summary>

- **symfony-7-4-console** - Console commands, CLI, arguments, options, SymfonyStyle
- **symfony-7-4-http-foundation** - Request, Response, Session, Cookie, File uploads
- **symfony-7-4-http-kernel** - Kernel, Controllers, Events, Middleware
- **symfony-7-4-dependency-injection** - Service container, DI, Autowiring, Tags
- **symfony-7-4-config** - Configuration loading, validation, caching
- **symfony-7-4-event-dispatcher** - Event system, listeners, subscribers

</details>

<details>
<summary><b>Data & Validation</b></summary>

- **symfony-7-4-form** - Form creation, validation, theming, events
- **symfony-7-4-validator** - Constraints, custom validators, validation groups
- **symfony-7-4-property-access** - Property paths, nested access
- **symfony-7-4-property-info** - Property metadata, type extraction
- **symfony-7-4-type-info** - Type system, reflection

</details>

<details>
<summary><b>Utilities & Infrastructure</b></summary>

- **symfony-7-4-filesystem** - File operations, manipulation, permissions
- **symfony-7-4-finder** - Advanced file/directory searching
- **symfony-7-4-process** - Process execution, async processes
- **symfony-7-4-cache** - PSR-6/PSR-16 caching, adapters
- **symfony-7-4-lock** - Distributed locks, semaphores
- **symfony-7-4-messenger** - Message bus, async processing, transports
- **symfony-7-4-workflow** - State machines, workflows

</details>

<details>
<summary><b>Format Handling & Output</b></summary>

- **symfony-7-4-yaml** - YAML parsing, dumping, inline syntax
- **symfony-7-4-mime** - MIME types, email creation, file info
- **symfony-7-4-var-dumper** - Debugging, dump(), VarCloner
- **symfony-7-4-var-exporter** - Export PHP variables
- **symfony-7-4-asset** - Asset management, versioning

</details>

<details>
<summary><b>Specialized Components</b></summary>

- **symfony-7-4-clock** - Time testing, frozen time, mocking
- **symfony-7-4-css-selector** - CSS to XPath conversion
- **symfony-7-4-dom-crawler** - HTML/XML traversal, scraping
- **symfony-7-4-browser-kit** - HTTP client for testing
- **symfony-7-4-expression-language** - Expression parsing, evaluation
- **symfony-7-4-intl** - Internationalization utilities
- **symfony-7-4-json-path** - JSON path expressions
- **symfony-7-4-ldap** - LDAP operations
- **symfony-7-4-options-resolver** - Options configuration
- **symfony-7-4-phpunit-bridge** - PHPUnit integration
- **symfony-7-4-psr7** - PSR-7 HTTP message bridge
- **symfony-7-4-runtime** - Runtime architecture
- **symfony-7-4-semaphore** - Semaphore locks
- **symfony-7-4-uid** - UUID, ULID generation
- **symfony-7-4-contracts** - Symfony component contracts

</details>

### API & Data Layer

- **api-platform-4-2** - REST & GraphQL API creation, operations, DTOs, state providers/processors
- **doctrine-orm-3** - ORM, entities, repositories, DQL, query builder
- **frankenphp-1** - Modern PHP application server, worker mode, real-time

## 🚀 Utilisation

### Avec Claude Code (CLI)

Les skills se chargent automatiquement quand Claude détecte que vous travaillez avec la bibliothèque correspondante :

```bash
# Claude détecte automatiquement les classes Symfony dans votre projet
# et charge les skills appropriés

# Exemple : créer une commande console
claude "créer une commande pour importer des utilisateurs depuis un CSV"

# Exemple : créer une API REST
claude "créer une ressource API pour gérer des articles de blog avec validation"
```

### Installation manuelle d'un skill

Pour installer un skill spécifique dans votre projet :

```bash
# Copier le dossier du skill dans votre projet
cp -r symfony-7-4-console ~/.claude/skills/

# Ou créer un lien symbolique
ln -s /path/to/claude-skills/symfony-7-4-console ~/.claude/skills/
```

### Comment Claude utilise les skills

1. **Détection automatique** : Claude scanne votre code et détecte les classes, attributs et patterns
2. **Chargement contextuel** : Les skills pertinents se chargent automatiquement
3. **Quick Reference** : Claude consulte d'abord les exemples rapides du `SKILL.md`
4. **Documentation détaillée** : Si nécessaire, Claude lit les fichiers de référence dans `references/`

## 📁 Structure d'un skill

```
symfony-7-4-console/
├── SKILL.md                    # Fichier principal du skill
│   ├── Métadonnées (YAML)      # name, description, triggers
│   ├── Quick Reference         # 5-10 exemples courants
│   └── Documentation links     # Liens vers les références
│
└── references/                 # Documentation détaillée
    ├── commands.md             # Guide complet des commandes
    ├── output.md               # Styling et formatage de sortie
    ├── helpers.md              # Tables, progress bars, questions
    ├── events.md               # Events et signal handling
    └── testing.md              # Tests avec CommandTester
```

### Anatomie d'un SKILL.md

```markdown
---
name: "symfony-7-4-console"
description: "Symfony Console component. Use when creating commands, CLI tools. Triggers on: AsCommand, InputInterface, OutputInterface, SymfonyStyle"
---

# Symfony Console Component

Quick Reference with 5-10 practical code examples...

## Documentation Structure
Links to detailed reference files...

## When to Read References
Guidance on when to consult detailed docs...
```

## 🎯 Conventions

### Nommage

Format : `{library-name}-{major}-{minor}` (sans patch version)

- ✅ `symfony-7-4-console`
- ✅ `api-platform-4-2`
- ✅ `doctrine-orm-3`
- ❌ `symfony-7.4.1-console` (pas de patch)
- ❌ `Symfony_Console` (pas de camelCase)

### Contenu

**SKILL.md** (Quick Reference) :
- Exemples de code complets et fonctionnels
- Syntaxe PHP moderne (attributs, typed properties, PHP 8.3+)
- Les 80% de cas d'usage les plus fréquents
- 5-10 exemples concis mais pratiques

**references/*.md** (Documentation détaillée) :
- Couverture exhaustive des fonctionnalités
- Cas avancés et edge cases
- Considérations de performance et sécurité
- Peut être long (200-500+ lignes)

## 🤝 Contribution

### Ajouter un nouveau skill

1. Créer le dossier : `{library}-{major}-{minor}/`
2. Créer `SKILL.md` avec :
   - Frontmatter YAML (name, description, triggers)
   - Quick reference avec exemples pratiques
   - Liens vers la documentation détaillée
3. Créer `references/` avec fichiers de référence détaillés
4. Tester que les exemples sont syntaxiquement corrects

### Mettre à jour un skill existant

1. Garder le `SKILL.md` focus sur le quick reference
2. Déplacer le contenu avancé dans `references/`
3. Mettre à jour les numéros de version si nécessaire
4. Réviser les mots-clés de déclenchement (triggers)

### Checklist qualité

- [ ] YAML frontmatter valide
- [ ] Exemples de code syntaxiquement corrects
- [ ] Syntaxe moderne et idiomatique (PHP 8.3+, attributs)
- [ ] Tous les liens vers les références fonctionnent
- [ ] Description contient des mots-clés de déclenchement pertinents
- [ ] Exemples couvrent les cas d'usage principaux

## 📝 Pourquoi ce projet ?

Les skills permettent à Claude Code de :

- 🎓 **Apprendre en contexte** : Documentation chargée uniquement quand nécessaire
- 🚀 **Réponses plus rapides** : Pas besoin de chercher dans la doc officielle
- 💡 **Meilleurs exemples** : Code pratique et idiomatique, pas juste de la théorie
- 🔄 **Toujours à jour** : Version-specific skills avec la syntaxe moderne
- 🎯 **Précision maximale** : Documentation exacte pour votre version

## 🔗 Liens utiles

- [Claude Code](https://claude.com/code) - L'agent de développement IA
- [Symfony Documentation](https://symfony.com/doc/7.4/index.html)
- [API Platform](https://api-platform.com/docs/symfony/)
- [Doctrine ORM](https://www.doctrine-project.org/projects/orm.html)
- [FrankenPHP](https://frankenphp.dev/)

## 📄 Licence

Ce dépôt est un projet de documentation. Les exemples de code peuvent être utilisés librement dans vos projets.

---

<p align="center">
  Fait avec ❤️ pour la communauté <a href="https://symfony.com">Symfony</a> et <a href="https://www.php.net">PHP</a>
</p>
