# Contributing to NuvTools Pipelines

Thank you for your interest in contributing! This guide will help you get started.

## How to Contribute

### Reporting Issues

- Use GitHub Issues to report bugs or request features
- Include the workflow version, runner OS, and relevant logs
- Redact any secrets or sensitive information from logs

### Pull Requests

1. Fork the repository
2. Create a feature branch from `main` (`git checkout -b feature/my-feature`)
3. Make your changes following the conventions below
4. Test your changes (see Testing section)
5. Commit with a clear message (`git commit -m "feat: add X support"`)
6. Push to your fork (`git push origin feature/my-feature`)
7. Open a Pull Request against `main`

### Commit Message Convention

We follow [Conventional Commits](https://www.conventionalcommits.org/):

- `feat:` — New feature
- `fix:` — Bug fix
- `docs:` — Documentation only
- `chore:` — Maintenance (CI, dependencies)
- `refactor:` — Code change that neither fixes a bug nor adds a feature
- `test:` — Adding or updating tests

### Code Conventions

- **YAML**: 2-space indentation, no trailing whitespace
- **Actions**: Use `description` for all inputs/outputs
- **Workflows**: Include comments explaining non-obvious logic
- **Helm charts**: Follow [Helm best practices](https://helm.sh/docs/chart_best_practices/)

### Testing

Before submitting a PR:

1. Validate YAML syntax: `yamllint .github/`
2. Validate GitHub Actions: `actionlint`
3. Validate Helm chart: `helm lint charts/default`
4. If modifying workflows, test with a consumer repo referencing your fork's branch

### Documentation

- Update `README.md` if adding new inputs/outputs
- Update `docs/` for architectural changes
- Add examples in `examples/` for new use cases
- Update `CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/)

## Code of Conduct

Be respectful and constructive. We follow the [Contributor Covenant](https://www.contributor-covenant.org/) code of conduct.

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
