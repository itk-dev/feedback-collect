# Tidy Feedback Backend

Centralized feedback collection and triage backend. Aggregates feedback
from staging sites running the `itk-dev/tidy-feedback` widget.

## Tech Stack

- **Framework:** Symfony 7.x
- **Database:** MariaDB (via Doctrine ORM)
- **Frontend:** Stimulus + Turbo (Symfony UX), Bootstrap 5, AssetMapper
- **Dev environment:** itkdev-docker-compose (symfony-6 template)

## Development Workflow

All commands run inside Docker containers via Taskfile tasks.

```shell
# Start containers
task compose-up

# Install dependencies
task composer-install

# Run migrations
task console -- doctrine:migrations:migrate --no-interaction

# Apply coding standards
task coding-standards:apply

# Check coding standards
task coding-standards:check

# Run static analysis
task analyze

# Run tests
task test
```

## Key Directories

```
docs/           Specification documents and ADRs
docs/adr/       Architecture Decision Records
.docker/        Nginx config and Docker data
.github/        GitHub Actions CI workflows
```

## Coding Standards

- PHP: `@Symfony` rules via PHP CS Fixer
- Twig: Twig CS Fixer
- JavaScript/CSS/YAML: Prettier
- Markdown: markdownlint (120 char line limit)
- Static analysis: PHPStan

## Commits and Branches

Follow ITK Dev GitHub workflow:

- Never commit directly to `main`
- Branch naming: `feature/issue-{number}-{short-description}`
- Conventional Commits format
- Update `CHANGELOG.md` with every PR
