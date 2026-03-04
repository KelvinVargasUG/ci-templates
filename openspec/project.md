# CI Templates Project

## Overview
Repository of reusable GitHub Actions workflows and actions for automating CI/CD pipelines for Java-based microservices.

## Tech Stack

### Core Technologies
- **CI/CD Platform**: GitHub Actions
- **Primary Language**: YAML (workflows/actions)
- **Target Projects**: Java (Gradle-based microservices)
- **Container Platform**: Docker
- **Security Scanning**: Trivy
- **Version Control**: Git with semantic versioning

### Supported Java Ecosystem
- **Build Tool**: Gradle
- **Java Distribution**: Temurin
- **Default Java Version**: 21 (configurable)

## Project Structure

```
.github/
  ├── actions/           # Reusable composite actions
  │   ├── gradle-version/    # Read Gradle version
  │   └── compute-next-tag/  # Calculate next semantic version
  ├── workflows/        # Reusable workflows
  │   ├── microservice-pipeline.yml  # Main pipeline for microservices
  │   ├── _build-test-java.yml      # Java build & test
  │   ├── _docker-scan.yml          # Docker security scanning
  │   └── .trivy.yaml               # Trivy configuration
  ├── prompts/          # OpenSpec prompts
  └── skills/           # OpenSpec skills
openspec/              # OpenSpec change management
  ├── config.yaml
  ├── changes/
  └── specs/
```

## Versioning Conventions

### Semantic Versioning
- **Format**: `X.Y.Z-RELEASE`
- **Strategy**: Patch increment by default (Z+1)
- **Tag Pattern**: `*.*.*-RELEASE`
- **Example**: `1.2.3-RELEASE` → `1.2.4-RELEASE`

### Version Validation
- Gradle version MUST match the expected next tag
- PRs are validated to ensure version is incremented correctly
- Last release tag is auto-detected from Git tags

## Coding Conventions

### GitHub Actions Workflows

#### Naming
- **Reusable workflows**: Prefix with `_` (e.g., `_build-test-java.yml`)
- **Main workflows**: Descriptive names (e.g., `microservice-pipeline.yml`)
- **Use kebab-case** for file names

#### Structure
- Use `workflow_call` for reusable workflows
- Define clear `inputs` with defaults
- Use `permissions` declarations (least privilege)
- Organize jobs logically (validate → build → scan → deploy)

#### Best Practices
- Use `set -euo pipefail` in bash scripts
- Fetch full Git history when needed (`fetch-depth: 0`)
- Use `actions/checkout@v4` and pin action versions
- Always fetch tags before version operations
- Use `GITHUB_OUTPUT` for sharing data between steps

### Shell Scripts in Workflows

```bash
# Good example
run: |
  set -euo pipefail
  
  VERSION=$(./gradlew -q printVersion | tr -d '\r' | tail -n 1)
  echo "version=$VERSION" >> "$GITHUB_OUTPUT"
  echo "Gradle version: $VERSION"
```

#### Script Requirements
- Start with `set -euo pipefail` for error handling
- Use clear variable names in UPPERCASE
- Always quote variables: `"$VAR"` not `$VAR`
- Use `|| true` for commands that may fail gracefully
- Provide informative echo statements
- Use `IFS` for proper string splitting

### Pull Request Conventions
- **One commit per PR** (enforced by pipeline)
- Version must be incremented before merge
- Commits should be atomic and complete

### Error Messages
- Use `❌` for errors, `✅` for success
- Include actual vs expected values
- Provide actionable guidance

```bash
echo "❌ Versión no incrementada correctamente."
echo "   - Versión actual: $CURRENT"
echo "   - Tag calculado : $EXPECTED"
echo "Actualiza version en Gradle para que coincida con el tag calculado."
```

## OpenSpec Integration

### Workflow
- Uses **spec-driven** schema (defined in `openspec/config.yaml`)
- Changes tracked in `openspec/changes/`
- Archived changes in `openspec/changes/archive/`
- Specifications in `openspec/specs/`

### Available Skills
- `openspec-new-change` - Start new change
- `openspec-continue-change` - Progress change
- `openspec-ff-change` - Fast-forward through artifacts
- `openspec-apply-change` - Implement tasks
- `openspec-verify-change` - Validate implementation
- `openspec-archive-change` - Archive completed change
- `openspec-sync-specs` - Sync delta specs to main
- `openspec-explore` - Thinking partner mode

## Common Patterns

### Version Validation Flow
1. Fetch all tags from Git
2. Read current version from Gradle
3. Find last release tag (`*.*.*-RELEASE`)
4. Calculate expected next version (patch+1)
5. Validate Gradle version matches expected
6. Fail build if mismatch

### Pipeline Stages
1. **Validate** - Version check, commit count, PR validation
2. **Build** - Compile, test, package
3. **Scan** - Security scanning with Trivy
4. **Artifact** - Upload build artifacts
5. **Deploy** - Conditional deployment to environments

## Domain Knowledge

### Purpose
This repository serves as a centralized source of CI/CD templates for Java microservice projects, ensuring:
- Consistent build processes across projects
- Automated version management
- Security scanning compliance
- Standard deployment workflows

### Target Users
- Development teams building Java microservices
- DevOps engineers setting up CI/CD pipelines
- Projects requiring standardized release processes

## Configuration

### Workflow Customization
Workflows accept inputs for customization:
- `java_version` - Java version to use (default: "21")
- `promote_branch` - Branch for promotion (default: "release")

### Required Repository Settings
- Write permissions on contents (for tagging)
- GitHub Actions enabled
- Conventional commit messages recommended

## Future Considerations
- Support for additional build tools (Maven)
- Multi-environment deployment strategies
- Integration with artifact repositories
- Enhanced security scanning options
- Support for monorepo structures
