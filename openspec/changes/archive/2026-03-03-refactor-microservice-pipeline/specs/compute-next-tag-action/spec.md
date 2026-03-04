## ADDED Requirements

### Requirement: Action must be a composite action
The compute-next-tag action SHALL be implemented as a composite action using `runs.using: composite`.

#### Scenario: Action executes as composite
- **WHEN** action is called from a workflow step
- **THEN** action executes in the same job context as the caller

### Requirement: Pattern input must be configurable with default
The action SHALL accept an input `pattern` with default value `*.*.*-RELEASE` for filtering Git tags.

#### Scenario: Default pattern used
- **WHEN** action is called without pattern input
- **THEN** tags matching `*.*.*-RELEASE` are considered

#### Scenario: Custom pattern provided
- **WHEN** action is called with `pattern: "*.*.*-RC"`
- **THEN** tags matching `*.*.*-RC` are considered

### Requirement: Suffix input must be configurable with default
The action SHALL accept an input `suffix` with default value `-RELEASE` for constructing new tags.

#### Scenario: Default suffix used
- **WHEN** action is called without suffix input
- **THEN** expected tag is formatted with `-RELEASE` suffix

#### Scenario: Custom suffix provided
- **WHEN** action is called with `suffix: "-RC"`
- **THEN** expected tag is formatted with `-RC` suffix

### Requirement: Last tag must be output
The action SHALL provide an output named `last_tag` containing the most recent tag matching the pattern.

#### Scenario: Last tag found
- **WHEN** Git repository has tags matching pattern
- **THEN** output `last_tag` contains the highest semantic version tag

#### Scenario: No tags exist
- **WHEN** Git repository has no tags matching pattern
- **THEN** output `last_tag` is empty

### Requirement: Expected next tag must be output
The action SHALL provide an output named `expected_tag` containing the calculated next semantic version.

#### Scenario: Next tag calculated from existing tag
- **WHEN** last tag is `1.2.3-RELEASE`
- **THEN** output `expected_tag` is `1.2.4-RELEASE`

#### Scenario: First tag when none exist
- **WHEN** no tags exist matching pattern
- **THEN** output `expected_tag` is `0.0.1-RELEASE`

### Requirement: Tags must be fetched before analysis
The action SHALL execute `git fetch --force --tags` to ensure all tags are available locally.

#### Scenario: Tags fetched successfully
- **WHEN** action executes
- **THEN** all remote tags are available for analysis

### Requirement: Latest tag must be found by semantic version sort
The action SHALL use `git tag -l "$pattern" --sort=-v:refname | head -n 1` to find the most recent semantic version tag.

#### Scenario: Tags sorted correctly
- **WHEN** repository has tags 1.2.3-RELEASE, 1.10.0-RELEASE, 1.9.5-RELEASE
- **THEN** last_tag is 1.10.0-RELEASE (not 1.9.5-RELEASE)

### Requirement: Patch version must be incremented
The action SHALL parse the last tag as `major.minor.patch`, increment patch by 1, and construct expected tag.

#### Scenario: Patch incremented
- **WHEN** last tag is `2.5.9-RELEASE`
- **THEN** expected tag is `2.5.10-RELEASE`

### Requirement: Version parsing must validate format
The action SHALL validate that the tag can be parsed into major, minor, and patch components.

#### Scenario: Valid tag format
- **WHEN** last tag is `1.2.3-RELEASE`
- **THEN** parsing succeeds and extracts major=1, minor=2, patch=3

#### Scenario: Invalid tag format
- **WHEN** last tag is `invalid-tag`
- **THEN** action fails with error message indicating parsing failure

### Requirement: Suffix must be preserved in expected tag
The action SHALL append the configured suffix to the incremented version.

#### Scenario: Suffix appended
- **WHEN** last tag is `1.0.5-RELEASE` and suffix is `-RELEASE`
- **THEN** expected tag is `1.0.6-RELEASE`

### Requirement: Outputs must be written to GITHUB_OUTPUT
The action SHALL write both outputs to `$GITHUB_OUTPUT` in the format `last_tag=<value>` and `expected_tag=<value>`.

#### Scenario: Outputs set correctly
- **WHEN** action completes successfully
- **THEN** GITHUB_OUTPUT file contains both last_tag and expected_tag entries

### Requirement: Action must use bash with error handling
The action SHALL use `shell: bash` with `set -euo pipefail` for proper error propagation.

#### Scenario: Error handling enabled
- **WHEN** any command in the action fails
- **THEN** action immediately fails without continuing execution
