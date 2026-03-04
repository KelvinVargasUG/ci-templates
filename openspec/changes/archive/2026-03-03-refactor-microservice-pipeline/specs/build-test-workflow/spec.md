## ADDED Requirements

### Requirement: Workflow must be reusable via workflow_call
The build-test workflow SHALL be callable from other workflows using the `workflow_call` event trigger.

#### Scenario: Called from PR workflow
- **WHEN** a workflow calls this workflow using `uses: ./.github/workflows/_build-test-java.yml`
- **THEN** the workflow executes successfully and returns outputs

### Requirement: Java version must be configurable
The workflow SHALL accept a `java_version` input parameter with a default value of "21".

#### Scenario: Default Java version used
- **WHEN** workflow is called without specifying java_version
- **THEN** Java 21 is used for the build

#### Scenario: Custom Java version specified
- **WHEN** workflow is called with `java_version: "17"`
- **THEN** Java 17 is used for the build

### Requirement: Gradle version must be output
The workflow SHALL output the Gradle project version as a workflow output named `version`.

#### Scenario: Version output is available
- **WHEN** the build job completes successfully
- **THEN** the workflow outputs the version read from Gradle

### Requirement: Build must compile and package application
The workflow SHALL execute `./gradlew clean test bootJar` to build and package the application.

#### Scenario: Successful build
- **WHEN** Gradle build executes
- **THEN** a JAR file is created in `build/libs/` directory

#### Scenario: Build failure
- **WHEN** Gradle build fails
- **THEN** the workflow job fails with non-zero exit code

### Requirement: Tests must execute with JaCoCo coverage
The workflow SHALL run tests with JaCoCo coverage using `jacocoTestReport` and `jacocoTestCoverageVerification` tasks.

#### Scenario: Tests pass with sufficient coverage
- **WHEN** tests run and coverage meets 90% threshold
- **THEN** workflow continues successfully

#### Scenario: Coverage below threshold
- **WHEN** tests run but coverage is below 90%
- **THEN** workflow fails at jacocoTestCoverageVerification step

### Requirement: JAR artifact must be uploaded
The workflow SHALL upload the generated JAR file as a workflow artifact named `app-jar` from path `build/libs/*.jar`.

#### Scenario: JAR artifact available for downstream jobs
- **WHEN** build completes successfully
- **THEN** artifact named `app-jar` is available for download in subsequent jobs or workflows

### Requirement: Test and coverage reports must be uploaded
The workflow SHALL upload test results and JaCoCo reports as a workflow artifact named `reports`.

#### Scenario: Reports uploaded successfully
- **WHEN** tests complete
- **THEN** artifact named `reports` contains test results from `**/build/reports/tests/test/**` and JaCoCo reports from `**/build/reports/jacoco/test/**`

### Requirement: JAR existence must be verified
The workflow SHALL verify that at least one JAR file exists in `build/libs/` directory after build.

#### Scenario: JAR exists
- **WHEN** build completes and JAR is found
- **THEN** verification step succeeds with ✅ message

#### Scenario: No JAR produced
- **WHEN** build completes but no JAR found
- **THEN** verification step fails with ❌ error message

### Requirement: Test summary must be generated
The workflow SHALL parse JUnit XML test results and display summary of passed/failed/skipped tests.

#### Scenario: Test summary displayed
- **WHEN** tests complete and XML results exist
- **THEN** workflow prints total tests, passed, failures, errors, and skipped counts with ✅/❌ formatting

#### Scenario: Tests have failures
- **WHEN** JUnit XML shows failures or errors count > 0
- **THEN** workflow fails with ❌ message indicating test failures

### Requirement: Coverage percentage must be displayed
The workflow SHALL parse `jacocoTestReport.xml` and display LINE coverage percentage.

#### Scenario: Coverage percentage shown
- **WHEN** JaCoCo report is generated
- **THEN** workflow prints line coverage percentage with covered and missed line counts

#### Scenario: Coverage below 90%
- **WHEN** line coverage is less than 90%
- **THEN** workflow fails with ❌ message

### Requirement: Gradle cache must be configured
The workflow SHALL use `actions/setup-java` with Gradle caching enabled to improve build performance.

#### Scenario: Cache is used
- **WHEN** Java setup step executes
- **THEN** Gradle dependencies are cached between workflow runs

### Requirement: Workflow must use minimal permissions
The workflow SHALL only request `contents: read` permission at the job level.

#### Scenario: No excessive permissions
- **WHEN** workflow executes
- **THEN** only read access to repository contents is granted
