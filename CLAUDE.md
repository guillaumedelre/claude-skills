# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a collection of **Claude Code skills** that provide specialized documentation and reference material for various PHP frameworks and libraries, particularly focused on Symfony 7.4 ecosystem, API Platform, Doctrine ORM, and FrankenPHP.

Skills are self-contained documentation packages that Claude Code can load to provide expert guidance on specific libraries and frameworks. Each skill provides:
- Quick reference examples for common use cases
- Detailed reference documentation
- Best practices and patterns
- Testing guidance

## Repository Structure

```
claude-skills/
├── symfony-7-4-{component}/    # Symfony 7.4 components (38 total)
├── api-platform-4-2/           # API Platform 4.2 reference
├── doctrine-orm-3/             # Doctrine ORM 3 reference
└── frankenphp-1/               # FrankenPHP 1.x reference

Each skill directory contains:
├── SKILL.md                    # Main skill file with metadata and quick reference
└── references/                 # Detailed reference documentation
    └── *.md                    # Topic-specific reference files
```

## Skill File Structure

Each skill follows a standard structure:

### SKILL.md Format
- **YAML frontmatter**: Contains `name` and `description` with trigger keywords
- **Overview section**: Installation, quick reference, common patterns
- **Documentation structure**: Links to detailed reference files
- **When to read references**: Guidance on when to consult detailed docs

### Reference Files
Located in `references/` subdirectory, these provide comprehensive documentation on specific topics. Examples:
- `commands.md` - Command creation, arguments, options
- `output.md` - Output formatting and styling
- `doctrine-orm.md` - Full ORM documentation
- `api-platform.md` - Complete API Platform guide

## Creating or Modifying Skills

### Naming Convention
- Format: `{library-name}-{major-version}-{minor-version}`
- Examples: `symfony-7-4-console`, `api-platform-4-2`, `doctrine-orm-3`
- Use hyphens, lowercase only

### SKILL.md Requirements

**Frontmatter:**
```yaml
---
name: "skill-name"
description: "Brief description. Use when [scenarios]. Triggers on: keyword1, keyword2, Class names, attribute names."
---
```

The `description` field is critical - it determines when Claude Code loads the skill. Include:
1. Brief summary of what the skill covers
2. "Use when..." scenarios describing when to invoke the skill
3. "Triggers on:" followed by comma-separated keywords, class names, attributes, and concepts that should trigger the skill

**Content structure:**
1. GitHub link and official docs link
2. Installation instructions
3. Quick reference with 5-10 common code examples
4. Documentation structure (link to reference files)
5. "When to read references" section guiding users to appropriate detailed docs

**Code examples:**
- Must be complete, runnable code snippets
- Include relevant imports and attributes
- Show idiomatic, modern syntax (PHP 8.3+ attributes, typed properties)
- Cover the most common 80% use cases

### Reference Documentation

Place detailed documentation in `references/` subdirectory:
- Break complex topics into separate files (e.g., `commands.md`, `output.md`, `helpers.md`)
- Each file should be comprehensive and self-contained
- Include advanced features, edge cases, and less common scenarios
- Can be lengthy (reference files are 200-500+ lines)

### Content Guidelines

**Quick Reference (SKILL.md):**
- Keep examples concise but complete
- Focus on what developers need most frequently
- Modern PHP syntax (attributes over annotations, typed properties)
- Show practical, real-world patterns
- Include testing examples where applicable

**Detailed References:**
- Comprehensive coverage of all features
- Edge cases and advanced scenarios
- Performance considerations
- Security best practices
- Migration notes for version upgrades

## Symfony Component Coverage

The repository includes skills for most major Symfony 7.4 components:
- Core components: console, http-foundation, http-kernel, dependency-injection
- Data handling: form, validator, serializer, property-access, property-info
- Utilities: filesystem, finder, process, cache, lock
- Infrastructure: messenger, event-dispatcher, workflow
- Format handling: yaml, mime, var-dumper, var-exporter
- Specialized: clock, css-selector, dom-crawler, browser-kit, expression-language

Each follows the same structure pattern but focuses on its specific domain.

## Working with This Repository

**To add a new skill:**
1. Create directory: `{library-name}-{version}/`
2. Add `SKILL.md` with frontmatter, quick reference, and links
3. Create `references/` subdirectory
4. Add detailed reference files in `references/`
5. Ensure all cross-references between files work correctly

**To update a skill:**
1. Keep SKILL.md focused on quick reference and common patterns
2. Move detailed/advanced content to reference files
3. Update version numbers in directory name if major version changes
4. Review trigger keywords in description to ensure skill loads appropriately

**To verify a skill:**
1. Ensure YAML frontmatter is valid
2. Check that all code examples are syntactically correct
3. Verify all links to reference files work
4. Test that descriptions contain appropriate trigger keywords
5. Confirm examples use modern, idiomatic syntax

## Important Notes

- This is a **documentation repository**, not a code repository - no tests, builds, or CI/CD
- Skills are meant to be loaded by Claude Code when working on projects using these libraries
- Focus on accuracy and completeness over brevity in reference files
- The quick reference in SKILL.md should enable 80% of common tasks without reading detailed docs
- Skills should be version-specific to ensure accuracy with API changes
