# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Changed

- `azure-login` composite action now uses `azure/login@v3`
- Workflows are now driven by manual `workflow_dispatch` with an `environment` choice dropdown chosen by the operator — there is no automatic environment inference from git refs anymore
- `ci.yml` dropped the `resolve` job; it now outputs only `version` and takes an optional `app-version` input (defaults to `${{ github.run_number }}`)
- `ci-docker.yml` dropped the `resolve` job and the `should-deploy` gate (its Docker push job always runs); it takes optional `image-tag` (defaults to run number) and `push` inputs, and outputs `version` and `image-uri`
- Examples (`appservice-basic`, `appservice-docker`, `aks-full`) are now `workflow_dispatch`-driven with an `environment` dropdown and a `runDeploy` boolean gating the deploy job
- The template is now consumed via `@main` (all internal and example `uses:` references updated from `@v1`/version tags to `@main`)

### Added

- `cd-appservice.yml` gained optional non-secret `client-id`, `tenant-id`, and `subscription-id` inputs for Azure OIDC login; when provided they are used instead of the `AZURE_*` secrets (secrets are now optional fallbacks), letting consumers pass identifiers from a non-secret source such as a `bicepparam`

### Removed

- Removed the `resolve-version` composite action and all tag-driven / branch-driven / PR-driven resolution of version, environment, and should-deploy
- `ci.yml` and `ci-docker.yml` no longer output `environment`, `should-deploy`, `is-release`, or `is-prerelease`

### Fixed

- Corrected `uses:` references from `nuvtools/nuvtools-pipelines` to `nuvtools/nuvtools-templates-azure-cicd` in all reusable workflows, composite actions, examples, and docs
- Translated all Portuguese comments to English across workflows, actions, and documentation

### Added

- New example `appservice-docker` — App Service pipeline with Docker container deploy
- README for each example with prerequisites, parameters, and orchestration flow
- Top-level `examples/README.md` with comparison table and decision guide

### Removed

- Removed `aks-gitops` and `pipeline-repo` examples — values should live alongside application code rather than in a separate config repo
- Removed `migrating-from-azure-devops` documentation

## [1.0.0] - 2026-02-15

### Added

- Composite action `resolve-version` for version/environment detection from git refs
- Composite action `azure-login` with OIDC and Service Principal fallback
- Composite action `dotnet-build-test` for .NET build, test, and coverage
- Composite action `docker-build-push` for Docker build and ACR push
- Composite action `helm-deploy` for AKS deployment via Helm
- Reusable workflow `ci.yml` for continuous integration
- Reusable workflow `cd-aks.yml` for AKS continuous deployment
- Reusable workflow `cd-appservice.yml` for App Service continuous deployment
- Default Helm chart with ConfigMap env var support
- Default .NET runtime Dockerfile
- Examples for AKS full, App Service basic, and GitOps pipeline repo patterns
- Documentation: onboarding, architecture, authentication setup
