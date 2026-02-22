# Rage Framework Skills

Skills to teach coding agents to work with the [Rage](https://github.com/rage-rb/rage) framework.

This repository follows the [Agent Skills](https://agentskills.io/home) specification - an open format for giving agents new capabilities and expertise. Skills are folders of instructions, scripts, and resources that agents can discover and use to work more accurately and efficiently.

## What's Included

These skills cover all major Rage framework features:

- **Cable** - Building WebSocket applications with Action Cable compatibility
- **OpenAPI** - Creating OpenAPI documentation for your API endpoints
- **RSpec** - Writing and organizing RSpec tests for Rage applications
- **Observability** - Working with logging infrastructure and telemetry handlers
- **Events** - Building event-driven architectures with typed events
- **Cookies & Sessions** - Cookie and session management

## Versioning

This project uses semantic versioning where the **major.minor** version matches the Rage framework version. The **patch** version is used to iterate on the skills independently of the framework.

For example:
- `v1.20.0` - Skills for Rage v1.20.x (initial release)
- `v1.20.1` - Skills for Rage v1.20.x (skill improvements)
- `v1.21.0` - Skills for Rage v1.21.x

## Installation

Rage provides built-in CLI utilities to manage skills:

```bash
# Install skills
rage skills install

# Update to the latest version
rage skills update
```

Alternatively, you can manually download releases from the [releases page](https://github.com/rage-rb/skills/releases).

## Verifying Downloads

### Checksum Verification

Each release includes a `checksums.txt` file. After downloading a release artifact, verify its integrity:

```bash
sha256sum -c checksums.txt
```

Or verify a single file:

```bash
sha256sum skills.tar.gz
# Compare the output with the corresponding line in checksums.txt
```

### Verifying Immutable Releases

To mitigate supply chain attacks, skills are published via [GitHub Immutable Releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases). This ensures releases cannot be tampered with after publication.

#### Using the GitHub CLI

First, [install the GitHub CLI](https://cli.github.com/) if you haven't already.

**Verify a release exists and hasn't been tampered with:**

```bash
gh release verify vX.Y.Z -R rage-rb/skills
```

**Verify a downloaded artifact matches the release:**

```bash
gh release verify-asset vX.Y.Z skills.tar.gz -R rage-rb/skills
```

#### Using the Web Interface

Navigate to the [releases page](https://github.com/rage-rb/skills/releases) and confirm that **"Immutable"** is displayed near the release author information.

### Verifying Artifact Attestations

Each release includes [artifact attestations](https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations) that cryptographically prove the artifact was built by GitHub Actions from this repository.

**Verify an artifact's attestation:**

```bash
gh attestation verify skills.tar.gz -R rage-rb/skills
```

A successful verification confirms:
- The artifact was built by GitHub Actions
- The artifact was built from the `rage-rb/skills` repository
- The artifact has not been modified since it was built

## License

[MIT](LICENSE)
