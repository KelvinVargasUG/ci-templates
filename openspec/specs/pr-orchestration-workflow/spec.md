# PR-Orchestration Workflow

## Purpose

Pull request workflow that orchestrates validation, build, and security scanning for Java microservices. Enforces version increment validation, single-commit policy, and coordinates execution of reusable build-test and docker-scan workflows.

## Requirements

### Requirement: Workflow must trigger on pull requests
The workflow SHALL trigger on `pull_request` events targeting `develop` and `release` branches.

#### Scenario: PR to develop branch
- **WHEN** pull request is opened to develop branch
- **THEN** workflow executes

#### Scenario: PR to release branch
- **WHEN** pull request is opened to release branch
- **THEN** workflow executes

#### Scenario: PR to other branches
- **WHEN** pull request is opened to feature branch
- **THEN** workflow does not execute

### Requirement: Concurrent runs must be managed per PR
The workflow SHALL use concurrency control with group `pr-factura-${{ github.ref }}` and `cancel-in-progress: true`.

#### Scenario: New commit pushed to PR
- **WHEN** PR already has a running workflow and new commit is pushed
- **THEN** old workflow run is cancelled and new run starts

### Requirement: Single commit must be enforced
The workflow SHALL validate that the PR contains exactly 1 commit from base to head.

#### Scenario: PR has one commit
- **WHEN** PR contains exactly 1 commit
- **THEN** validation passes with ✅ message

#### Scenario: PR has multiple commits
- **WHEN** PR contains more than 1 commit
- **THEN** validation fails with ❌ message stating exact commit count

### Requirement: Version must match expected next tag
The workflow SHALL validate that the Gradle version matches the calculated expected next tag.

#### Scenario: Version correctly incremented
- **WHEN** last tag is `1.2.3-RELEASE` and Gradle version is `1.2.4-RELEASE`
- **THEN** validation passes with ✅ message

#### Scenario: Version not incremented
- **WHEN** Gradle version does not match expected next tag
- **THEN** validation fails with ❌ message showing current vs expected versions

#### Scenario: No previous tags exist
- **WHEN** no release tags exist and Gradle version is `0.0.1-RELEASE`
- **THEN** validation passes

### Requirement: Repository must be checked out with full history
The workflow SHALL use `actions/checkout` with `fetch-depth: 0` to enable version validation.

#### Scenario: Full history available
- **WHEN** checkout step executes
- **THEN** all Git history and tags are available for version calculation

### Requirement: Build and test workflow must be called
The workflow SHALL call the reusable build-test workflow as a separate job.

#### Scenario: Build workflow called
- **WHEN** validation job succeeds
- **THEN** build-test workflow is invoked via `uses: ./.github/workflows/_build-test-java.yml`

### Requirement: Build job must depend on validation
The build job SHALL use `needs: validate_pr` to ensure validation completes first.

#### Scenario: Validation passes
- **WHEN** validation job succeeds
- **THEN** build job starts

#### Scenario: Validation fails
- **WHEN** validation job fails
- **THEN** build job is skipped

### Requirement: Docker scan workflow must be called
The workflow SHALL call the reusable Docker scan workflow as a separate job.

#### Scenario: Docker workflow called
- **WHEN** build job succeeds
- **THEN** Docker scan workflow is invoked via `uses: ./.github/workflows/_docker-scan.yml`

### Requirement: Docker job must depend on build
The Docker scan job SHALL use `needs: build_test` to ensure build completes first.

#### Scenario: Build passes
- **WHEN** build job succeeds
- **THEN** Docker scan job starts

#### Scenario: Build fails
- **WHEN** build job fails
- **THEN** Docker scan job is skipped

### Requirement: Version must be passed from build to Docker
The workflow SHALL pass the version output from build workflow to Docker workflow as input.

#### Scenario: Version propagated
- **WHEN** build workflow outputs version `1.2.3-RELEASE`
- **THEN** Docker workflow receives `version: 1.2.3-RELEASE` as input

### Requirement: Workflow must use minimal permissions
The workflow SHALL declare minimal permissions at workflow level, allowing jobs to override as needed.

#### Scenario: Base permissions set
- **WHEN** workflow executes
- **THEN** workflow-level permissions are `contents: write` for potential tagging operations

### Requirement: Validation job must use compute-next-tag action
The workflow SHALL use the compute-next-tag composite action to calculate the expected version.

#### Scenario: Action called for tag calculation
- **WHEN** validation job runs
- **THEN** compute-next-tag action is invoked and outputs are used for comparison

### Requirement: Validation job must use gradle-version action
The workflow SHALL use the gradle-version composite action to read the current Gradle version.

#### Scenario: Action called for version reading
- **WHEN** validation job runs
- **THEN** gradle-version action is invoked and output is compared against expected tag

### Requirement: Commit count must be calculated from base to head SHA
The workflow SHALL use `github.event.pull_request.base.sha` and `github.event.pull_request.head.sha` to calculate commit count.

#### Scenario: Commit count accurate
- **WHEN** PR has specific base and head commits
- **THEN** commit count is calculated using `git rev-list --count "$BASE_SHA..$HEAD_SHA"`

### Requirement: Error messages must use standard formatting
The workflow SHALL use ❌ for errors and ✅ for success messages consistently.

#### Scenario: Validation passes
- **WHEN** all validations succeed
- **THEN** messages include ✅ prefix

#### Scenario: Validation fails
- **WHEN** validation fails
- **THEN** messages include ❌ prefix with actionable guidance
