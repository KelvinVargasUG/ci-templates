## 1. Create Composite Actions

- [x] 1.1 Create `.github/actions/gradle-version/action.yml` with composite action structure
- [x] 1.2 Add `name` and `description` metadata to gradle-version action
- [x] 1.3 Define `version` output in gradle-version action
- [x] 1.4 Implement bash step to run `./gradlew -q printVersion | tr -d '\r' | tail -n 1`
- [x] 1.5 Write version to `$GITHUB_OUTPUT` in format `version=<value>`
- [x] 1.6 Set `shell: bash` explicitly in gradle-version run step
- [x] 1.7 Create `.github/actions/compute-next-tag/action.yml` with composite action structure
- [x] 1.8 Add `name` and `description` metadata to compute-next-tag action
- [x] 1.9 Define inputs `pattern` (default: `*.*.*-RELEASE`) and `suffix` (default: `-RELEASE`) for compute-next-tag
- [x] 1.10 Define outputs `last_tag` and `expected_tag` for compute-next-tag
- [x] 1.11 Implement `git fetch --force --tags` step with bash and `set -euo pipefail`
- [x] 1.12 Implement tag discovery using `git tag -l "$pattern" --sort=-v:refname | head -n 1`
- [x] 1.13 Add logic to handle no existing tags case (output `expected_tag=0.0.1-RELEASE`)
- [x] 1.14 Parse last tag into major.minor.patch components with validation
- [x] 1.15 Increment patch version by 1 and construct expected tag with suffix
- [x] 1.16 Write both outputs to `$GITHUB_OUTPUT`

## 2. Create Build-Test Reusable Workflow

- [x] 2.1 Create `.github/workflows/_build-test-java.yml` file
- [x] 2.2 Configure `on: workflow_call` trigger
- [x] 2.3 Define input `java_version` with type string and default "21"
- [x] 2.4 Define workflow output `version` that references job output
- [x] 2.5 Create job `build_test_coverage` with `runs-on: ubuntu-latest`
- [x] 2.6 Set job-level permissions to `contents: read`
- [x] 2.7 Define job output `version` referencing gradle_version step
- [x] 2.8 Add checkout step with `actions/checkout@v4`
- [x] 2.9 Add Java setup step with Temurin distribution and Gradle caching
- [x] 2.10 Use `java_version` input for setup-java version parameter
- [x] 2.11 Add step to run gradle-version composite action with id `gradle_version`
- [x] 2.12 Add build step running `./gradlew clean test bootJar jacocoTestReport jacocoTestCoverageVerification`
- [x] 2.13 Use `set -euo pipefail` in all bash steps
- [x] 2.14 Add JAR verification step checking `build/libs/*.jar` exists
- [x] 2.15 Implement JAR verification with ✅/❌ formatted messages
- [x] 2.16 Add Python script step to parse JUnit XML and print test summary (passed/failed/skipped)
- [x] 2.17 Fail workflow if JUnit XML shows failures or errors > 0
- [x] 2.18 Add Python script step to parse `jacocoTestReport.xml` for LINE coverage
- [x] 2.19 Print coverage percentage with covered/missed line counts
- [x] 2.20 Fail workflow if coverage < 90%
- [x] 2.21 Add upload-artifact step for JAR with name `app-jar` and path `build/libs/*.jar`
- [x] 2.22 Add upload-artifact step for reports with name `reports` and paths for test/jacoco reports

## 3. Create Docker-Scan Reusable Workflow

- [x] 3.1 Create `.github/workflows/_docker-scan.yml` file
- [x] 3.2 Configure `on: workflow_call` trigger
- [x] 3.3 Define input `image_name` with type string and default "ms-factura"
- [x] 3.4 Define required input `version` with type string
- [x] 3.5 Create job `docker_build_and_scan` with `runs-on: ubuntu-latest`
- [x] 3.6 Set job permissions: `contents: read`, `security-events: write`, `pull-requests: write`
- [x] 3.7 Add checkout step with `actions/checkout@v4`
- [x] 3.8 Add download-artifact step for `app-jar` to `build/libs/` path
- [x] 3.9 Add verification step to confirm JAR exists after download
- [x] 3.10 Add Docker build step with tag `${image_name}:${version}`
- [x] 3.11 Add Docker tag step for `${image_name}:latest`
- [x] 3.12 Add Trivy scan step using `aquasecurity/trivy-action@0.24.0`
- [x] 3.13 Configure Trivy with `format: sarif`, `output: trivy.sarif`, `exit-code: 0`
- [x] 3.14 Set Trivy options: `ignore-unfixed: true`, `vuln-type: os,library`, `scanners: vuln`
- [x] 3.15 Reference `.trivy.yaml` config file in Trivy scan
- [x] 3.16 Add step to upload SARIF using `github/codeql-action/upload-sarif@v3`
- [x] 3.17 Set SARIF category to `trivy-container`
- [x] 3.18 Add Python script step to parse SARIF and count CRITICAL/HIGH vulnerabilities
- [x] 3.19 Write CRITICAL and HIGH counts plus `has_vulns` boolean to `$GITHUB_OUTPUT`
- [x] 3.20 Add conditional PR comment step when `has_vulns == 'true'` using `actions/github-script@v7`
- [x] 3.21 Generate PR comment with vulnerability counts and link to filtered Code Scanning results
- [x] 3.22 Add gate step that fails job if `has_vulns == 'true'` with ❌ message
- [x] 3.23 Add step summary showing image, tag, CRITICAL/HIGH counts, and gate status

## 4. Create PR Orchestration Workflow

- [x] 4.1 Create `.github/workflows/pr-factura.yml` file
- [x] 4.2 Configure trigger on `pull_request` to branches `develop` and `release`
- [x] 4.3 Add concurrency group `pr-factura-${{ github.ref }}` with `cancel-in-progress: true`
- [x] 4.4 Set workflow-level permissions to `contents: write`
- [x] 4.5 Create job `validate_pr` with `runs-on: ubuntu-latest`
- [x] 4.6 Add checkout step with `fetch-depth: 0` for full Git history
- [x] 4.7 Add Java setup step with Temurin distribution and Java 21
- [x] 4.8 Add step to fetch Git tags using `git fetch --force --tags`
- [x] 4.9 Add step using gradle-version action to read current version
- [x] 4.10 Add step using compute-next-tag action to calculate expected version
- [x] 4.11 Add validation step comparing Gradle version against expected tag
- [x] 4.12 Fail with ❌ message if versions don't match, showing current vs expected
- [x] 4.13 Add validation step for single commit in PR using base and head SHAs
- [x] 4.14 Calculate commit count using `git rev-list --count "$BASE_SHA..$HEAD_SHA"`
- [x] 4.15 Fail with ❌ message if commit count != 1
- [x] 4.16 Use ✅ messages for successful validations
- [x] 4.17 Create job `build_test` that uses `_build-test-java.yml` workflow
- [x] 4.18 Set `needs: validate_pr` on build_test job
- [x] 4.19 Create job `docker_scan` that uses `_docker-scan.yml` workflow
- [x] 4.20 Set `needs: build_test` on docker_scan job
- [x] 4.21 Pass build_test version output to docker_scan as version input

## 5. Validation and Testing

- [x] 5.1 Verify all 5 files are created in correct locations
- [x] 5.2 Check YAML syntax is valid for all workflow and action files
- [x] 5.3 Verify all input/output contracts match between workflows
- [x] 5.4 Confirm artifact names (`app-jar`, `reports`) are preserved
- [x] 5.5 Validate permissions are minimal and correctly scoped per job
- [x] 5.6 Ensure all bash scripts use `set -euo pipefail`
- [x] 5.7 Verify error messages consistently use ❌/✅ formatting
- [x] 5.8 Confirm backward compatibility: existing `microservice-pipeline.yml` unchanged
- [x] 5.9 Test composite actions can be called independently
- [x] 5.10 Validate workflow dependencies chain correctly (validate → build → scan)
