# Docker-Scan Workflow

## Purpose

Reusable GitHub Actions workflow for building Docker images and performing security scanning with Trivy. Uploads security findings to GitHub Code Scanning and enforces quality gates by failing on HIGH/CRITICAL vulnerabilities.

## Requirements

### Requirement: Workflow must be reusable via workflow_call
The Docker scan workflow SHALL be callable from other workflows using the `workflow_call` event trigger.

#### Scenario: Called from PR workflow
- **WHEN** a workflow calls this workflow using `uses: ./.github/workflows/_docker-scan.yml`
- **THEN** the workflow executes successfully

### Requirement: Image name must be configurable with default
The workflow SHALL accept an `image_name` input parameter with a default value of "ms-factura".

#### Scenario: Default image name used
- **WHEN** workflow is called without specifying image_name
- **THEN** Docker images are tagged with "ms-factura"

#### Scenario: Custom image name specified
- **WHEN** workflow is called with `image_name: "custom-service"`
- **THEN** Docker images are tagged with "custom-service"

### Requirement: Version must be required input
The workflow SHALL require a `version` input parameter for tagging Docker images.

#### Scenario: Version provided
- **WHEN** workflow is called with `version: "1.2.3-RELEASE"`
- **THEN** Docker image is tagged with version "1.2.3-RELEASE"

### Requirement: JAR artifact must be downloaded
The workflow SHALL download the `app-jar` artifact to `build/libs/` directory before building Docker image.

#### Scenario: JAR downloaded successfully
- **WHEN** download artifact step executes
- **THEN** JAR file exists in `build/libs/` directory

#### Scenario: JAR not available
- **WHEN** app-jar artifact cannot be downloaded
- **THEN** workflow fails with ❌ error message

### Requirement: Docker image must be built with version tag
The workflow SHALL build a Docker image tagged as `${image_name}:${version}`.

#### Scenario: Image built with version
- **WHEN** Docker build executes with image_name "ms-factura" and version "1.0.0-RELEASE"
- **THEN** image "ms-factura:1.0.0-RELEASE" is created

### Requirement: Docker image must be tagged as latest
The workflow SHALL tag the built Docker image as `${image_name}:latest`.

#### Scenario: Latest tag created
- **WHEN** Docker build completes
- **THEN** image is also tagged as `${image_name}:latest`

### Requirement: Trivy scan must generate SARIF output
The workflow SHALL run Trivy security scan and generate SARIF format output to `trivy.sarif`.

#### Scenario: SARIF file generated
- **WHEN** Trivy scan executes
- **THEN** `trivy.sarif` file is created with vulnerability findings

### Requirement: Trivy scan must not fail on vulnerabilities initially
The workflow SHALL run Trivy with `exit-code: 0` to ensure SARIF upload happens regardless of findings.

#### Scenario: Scan completes with vulnerabilities
- **WHEN** Trivy finds vulnerabilities
- **THEN** Trivy scan step succeeds but generates SARIF with findings

### Requirement: SARIF must be uploaded to Code Scanning
The workflow SHALL upload the SARIF file to GitHub Security Code Scanning.

#### Scenario: SARIF uploaded successfully
- **WHEN** Trivy scan completes
- **THEN** SARIF is uploaded and visible in GitHub Security tab

### Requirement: SARIF must be parsed for vulnerability counts
The workflow SHALL parse the SARIF file to count CRITICAL and HIGH severity vulnerabilities.

#### Scenario: Vulnerabilities counted
- **WHEN** SARIF file contains vulnerabilities
- **THEN** script outputs counts for CRITICAL and HIGH severities

### Requirement: Workflow must fail on CRITICAL or HIGH vulnerabilities
The workflow SHALL fail the job if any CRITICAL or HIGH severity vulnerabilities are detected.

#### Scenario: CRITICAL vulnerabilities found
- **WHEN** SARIF parsing detects CRITICAL vulnerabilities > 0
- **THEN** workflow fails with ❌ message

#### Scenario: HIGH vulnerabilities found
- **WHEN** SARIF parsing detects HIGH vulnerabilities > 0
- **THEN** workflow fails with ❌ message

#### Scenario: Only LOW or MEDIUM vulnerabilities
- **WHEN** SARIF parsing finds no CRITICAL or HIGH vulnerabilities
- **THEN** workflow continues successfully

### Requirement: PR must receive comment on vulnerabilities
The workflow SHALL comment on the PR with vulnerability counts and link to Code Scanning when vulnerabilities are found.

#### Scenario: Comment posted with vulnerability details
- **WHEN** HIGH or CRITICAL vulnerabilities are detected on a pull_request
- **THEN** PR receives a comment with CRITICAL count, HIGH count, and filtered link to Security → Code scanning

#### Scenario: No comment on clean scan
- **WHEN** no HIGH or CRITICAL vulnerabilities are detected
- **THEN** no comment is posted to the PR

### Requirement: Trivy configuration must be used
The workflow SHALL use Trivy configuration from `.trivy.yaml` file with `ignore-unfixed: true` and scan types `os,library`.

#### Scenario: Trivy config applied
- **WHEN** Trivy scan executes
- **THEN** settings from `.trivy.yaml` are applied including ignoring unfixed vulnerabilities

### Requirement: Workflow must use scoped permissions
The workflow SHALL request `contents: read`, `security-events: write`, and `pull-requests: write` permissions.

#### Scenario: Minimal permissions granted
- **WHEN** workflow executes
- **THEN** only necessary permissions for SARIF upload and PR comments are granted

### Requirement: Step summary must show scan results
The workflow SHALL generate a step summary showing image name, tag, CRITICAL count, HIGH count, and gate status.

#### Scenario: Summary displayed
- **WHEN** Docker scan completes
- **THEN** GitHub Actions summary shows vulnerability counts and pass/fail status
