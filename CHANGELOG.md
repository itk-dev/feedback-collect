# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Backend specification document (`docs/feedback-backend-spec.md`)
- Frontend specification document (`docs/feedback-frontend-spec.md`)
- User stories document (`docs/user-stories.md`)
- Architecture Decision Records (ADR-001 through ADR-006)
- ITK Dev Docker setup based on symfony-6 template with MariaDB
- GitHub Actions CI workflows (changelog, composer, php, twig,
  markdown, yaml, javascript, styles)
- Code quality configs (PHP CS Fixer, Twig CS Fixer, Prettier,
  markdownlint, EditorConfig)
- Taskfile.yml with ITK Dev standard tasks
- README.md with ITK Dev documentation structure
- Project-level CLAUDE.md

### Changed

- Simplified documentation: trimmed README, archived planning specs
  to `docs/archive/`, added architecture summary to CLAUDE.md
- Updated backend spec with ITK Dev tooling references, PHP 8.4,
  and ADR cross-references
