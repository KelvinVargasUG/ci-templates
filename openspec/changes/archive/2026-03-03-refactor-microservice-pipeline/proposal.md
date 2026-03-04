## Why

The current `microservice-pipeline.yml` workflow is monolithic and contains PR-specific validations (pull_request event dependencies) within a workflow_call context, creating event-context conflicts. This prevents proper reusability across different trigger types and makes the pipeline difficult to maintain and extend. Modularizing into separate, focused workflows and composite actions will improve maintainability, reusability, and testability.

## What Changes

- **Split monolithic workflow into modular components**:
  - Extract build/test/coverage logic into a reusable workflow (`_build-test-java.yml`)
  - Extract Docker build/Trivy scanning into a reusable workflow (`_docker-scan.yml`)
  - Create PR orchestration workflow (`pr-factura.yml`) handling PR-specific validations
  - Extract common version/tag operations into composite actions

- **New composite actions for shared logic**:
  - `gradle-version`: Read and output Gradle version
  - `compute-next-tag`: Calculate expected next semantic version tag

- **Improved separation of concerns**:
  - PR validations only in PR-triggered workflows
  - Reusable workflows are event-agnostic
  - Minimal permissions per job
  - Clear input/output contracts

- **Maintained functionality**:
  - Version validation (Gradle vs expected tag)
  - Single-commit PR enforcement
  - JaCoCo 90% coverage requirement
  - Trivy HIGH/CRITICAL vulnerability blocking
  - Test and coverage reporting

## Capabilities

### New Capabilities
- `build-test-workflow`: Reusable Java build/test workflow with Gradle, JaCoCo coverage reporting, and artifact uploading
- `docker-scan-workflow`: Reusable Docker build and Trivy security scanning workflow with SARIF upload and vulnerability gating
- `gradle-version-action`: Composite action to read Gradle project version
- `compute-next-tag-action`: Composite action to calculate expected next semantic version tag
- `pr-orchestration-workflow`: PR-specific workflow orchestrating validation, build, and scan jobs

### Modified Capabilities
<!-- No existing spec requirements are changing - this is net-new modularization -->

## Impact

**Files to create**:
- `.github/workflows/pr-factura.yml`
- `.github/workflows/_build-test-java.yml`
- `.github/workflows/_docker-scan.yml`
- `.github/actions/gradle-version/action.yml`
- `.github/actions/compute-next-tag/action.yml`

**Files to deprecate** (future):
- `.github/workflows/microservice-pipeline.yml` (can remain for backward compatibility initially)

**Backward compatibility**:
- Existing projects using `microservice-pipeline.yml` can migrate incrementally
- New modular workflows can coexist with legacy workflow
- Same artifact names and paths maintained (`app-jar`, `reports`)
- No changes to required permissions or secrets

**Benefits**:
- Each workflow/action has single responsibility
- Easier to test individual components
- Better reusability across different projects and trigger types
- Clearer permission scoping per job
- Reduced code duplication
