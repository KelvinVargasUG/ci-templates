# Gradle-Version Action

## Purpose

Composite GitHub Action that reads the project version from Gradle and makes it available as an output for use in workflows. Provides a consistent way to retrieve version information across multiple workflow steps and jobs.

## Requirements

### Requirement: Action must be a composite action
The gradle-version action SHALL be implemented as a composite action using `runs.using: composite`.

#### Scenario: Action executes as composite
- **WHEN** action is called from a workflow step
- **THEN** action executes in the same job context as the caller

### Requirement: Action must output version
The action SHALL provide an output named `version` containing the Gradle project version.

#### Scenario: Version output accessible
- **WHEN** action completes successfully
- **THEN** output `version` is available via `steps.<step-id>.outputs.version`

### Requirement: Version must be read from Gradle
The action SHALL execute `./gradlew -q printVersion` to retrieve the project version.

#### Scenario: Gradle command executes
- **WHEN** action runs
- **THEN** `./gradlew -q printVersion` is executed

### Requirement: Version output must be sanitized
The action SHALL remove carriage returns and extract the last line of Gradle output using `tr -d '\r' | tail -n 1`.

#### Scenario: Clean version output
- **WHEN** Gradle outputs version with potential noise
- **THEN** action extracts only the version string without whitespace or control characters

### Requirement: Version must be written to GITHUB_OUTPUT
The action SHALL write the version to `$GITHUB_OUTPUT` in the format `version=<value>`.

#### Scenario: Output variable set
- **WHEN** version is retrieved successfully
- **THEN** `version=X.Y.Z-RELEASE` is appended to GITHUB_OUTPUT file

### Requirement: Action must use shell run step
The action SHALL use `shell: bash` for executing the version retrieval command.

#### Scenario: Bash shell used
- **WHEN** action executes
- **THEN** commands run in bash shell with proper error handling

### Requirement: Action must explicitly set shell in composite action
The action SHALL specify `shell: bash` in the run step since composite actions require explicit shell declaration.

#### Scenario: Shell specified for composite
- **WHEN** action is defined
- **THEN** run step includes `shell: bash`

### Requirement: Action must handle Gradle execution errors
The action SHALL fail if `./gradlew printVersion` exits with non-zero code.

#### Scenario: Gradle command fails
- **WHEN** Gradle cannot determine version
- **THEN** action fails and workflow execution stops
