# Tidy Feedback Backend

Centralized feedback collection and triage backend for
[itk-dev/tidy-feedback](https://github.com/itk-dev/tidy-feedback)
widget installations across multiple staging sites.

## Overview

This Symfony application aggregates feedback from staging sites running
the tidy-feedback widget into a single dashboard where Product Managers
can review, triage, and export feedback as GitHub issues.

See [Architecture Decision Records](docs/adr/) for key design decisions.

## Requirements

- [Docker](https://www.docker.com/) and
  [Docker Compose](https://docs.docker.com/compose/)
- [itkdev-docker-compose](https://github.com/itk-dev/devops_itkdev-docker)
- [Task](https://taskfile.dev/) (go-task)

## Installation

```shell
git clone https://github.com/itk-dev/feedback-collect.git
cd feedback-collect
docker network create frontend
task compose-up
task composer-install
task console -- doctrine:migrations:migrate --no-interaction
```

Open the site at the URL defined in `COMPOSE_DOMAIN`
(default: `feedback-collect.local.itkdev.dk`).

### Configuration

| Variable | Description | Default |
| -------- | ----------- | ------- |
| `COMPOSE_PROJECT_NAME` | Docker project name | `feedback-collect` |
| `COMPOSE_DOMAIN` | Local development domain | `feedback-collect.local.itkdev.dk` |
| `COMPOSE_SERVER_DOMAIN` | Server domain | |
| `DATABASE_URL` | MariaDB connection string | See `.env` |
| `APP_SECRET` | Symfony application secret | |

## Development

| Task | Description |
| ---- | ----------- |
| `task compose-up` | Start Docker containers |
| `task compose -- down` | Stop Docker containers |
| `task composer-install` | Install PHP dependencies |
| `task console -- <command>` | Run Symfony console command |
| `task coding-standards:apply` | Apply all coding standards |
| `task coding-standards:check` | Check all coding standards |
| `task analyze` | Run PHPStan analysis |
| `task test` | Run PHPUnit tests |

## Contributing

See the [GitHub workflow guidelines](docs/adr/) and ITK Dev conventions.
All changes must go through pull requests with at least one approving
review. Update `CHANGELOG.md` with every PR.

## License

*License to be determined.*
