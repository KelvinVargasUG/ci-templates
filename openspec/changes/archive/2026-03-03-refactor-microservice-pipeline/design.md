## Context

The current `microservice-pipeline.yml` is a single, monolithic workflow that handles PR validation, build/test, and Docker scanning. It uses `workflow_call` but contains pull_request-specific operations (like PR commit counting and commenting), creating event-context conflicts that limit reusability.

**Current state**:
- 395 lines in a single workflow file
- Mixes PR-specific logic with reusable build logic
- Duplicates version reading logic across multiple steps
- No clear separation between orchestration and execution
- Hard to test individual components in isolation

**Constraints**:
- Must maintain all existing validations (version, commit count, coverage, security)
- Cannot change artifact names (`app-jar`, `reports`) - consumers depend on them
- Must preserve security posture (Trivy blocking, SARIF upload)
- Should remain backward compatible for gradual migration

## Goals / Non-Goals

**Goals**:
- Separate PR orchestration from reusable build/scan workflows
- Extract version/tag logic into testable composite actions
- Enable workflow reuse across different trigger types (push, workflow_dispatch, etc.)
- Reduce code duplication while maintaining all functionality
- Improve maintainability through single-responsibility components
- Provide clear input/output contracts for each component

**Non-Goals**:
- Publishing Docker images to registries (out of scope)
- Supporting non-Gradle build tools in this iteration
- Modifying the actual build/test/scan behavior
- Changing coverage thresholds or security policies
- Migrating existing consumers of `microservice-pipeline.yml`

## Decisions

### Decision 1: Three-tier architecture (orchestration → workflows → actions)

**Choice**: Split into PR orchestrator + 2 reusable workflows + 2 composite actions

**Rationale**:
- **PR orchestrator** (`pr-factura.yml`): Handles PR-specific concerns (commit validation, version checking) and calls reusable workflows. This tier is event-aware.
- **Reusable workflows** (`_build-test-java.yml`, `_docker-scan.yml`): Event-agnostic, can be called from any workflow type. Accept inputs, produce outputs/artifacts.
- **Composite actions** (`gradle-version`, `compute-next-tag`): Encapsulate single-purpose logic for reuse across workflows and jobs.

**Alternatives considered**:
- *Single reusable workflow with conditionals*: Would still mix concerns and make event handling complex
- *More granular workflows (separate test/coverage/jar)*: Over-engineered for current needs, harder to orchestrate
- *No composite actions, inline logic*: Would duplicate version/tag logic across 3+ workflows

**Why this approach**: Clear separation of concerns, testable units, maximum reusability without over-engineering.

### Decision 2: Composite actions for version operations

**Choice**: Create `gradle-version` and `compute-next-tag` as composite actions (not reusable workflows)

**Rationale**:
- These are single-purpose utilities called within jobs, not full pipelines
- Composite actions run in the same job context (faster, share filesystem)
- Can be used by both PR orchestrator and reusable workflows
- Easier to test: just run the action in isolation

**Alternatives considered**:
- *Reusable workflows*: Overkill for simple operations, slower (separate jobs)
- *Bash functions in each workflow*: Code duplication, hard to test

### Decision 3: Build workflow outputs version for Docker workflow

**Choice**: `_build-test-java.yml` declares output `version`, consumed by `_docker-scan.yml` via inputs

**Rationale**:
- Docker workflow needs version to tag images
- Reading version once (in build) avoids duplication and ensures consistency
- Makes dependency explicit: Docker scan requires build to complete first
- Enables workflows to be chained with clear data flow

**Implementation**:
```yaml
# _build-test-java.yml
outputs:
  version:
    value: ${{ jobs.build_test_coverage.outputs.version }}

jobs:
  build_test_coverage:
    outputs:
      version: ${{ steps.gradle_version.outputs.version }}
    steps:
      - uses: ./.github/actions/gradle-version
        id: gradle_version
```

### Decision 4: Artifact download in Docker workflow (not build upload + download)

**Choice**: Build workflow uploads artifacts, Docker workflow downloads them

**Rationale**:
- Maintains separation: build creates JARs, Docker consumes them
- Artifacts are needed for potential deployment steps later
- Standard GitHub Actions pattern for sharing files between jobs
- Enables parallel workflows if needed in future

**Alternatives considered**:
- *Build directly in Docker job*: Violates separation, can't reuse build workflow separately
- *Shared workspace*: Only works within same workflow run, not across workflows

### Decision 5: Python for XML/JSON parsing, not external actions

**Choice**: Keep inline Python scripts for parsing test results, JaCoCo reports, and SARIF

**Rationale**:
- Already working and battle-tested
- No external dependencies to version or maintain
- Fast execution (no action download overhead)
- Full control over parsing logic and error messages

**Alternatives considered**:
- *xmllint/jq in bash*: Less readable, harder to maintain complex logic
- *Third-party parsing actions*: Additional dependencies, version management overhead

### Decision 6: Trivy dual-output strategy (SARIF + gate)

**Choice**: Generate SARIF for Code Scanning, then parse SARIF for gating logic

**Rationale**:
- SARIF upload is required for GitHub Security integration
- Trivy's native exit-code gating doesn't provide counts for PR comments
- Parsing SARIF gives us structured data for both gating and reporting
- Single scan, multiple uses (upload + gate + comment)

**Implementation flow**:
1. Run Trivy with `exit-code: 0` and `format: sarif`
2. Upload SARIF to Code Scanning (always succeeds)
3. Parse SARIF with Python to count CRITICAL/HIGH
4. Generate PR comment with counts and link
5. Gate: fail job if CRITICAL/HIGH > 0

### Decision 7: Minimal permissions per job

**Choice**: Each job declares only the permissions it needs

**Rationale**:
- Security best practice: least privilege
- Makes permission requirements explicit and auditable
- Prevents accidental permission escalation

**Permission mapping**:
- PR validation job: `contents: read` (fetch repo)
- Build job: `contents: read` (fetch repo, cache)
- Docker/scan job: `contents: read`, `security-events: write` (SARIF), `pull-requests: write` (comments)

### Decision 8: Concurrency control at PR level

**Choice**: `concurrency: group: pr-factura-${{ github.ref }}, cancel-in-progress: true`

**Rationale**:
- Prevents multiple runs for same PR (saves CI resources)
- Auto-cancels outdated runs when new commits pushed
- Standard pattern for PR workflows

### Decision 9: Preserve existing artifact and report names

**Choice**: Keep `app-jar` and `reports` as artifact names, same paths

**Rationale**:
- Backward compatibility: downstream workflows may depend on these names
- No need to change if already working
- Future workflows can easily consume these artifacts

## Risks / Trade-offs

### Risk: Breaking existing consumers of microservice-pipeline.yml
**Mitigation**: Do not delete the monolithic workflow. Keep it for backward compatibility. Document migration path in README. New projects use modular approach, existing projects migrate when ready.

### Risk: Composite actions can't be tested via workflow_dispatch
**Mitigation**: Create test workflows that call the actions. Use act or similar for local testing. Actions are simple enough that integration tests (calling workflows) provide sufficient coverage.

### Risk: Output passing between workflows is fragile
**Mitigation**: Use explicit input validation in consuming workflows. Fail fast with clear error messages if required inputs missing. Document input/output contracts in workflow headers.

### Risk: More files to maintain (5 instead of 1)
**Trade-off accepted**: Complexity shifts from one large file to multiple small files. Each file is simpler and single-purpose. Benefits (reusability, testability) outweigh maintenance overhead.

### Risk: Version mismatch between build and Docker workflows
**Mitigation**: Docker workflow receives version as required input from build workflow output. Single source of truth. Validation step in Docker workflow verifies version matches JAR file.

### Risk: Python script failures harder to debug in CI
**Mitigation**: Keep Python scripts simple and readable. Add extensive error messages with ❌/✅ formatting. Print intermediate values for debugging. Test scripts locally before committing.

### Risk: Migration path unclear for teams
**Mitigation**: Document in project README with example. Provide side-by-side comparison of old vs new workflow. Create migration guide showing how to replace monolithic workflow with orchestrator + calls to reusable workflows.

## Migration Plan

**Phase 1: Create new modular components**
1. Create composite actions (gradle-version, compute-next-tag)
2. Create reusable workflows (_build-test-java.yml, _docker-scan.yml)
3. Create PR orchestrator (pr-factura.yml)
4. Test all components together

**Phase 2: Validation**
1. Keep existing microservice-pipeline.yml active
2. Run both old and new pipelines in parallel (if possible)
3. Compare results: coverage %, vulnerabilities found, version validation
4. Fix any discrepancies

**Phase 3: Documentation**
1. Document each component's inputs/outputs
2. Create migration guide
3. Add examples of calling reusable workflows from other trigger types

**Phase 4: Future deprecation** (not in this change)
1. Mark microservice-pipeline.yml as deprecated
2. Update consuming projects to use new modular approach
3. Remove deprecated workflow after migration period

**Rollback strategy**:
- New files are additive (no deletions)
- If issues found, simply don't use pr-factura.yml
- Existing microservice-pipeline.yml continues working unchanged
- Zero-risk deployment

## Open Questions

None currently. All technical decisions are documented above and ready for implementation.
