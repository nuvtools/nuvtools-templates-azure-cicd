# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Fixed

- Corrected `uses:` references from `nuvtools/nuvtools-pipelines` to `nuvtools/nuvtools-templates-azure-cicd` in all reusable workflows, composite actions, examples, and docs
- Translated all Portuguese comments to English across workflows, actions, and documentation

### Added

- New example `aks-gitops` — AKS pipeline using an external GitOps config repository
- New example `appservice-docker` — App Service pipeline with Docker container deploy
- README for each example with prerequisites, parameters, and orchestration flow
- Top-level `examples/README.md` with comparison table and decision guide

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
- Documentation: onboarding, architecture, authentication setup, migration guide
