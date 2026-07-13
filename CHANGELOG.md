# Changelog

All notable changes to the WoowTech Hermes Agent deployment package.

## [0.15.1] - 2026-07-13

### Changed
- Restructured GitHub repo: clean layout with `deploy/`, `config/`, `docker/`, `branding/`, `tests/`
- Added bilingual README (English + Traditional Chinese) with Mermaid architecture diagrams
- Removed OpenClaw content pollution from k3s/podman branches

## [0.15.0] - 2026-07-12

### Fixed
- Model routing: added `@openai-api:*` routes for WebUI model picker compatibility
- Synced model list with WebUI picker (added gpt-5.5-pro, gpt-5.4-nano, removed gpt-5.5-mini)

### Added
- `.env` fingerprint sync patch for K3s/Podman deployments
- Playwright-based E2E test suite (10/10 pass)

## [0.14.0] - 2026-06

### Added
- Multi-instance deployment with `deploy-instance.sh`
- White-label branding system (WoowTech + Apporo templates)
- Golden config/settings templates (`golden-config.yaml`, `golden-settings.json`)
- Cloudflare Tunnel initialization script
- Instance registry (`instances.json`)

## [0.13.0] - 2026-05

### Added
- Custom Docker image with 47 CLI tools + Playwright + Chromium 148
- 7-round enterprise test suite (infrastructure, API, security, resilience, integration, LLM, WebUI)
- API contract documentation (46 verified endpoints)
- Podman compose deployment option
- 907-line Chinese user manual (25 chapters)

## [0.12.0] - 2026-04

### Added
- Initial Hermes Agent deployment on K3s Kubernetes
- Basic deploy.sh for single-instance deployment
- PostgreSQL + Redis stack
