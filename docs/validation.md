# Validation

[← Back to README](../README.md) | [← Implementation](implementation.md) | [Architecture →](architecture.md)

## 1. Validation Overview

The purpose of validation is to prove that the GitLab CI/CD implementation behaves as intended from source change through container publication.

The validation model is:

```text
Source Change
     │
     ▼
Pipeline Trigger
     │
     ▼
Build
     │
     ├───────────────┐
     ▼               ▼
Test          Security Scan
     │               │
     │               └──► Trivy Report
     │
     └───────────────┐
                     ▼
              Docker Build
                     │
                     ▼
                  Push
                     │
                     ▼
          GitLab Container Registry
```

The validation covers:

- pipeline execution
- build success
- Maven test and Checkstyle execution
- GitLab `rules` behavior
- merge-request pipeline behavior
- parallel test/security execution
- Trivy scan execution
- build artifact preservation
- security artifact preservation
- Docker image construction
- registry authentication
- Docker image publication
- commit-SHA image tagging
- failure notification behavior

The project does **not** claim runtime application validation, Kubernetes deployment validation, or production operational validation.

---

## 2. Validation Philosophy

Validation is divided into four levels:

```text
Level 1 — Pipeline
       ↓
Did GitLab execute the expected jobs?

Level 2 — Job
       ↓
Did each job perform its intended operation?

Level 3 — Artifact / Registry
       ↓
Did the expected outputs actually exist?

Level 4 — Traceability
       ↓
Can outputs be associated with the source revision?
```

This prevents relying only on a green pipeline indicator.

A successful pipeline is necessary, but individual outputs must also be inspected.

---

## 3. Validation Matrix

| Capability | Validation Method | Expected Result |
|---|---|---|
| Pipeline configuration | GitLab pipeline execution | Pipeline created successfully |
| Build | Inspect `build-job` | Maven build succeeds |
| WAR generation | Inspect build artifact | `target/*.war` exists |
| Testing | Inspect `test-job` | `mvn test` succeeds |
| Checkstyle | Inspect `test-job` logs | Checkstyle command completes |
| Rules | Push to `main` | Pipeline runs |
| Rules | Push to feature branch | Pipeline does not run |
| Merge request | Create MR | Merge-request pipeline runs |
| Parallelism | Pipeline graph | Test and security become eligible after build |
| Trivy | Inspect security job | Scan completes and JSON report is produced |
| Security artifact | Download artifact | `trivy-results.json` exists |
| Docker build | Inspect Docker job | Image builds successfully |
| Registry authentication | Inspect Docker logs | Login succeeds without password exposure |
| Docker push | Inspect Docker logs | Image layers are uploaded |
| Registry | GitLab Container Registry | Image repository and tag exist |
| Commit traceability | Compare SHA and image tag | Values match |
| Failure notification | Observe successful pipeline | Notification job is skipped |

---

# 4. Pipeline Execution Validation

## Objective

Verify that GitLab recognizes `.gitlab-ci.yml` and creates a valid pipeline.

The pipeline is defined in the repository root using:

```text
.gitlab-ci.yml
```

GitLab uses this file as the pipeline definition.

## Validation Method

Navigate to:

```text
GitLab Project
    ↓
Build
    ↓
Pipelines
```

Inspect the latest pipeline.

## Expected Result

The pipeline should appear with the expected stages:

```text
build
  ↓
test
  ↓
security
  ↓
docker
  ↓
notify
```

with test and security execution connected through the declared `needs` relationship.

## What This Proves

A valid pipeline proves that:

- `.gitlab-ci.yml` is syntactically acceptable to GitLab
- referenced stages exist
- jobs can be scheduled
- the runner can execute the jobs

GitLab's pipeline visualization provides visibility into job status and execution.

---

# 5. Build Job Validation

## Objective

Verify that the application can be built automatically inside GitLab CI.

The build job executes Maven:

```bash
mvn install
```

The build produces a WAR file under:

```text
target/
```

## Validation Method

Open:

```text
CI/CD
  → Pipelines
  → Latest Pipeline
  → build-job
```

Inspect the job log.

## Expected Result

The Maven build completes successfully.

The pipeline should show:

```text
build-job
    ✓ passed
```

## Artifact Validation

The build job is configured to preserve:

```text
target/*.war
```

as a GitLab CI artifact.

Download the build artifact from the job.

Expected:

```text
build artifact
     ↓
ZIP download
     ↓
target/
     ↓
*.war
```

## What This Proves

This validates two separate things:

1. Maven can build the application in the GitLab runner.
2. The generated WAR is successfully preserved beyond the ephemeral job environment.

---

# 6. Test Job Validation

## Objective

Verify that the pipeline performs application testing and Checkstyle validation.

The test job executes:

```bash
mvn test
mvn checkstyle:checkstyle
```

## Validation Method

Open:

```text
CI/CD
  → Pipelines
  → Latest Pipeline
  → test-job
```

Inspect the job log.

## Expected Result

Both commands complete successfully.

```text
mvn test
    ✓

mvn checkstyle:checkstyle
    ✓
```

The job should finish with:

```text
test-job
    ✓ passed
```

## What This Proves

This demonstrates that testing and code-quality validation are integrated into the pipeline rather than being manual local operations.

It does **not** prove that the application is production-ready or functionally correct under real production traffic.

---

# 7. `needs` Dependency Validation

## Objective

Verify that the build is the prerequisite for both testing and security scanning.

The intended dependency graph is:

```text
             build-job
                 │
          ┌──────┴──────┐
          ▼             ▼
      test-job     security-scan
```

Both jobs use:

```yaml
needs:
  - build-job
```

The section material explains that this allows both jobs to start once the build completes rather than waiting for one another.

## Validation Method

Inspect the GitLab pipeline graph.

## Expected Result

After `build-job` completes:

```text
build-job
    ✓
    │
    ├──────────► test-job
    │
    └──────────► security-scan
```

Testing and security should not be forced into a test → security sequence.

## What This Proves

This validates explicit dependency-based pipeline execution.

It demonstrates that:

```text
Dependency
    ≠
Simple stage ordering
```

---

# 8. GitLab Rules Validation

## Objective

Verify that pipeline execution is controlled by GitLab `rules`.

The rules iteration replaces the older `only` approach with `rules` and validates different pipeline sources.

The important scenarios are:

```text
main push
feature branch push
merge request
```

---

## 8.1 Main Branch Push

### Test

Push a commit to:

```text
main
```

### Expected Result

A pipeline is created.

```text
Push → main
      ↓
rules match
      ↓
Pipeline runs
```

### Evidence

Capture the pipeline graph showing the executed jobs.

---

## 8.2 Feature Branch Push

### Test

Create a feature branch and push a commit without creating a merge request.

Example:

```text
feature-91
```

### Expected Result

No pipeline should be triggered if the rules only permit the configured main/develop and merge-request/web/schedule cases.

### What This Proves

The rules behave as an execution whitelist:

```text
Allowed condition
       ↓
Execute

No matching condition
       ↓
Do not execute
```

---

# 9. Merge Request Validation

## Objective

Verify that a merge request triggers the intended validation pipeline.

The workflow is:

```text
main
 │
 ▼
Feature Branch
 │
 ▼
Change
 │
 ▼
Push
 │
 ▼
Merge Request
 │
 ▼
Pipeline
 │
 ▼
Validation
```

The merge-request event matches:

```text
CI_PIPELINE_SOURCE
        =
merge_request_event
```

## Validation Method

1. Create a feature branch.
2. Make a change.
3. Push the branch.
4. Create a merge request.
5. Open the pipeline view.

## Expected Result

```text
Merge Request
      ↓
merge request pipeline
      ↓
Build
      ↓
Test + Security
```

After successful validation, the merge request can be merged.

---

# 10. Security Scan Validation

## Objective

Verify that Trivy runs against the repository filesystem and generates a machine-readable vulnerability report.

The scan command is:

```bash
trivy fs \
  --format json \
  --exit-code 0 \
  --vuln-type os,library \
  --output trivy-results.json \
  .
```

The security job uses the Trivy container image and overrides its entrypoint so that the command can be invoked correctly.

## Validation Method

Open:

```text
CI/CD
  → Pipelines
  → security-scan
```

Inspect the job log.

## Expected Result

The Trivy command executes successfully and produces:

```text
trivy-results.json
```

## Important Interpretation

The demonstrated configuration uses:

```text
--exit-code 0
```

Therefore, the scan is informational/reporting-oriented.

It does **not** act as a blocking vulnerability quality gate.

A blocking configuration could use:

```text
--exit-code 1
```

and optionally severity filtering, but that is not the final demonstrated configuration.

---

# 11. Security Artifact Validation

## Objective

Verify that the Trivy report survives beyond the security job.

The security job declares:

```yaml
artifacts:
  paths:
    - trivy-results.json
  when: always
```

GitLab artifacts are specifically intended to preserve files produced inside ephemeral job environments.

## Validation Method

Open the completed pipeline and locate the security job artifact download.

Download the artifact.

Extract it.

## Expected Result

The extracted files contain:

```text
trivy-results.json
```

Open the JSON file.

It should contain structured Trivy scan results.

## What This Proves

This validates:

```text
Trivy
  ↓
JSON Report
  ↓
GitLab Artifact Storage
  ↓
Downloadable Evidence
```

The artifact is therefore not merely created inside the job; it is preserved as pipeline output.

---

# 12. Build Artifact Validation

The build job produces:

```text
target/*.war
```

and declares it as an artifact.

## Validation Method

Download the artifact associated with `build-job`.

Extract the ZIP.

## Expected Result

A WAR file exists under the expected target path.

Example:

```text
target/
└── vprofile-v2.war
```

## What This Proves

The artifact chain is:

```text
Maven
  ↓
WAR
  ↓
GitLab Artifact
  ↓
Download
```

This provides tangible evidence of the build result.

---

# 13. Docker Build Validation

## Objective

Verify that the validated repository source can be transformed into a Docker image inside GitLab CI.

The Docker job uses:

```yaml
image: docker:latest
services:
  - docker:dind
```

and depends on the security stage.

The Docker stage is declared after security.

## Validation Method

Open:

```text
CI/CD
  → Pipelines
  → docker-build-publish
```

Inspect the job log.

## Expected Sequence

The expected log progression is:

```text
Docker executor
      ↓
docker:latest
      ↓
docker:dind
      ↓
Wait for daemon
      ↓
Registry login
      ↓
docker build
      ↓
docker push
```

## Expected Result

The Docker job completes successfully.

```text
docker-build-publish
        ✓ passed
```

---

# 14. Docker Build Context Validation

The Docker build uses:

```bash
docker build \
  -f Docker-files/app/multistage/Dockerfile \
  -t $CI_REGISTRY_IMAGE:$IMAGE_TAG \
  .
```

The final:

```text
.
```

represents the repository root as the build context.

The Dockerfile was adapted to use the already checked-out source:

```dockerfile
COPY . /app
WORKDIR /app
RUN mvn install
```

instead of cloning the source again.

## What This Proves

The Docker image is being built from the same repository workspace handled by GitLab CI.

This maintains:

```text
Pipeline Source Revision
          ↓
Docker Build Context
          ↓
Docker Image
```

---

# 15. Docker Registry Authentication Validation

## Objective

Verify that the Docker job can authenticate to the GitLab Container Registry.

The login command is:

```bash
echo "$CI_REGISTRY_PASSWORD" |
docker login \
  -u "$CI_REGISTRY_USER" \
  --password-stdin \
  $CI_REGISTRY
```

The password is passed through standard input rather than as a visible command-line argument.

## Validation Method

Inspect the Docker job logs.

## Expected Result

The login completes without an authentication error.

The password should **not** appear in the log.

## What This Proves

It validates:

```text
GitLab predefined credentials
          ↓
docker login
          ↓
Authenticated Docker client
```

---

# 16. Docker Image Push Validation

## Objective

Verify that the image produced by the Docker build is successfully pushed to GitLab's Container Registry.

The push command is:

```bash
docker push $CI_REGISTRY_IMAGE:$IMAGE_TAG
```

The image layers should be uploaded to the registry.

## Validation Method

Inspect the final portion of the Docker job log.

## Expected Result

The push completes successfully.

The job should finish:

```text
docker-build-publish
        ✓ passed
```

The registry should contain the corresponding image.

---

# 17. Container Registry Validation

## Objective

Verify that the image exists outside the pipeline execution environment.

Navigate to:

```text
Deploy
  ↓
Container Registry
```

## Validation Method

1. Open the project's Container Registry.
2. Locate the image repository.
3. Open the image repository.
4. Inspect the available tags.

## Expected Result

An image repository corresponding to the project exists.

The image should have a tag based on the Git commit SHA.

Expected conceptual structure:

```text
GitLab Container Registry
        │
        └── project image
              │
              └── <commit-sha>
```

---

# 18. Commit-SHA Traceability Validation

## Objective

Verify that the published Docker image can be traced directly back to the Git commit that produced it.

The pipeline defines:

```text
IMAGE_TAG = CI_COMMIT_SHA
```

The resulting image reference is:

```text
$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

## Validation Method

Record:

```text
GitLab commit SHA
```

Then inspect the Container Registry image tag.

Compare:

```text
Commit SHA
     =
Image Tag
```

## Expected Result

The values match.

Conceptually:

```text
Git Commit
    │
    ▼
CI_COMMIT_SHA
    │
    ▼
IMAGE_TAG
    │
    ▼
Docker Image
    │
    ▼
Container Registry
```

## What This Proves

This provides source-to-image traceability.

Given the image tag, the source revision that produced it can be identified.

This is one of the strongest pieces of evidence in the project.

---

# 19. Failure Notification Validation

## Objective

Verify the intended behavior of the `notify-on-failure` job.

The job uses:

```yaml
when: on_failure
```

The intended behavior is:

```text
All previous jobs succeed
          ↓
notify-on-failure
          ↓
SKIPPED
```

If an earlier job fails:

```text
Previous job fails
          ↓
notify-on-failure
          ↓
RUNS
```

## Validation Method

Inspect the completed successful pipeline.

## Expected Result

The notification job appears as:

```text
skipped
```

## Important Boundary

The current implementation uses:

```bash
echo "Build or Test job failed..."
```

as the notification action.

Therefore, this validates the **failure-trigger mechanism**, not a real Slack, email, or external notification integration.

---

# 20. End-to-End Validation

The complete successful pipeline should demonstrate:

```text
                    Source Change
                         │
                         ▼
                       Rules
                         │
                         ▼
                     build-job
                         │
                 ┌───────┴───────┐
                 ▼               ▼
             test-job       security-scan
                 │               │
                 │               └──► trivy-results.json
                 │
                 └────────┬──────────┐
                          ▼
                  docker-build-publish
                          │
                     docker build
                          │
                      docker push
                          │
                          ▼
               GitLab Container Registry
                          │
                          ▼
                    Commit-SHA Image
```

The expected final state is:

```text
build-job              ✓
test-job               ✓
security-scan          ✓
docker-build-publish   ✓
notify-on-failure      skipped
```

Alongside:

```text
WAR artifact           ✓
Trivy JSON artifact    ✓
Container image        ✓
Registry tag           ✓
Commit SHA match       ✓
```

---

# 21. Evidence Mapping

The repository contains an `evidence/screenshots/` directory intended for high-signal execution evidence.

Expected evidence mapping:

| Evidence File | What It Should Demonstrate |
|---|---|
| `pipeline-success.png` | Successful end-to-end GitLab pipeline graph |
| `security-artifact.png` | Downloadable/visible Trivy security artifact |
| `docker-publish.png` | Successful Docker build and push job |
| `container-registry.png` | Published image and commit-SHA tag in GitLab Container Registry |

These files should contain **actual execution evidence from the completed environment**, not screenshots copied from course material.

---

# 22. Recommended Evidence Capture

## Evidence 1 — Successful Pipeline

Capture the GitLab pipeline graph showing the completed jobs.

Preferred information:

```text
build
test
security
docker
notify
```

with successful jobs clearly visible.

This demonstrates the overall pipeline execution.

---

## Evidence 2 — Security Artifact

Capture the security job or artifact interface showing:

```text
trivy-results.json
```

This demonstrates that the security scan produced a persistent artifact.

---

## Evidence 3 — Docker Publish

Capture the Docker job showing the progression:

```text
docker login
     ↓
docker build
     ↓
docker push
```

Do not capture credentials or sensitive information.

The password must remain hidden.

---

## Evidence 4 — Container Registry

Capture the GitLab Container Registry showing:

```text
project image
    ↓
commit-SHA tag
```

This is the strongest evidence that the image left the ephemeral runner environment and was successfully published.

---

# 23. Validation Boundaries

A successful pipeline should not be interpreted as proof of every aspect of the application or production system.

The current validation does **not** prove:

- production application behavior
- Kubernetes deployment
- runtime health
- production traffic handling
- high availability
- autoscaling
- disaster recovery
- rollback behavior
- production observability
- infrastructure provisioning
- deployment to a runtime platform
- an enforced Trivy vulnerability gate

The project ends at:

```text
Docker Image
      ↓
GitLab Container Registry
```

The Docker practical describes the image as ready for a later deployment target but does not implement that deployment in this project.

---

# 24. Security Validation Boundary

The security implementation must also be interpreted correctly.

The pipeline proves:

```text
Trivy executes
       ↓
Repository is scanned
       ↓
JSON report is generated
       ↓
Report is preserved
```

It does **not** prove:

```text
Vulnerability found
       ↓
Pipeline blocked
```

because the demonstrated configuration uses:

```text
--exit-code 0
```

The material presents `--exit-code 1` as the mechanism for converting vulnerabilities into a failed job, but that is an optional quality-gate configuration rather than the demonstrated final state.

---

# 25. Validation Summary

The project is considered technically validated when the following chain is demonstrated:

```text
                GitLab Pipeline
                       │
                       ▼
                    Build ✓
                       │
              ┌────────┴────────┐
              ▼                 ▼
           Test ✓          Security ✓
                                │
                                ▼
                         Trivy Artifact ✓
              └────────┬────────┘
                       ▼
                  Docker Build ✓
                       │
                       ▼
                   Docker Push ✓
                       │
                       ▼
              Registry Image ✓
                       │
                       ▼
             Commit SHA Match ✓
```

The corresponding execution evidence should show:

```text
✓ Successful pipeline
✓ WAR artifact
✓ Trivy JSON artifact
✓ Docker build/push
✓ Container Registry image
✓ Commit-SHA traceability
```

The final validation principle is:

> **Do not validate only that the pipeline is green. Validate that every important output exists, every major transition succeeded, and the published container can be traced back to the source revision that produced it.**

[← Back to README](../README.md) | [← Implementation](implementation.md) | [Architecture →](architecture.md) | [Limitations & Future Work →](limitations-and-future-work.md)
