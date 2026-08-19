# Implementation

[← Back to README](../README.md) | [Architecture →](architecture.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/d826f158-b86e-4aa5-8e03-d6c0d571a7d4" />


## 1. Implementation Overview

This document describes how the GitLab CI/CD workflow was assembled and evolved around the existing VProfile application workload.

The implementation progressed incrementally:

```text
Initial Repository Setup
        ↓
First Build/Test Pipeline
        ↓
Variables & Secrets
        ↓
Rules & Triggers
        ↓
Security Scan & Artifacts
        ↓
Docker Build & Publish
```

Each iteration extended the same GitLab repository and `.gitlab-ci.yml` rather than creating separate pipelines.

The resulting implementation provides:

```text
Build
  ↓
Test ───────────────┐
  │                 │
  └──── Security ───┤
                    ↓
              Docker Build
                    ↓
            Container Registry
```

---

## 2. Repository Setup

### 2.1 GitLab Repository

The practical began with a GitLab SaaS project and an empty repository.

The repository was organized under a GitLab group and project structure:

```text
GitLab
  └── Group
       └── Project
            └── Git Repository
```

The GitLab project provides the repository together with CI/CD and other project-level capabilities.

### 2.2 SSH Authentication

SSH authentication was configured so Git operations could use a dedicated GitLab identity.

The SSH configuration used an alias:

```text
Host gitlab.com-devops-with-gitlab
    HostName gitlab.com
    IdentityFile ~/.ssh/devops-with-gitlab
```

The public key was added to the GitLab user's SSH Keys settings.

The important relationship is:

```text
Git URL Alias
      ↓
~/.ssh/config
      ↓
IdentityFile
      ↓
Private Key
      ↓
GitLab Authentication
```

The alias was then used in the GitLab clone URL so that the correct SSH key was automatically selected for future `git push`, `git pull`, and related operations.

### 2.3 Repository Clone

The GitLab repository was cloned through SSH.

The practical verified the clone by checking:

```bash
ls -a
```

and confirming the `.git` directory existed.

The remote configuration was also inspected:

```bash
cat .git/config
```

This confirmed that the repository was connected to the intended GitLab remote.

### 2.4 VProfile Source Transfer

The VProfile source was originally available from the existing GitHub repository:

```text
hkhcoder/vprofile-project
```

The practical specifically used the `docker` branch.

The transfer process was:

```text
GitHub
  ↓
Download docker branch
  ↓
Extract source
  ↓
Copy into GitLab repository
  ↓
Commit
  ↓
Push
  ↓
GitLab Repository
```

This was a one-time source transfer rather than a GitHub/GitLab mirror or fork.

After the transfer, subsequent pipeline work was performed in the GitLab repository.

### Ownership Boundary

The original VProfile application source and business logic are not represented as personally developed application code.

The engineering work documented here concerns the CI/CD workflow around that existing workload.

---

## 3. Initial Pipeline Definition

### 3.1 Pipeline File

The pipeline definition was created at the repository root:

```text
.gitlab-ci.yml
```

GitLab recognizes this exact filename as the CI/CD pipeline definition.

The initial implementation contained two stages:

```yaml
stages:
  - build
  - test
```

Stage order defines the initial pipeline sequence:

```text
build
  ↓
test
```

### 3.2 Pipeline Variable

A top-level pipeline variable was introduced:

```yaml
variables:
  PROJECT_NAME: "Vprofile-App"
```

This provided a reusable non-sensitive value accessible from job scripts.

### 3.3 Build Job

The first job was configured around a Maven container:

```yaml
build-job:
  stage: build
  image: maven:3.9.9-eclipse-temurin-17
  only:
    - main
  script:
    - echo "Building project => $PROJECT_NAME"
    - mvn install
```

The job performs:

```text
GitLab Runner
     ↓
Maven Container
     ↓
Repository Workspace
     ↓
mvn install
     ↓
WAR Artifact
```

The Maven image provides the Maven and JDK tooling required by the job.

### 3.4 Test Job

Testing was implemented as a separate job:

```yaml
test-job:
  stage: test
  image: maven:3.9.9-eclipse-temurin-17
  only:
    - main
  needs:
    - build-job
  script:
    - echo "${PROJECT_NAME}-test"
    - mvn test
    - mvn checkstyle:checkstyle
```

The test job performs two Maven-related validation operations:

```text
mvn test
     +
mvn checkstyle:checkstyle
```

The job therefore separates build execution from validation.

### 3.5 Initial Execution

After saving `.gitlab-ci.yml`, the changes were committed and pushed.

A commit to the configured branch triggered the GitLab pipeline.

The pipeline was then inspected through:

```text
GitLab
  → Build
  → Pipelines
```

The job logs were used to observe:

- executor preparation
- Docker image pull
- repository checkout
- Maven execution
- job status
- pipeline status

The practical also notes that a first-time GitLab account may require account verification before pipelines can execute.

---

## 4. Pipeline Variables and Secrets

The next implementation iteration introduced three categories of variables:

```text
1. Pipeline-defined variables
2. UI-defined variables
3. GitLab predefined variables
```

These all become available within the job execution environment.

---

## 5. Pipeline-Defined Variables

Non-sensitive configuration was kept in `.gitlab-ci.yml`.

Example:

```yaml
variables:
  PROJECT_NAME: vprofile
  JAVA_OPTS: ${JAVA_OPTIONS}
```

`PROJECT_NAME` is directly defined in the repository.

`JAVA_OPTS` demonstrates how a YAML-defined variable can reference a UI-managed variable.

The implementation therefore creates:

```text
GitLab UI Variable
        ↓
JAVA_OPTIONS
        ↓
JAVA_OPTS
        ↓
Pipeline Job
```

Variable references use:

```text
$VARIABLE
```

or:

```text
${VARIABLE}
```

Curly braces are particularly useful when a variable is immediately followed by additional characters.

For example:

```bash
echo "${PROJECT_NAME}-test"
```

---

## 6. UI-Managed Variables

Project-level variables were configured through:

```text
Settings
  → CI/CD
  → Variables
```

The practical distinguishes normal configuration variables from secrets.

A normal variable can be visible.

A secret variable should use GitLab's visibility and protection controls.

The demonstrated secret configuration uses:

```text
Masked
Hidden
Protected
```

The implementation therefore keeps sensitive values outside the repository YAML.

Conceptually:

```text
.gitlab-ci.yml
     │
     ├── Non-sensitive configuration
     │
     └── References to secrets
                 │
                 ▼
        GitLab UI Variables
                 │
                 ▼
          Runtime Environment
```

The actual secret value should never be committed to the repository.

---

## 7. Predefined GitLab Variables

GitLab automatically injects predefined variables into pipeline jobs.

The practical demonstrates variables such as:

```text
CI_PIPELINE_SOURCE
CI_COMMIT_BRANCH
```

These provide runtime context.

For example:

```bash
echo "The source of the pipeline trigger is $CI_PIPELINE_SOURCE for branch $CI_COMMIT_BRANCH"
```

A push to `main` produces runtime context corresponding to:

```text
CI_PIPELINE_SOURCE = push
CI_COMMIT_BRANCH   = main
```

Later pipeline rules use this runtime information to make execution decisions.

---

## 8. Variable Validation

The implementation validated variables through job logs.

The practical checks:

```text
Pipeline variable
      ↓
Visible value in log

Normal UI variable
      ↓
Visible value in log

Masked/hidden secret
      ↓
[MASKED]

Predefined variable
      ↓
Runtime value
```

The important security behavior is that the secret's actual value is not displayed in the job log.

---

## 9. Rules and Trigger Implementation

The initial pipeline used:

```yaml
only:
  - main
```

The rules iteration replaced this older restriction with GitLab `rules`.

The purpose was to control job execution based on pipeline context.

### 9.1 Main Branch Rule

A branch-based condition was introduced for the main branch.

Conceptually:

```yaml
- if: '$CI_COMMIT_BRANCH == "main"'
```

This allows normal pushes to `main` to trigger the job.

### 9.2 Merge Request Rule

Merge-request execution was added using:

```yaml
- if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

This allows merge-request pipelines to perform validation before integration.

### 9.3 Manual/Web Trigger

The practical also demonstrates:

```yaml
- if: '$CI_PIPELINE_SOURCE == "web"'
```

This corresponds to manually starting a pipeline from the GitLab web interface.

### 9.4 Scheduled Trigger

Scheduled execution can be matched with:

```yaml
- if: '$CI_PIPELINE_SOURCE == "schedule"'
```

### 9.5 Catch-All

A defensive final rule can be used:

```yaml
- when: never
```

This prevents execution when none of the intended conditions match.

---

## 10. Rules Validation

The implementation validates three primary scenarios.

### Scenario 1 — Push to `main`

```text
Push to main
     ↓
Rules match
     ↓
Pipeline executes
```

### Scenario 2 — Push to Feature Branch

```text
Push to feature branch
     ↓
No matching rule
     ↓
Pipeline does not execute
```

### Scenario 3 — Merge Request

```text
Feature branch
     ↓
Merge Request
     ↓
merge_request_event
     ↓
Pipeline executes
```

This establishes the intended rule behavior as an explicit whitelist of permitted execution contexts.

---

## 11. Merge Request Workflow

The practical demonstrates the complete branch workflow:

```text
main
 │
 └── Create feature branch
          │
          ▼
      Make change
          │
          ▼
        Commit
          │
          ▼
     Push branch
          │
          ▼
  Create Merge Request
          │
          ▼
  Pipeline validation
          │
          ▼
        Merge
          │
          ▼
   Delete source branch
```

After the merge, local branch cleanup was demonstrated by:

```text
switch to main
     ↓
delete local branch
     ↓
prune stale references
     ↓
synchronize
```

This keeps local and remote branch state aligned.

---

## 12. Security Scan Implementation

The security iteration added a Trivy job.

The job was configured as:

```yaml
security-scan:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  needs: [build-job]
  script:
    - trivy fs --format json --exit-code 0 --vuln-type os,library --output trivy-results.json .
  after_script:
    - ls -la
  artifacts:
    paths:
      - trivy-results.json
    when: always
```

### 12.1 Trivy Image

The job uses:

```text
aquasec/trivy:latest
```

rather than installing Trivy manually inside the Maven image.

### 12.2 Entrypoint Override

The Trivy image has its own entrypoint.

Without overriding it, GitLab could effectively combine:

```text
trivy
```

with the script:

```text
trivy fs ...
```

resulting in an invalid invocation equivalent to:

```text
trivy trivy fs ...
```

The implementation therefore uses:

```yaml
entrypoint: [""]
```

This allows the script to invoke Trivy explicitly.

### 12.3 Filesystem Scan

The scan command is:

```bash
trivy fs \
  --format json \
  --exit-code 0 \
  --vuln-type os,library \
  --output trivy-results.json \
  .
```

The final `.` means the current repository workspace is scanned.

The result is written to:

```text
trivy-results.json
```

---

## 13. Security Quality-Gate Boundary

The demonstrated implementation uses:

```text
--exit-code 0
```

Therefore:

```text
Vulnerability Found
       ↓
Report Generated
       ↓
Trivy returns success
       ↓
Pipeline can continue
```

The material demonstrates the alternative:

```bash
--exit-code 1
```

which would cause Trivy to return a failure status when matching vulnerabilities are found.

However, the demonstrated final configuration uses `--exit-code 0`.

Therefore this project implements:

> **Security scanning and reporting**

rather than:

> **An enforced vulnerability quality gate**

---

## 14. Parallel Test and Security Execution

The security job depends on the build:

```yaml
needs: [build-job]
```

The test job was also updated:

```yaml
test-job:
  stage: test
  needs: [build-job]
```

This creates:

```text
             build-job
                 │
          ┌──────┴──────┐
          ▼             ▼
      test-job     security-scan
```

Both jobs become eligible after the build completes.

Without `needs`, the declared stage ordering would force the security stage to wait for the test stage.

With `needs`, GitLab can execute the independent validation jobs concurrently.

---

## 15. Stage Declaration

Every stage referenced by a job must be declared in the top-level `stages:` list.

The security iteration introduced:

```yaml
stages:
  - build
  - test
  - security
  - notify
```

The Docker iteration later extended this to:

```yaml
stages:
  - build
  - test
  - security
  - docker
  - notify
```

A stage mismatch results in an invalid pipeline configuration.

For example:

```yaml
stage: security
```

requires:

```yaml
- security
```

to exist in the top-level stage list.

---

## 16. Build Artifact Implementation

The build job was extended to preserve the generated WAR:

```yaml
build-job:
  stage: build
  script:
    - echo "Building the project..."
    - mvn install
  after_script:
    - ls -la
  artifacts:
    paths:
      - target/*.war
    when: always
```

The important artifact configuration is:

```yaml
artifacts:
  paths:
    - target/*.war
  when: always
```

The wildcard allows the generated WAR filename to be matched.

The `when: always` setting preserves the artifact even when the job fails.

---

## 17. Security Artifact Implementation

The Trivy job preserves:

```text
trivy-results.json
```

through:

```yaml
artifacts:
  paths:
    - trivy-results.json
  when: always
```

The two primary pipeline artifacts are therefore:

```text
build-job
    ↓
target/*.war

security-scan
    ↓
trivy-results.json
```

These artifacts can be downloaded from the GitLab pipeline interface.

---

## 18. Failure Notification Implementation

A notification job was added:

```yaml
notify-on-failure:
  stage: notify
  script:
    - echo "Build or Test job failed for $PROJECT_NAME! Please check logs."
  when: on_failure
```

This job demonstrates the failure-notification pattern.

Normal success:

```text
All previous jobs pass
        ↓
notify-on-failure
        ↓
SKIPPED
```

Failure:

```text
Previous job fails
        ↓
notify-on-failure
        ↓
RUNS
```

The demonstrated notification is an `echo` command rather than a real external notification integration.

---

## 19. Docker Build and Publish Implementation

The final implementation iteration added the Docker stage.

The stage list became:

```yaml
stages:
  - build
  - test
  - security
  - docker
  - notify
```

The Docker stage executes after the security stage.

---

## 20. Docker Build Job

The Docker job was defined as:

```yaml
docker-build-publish:
  stage: docker
  image: docker:latest
  services:
    - docker:dind
  needs: [security-scan]
  variables:
    DOCKER_TLS_CERTDIR: ""
    IMAGE_TAG: $CI_COMMIT_SHA
```

The implementation establishes two Docker containers:

```text
Main Container
docker:latest
      │
      │ Docker CLI
      ▼
Sidecar
docker:dind
      │
      │ Docker daemon
      ▼
Build / Push
```

The main container provides the Docker CLI.

The `docker:dind` service provides the Docker daemon.

---

## 21. Docker Registry Authentication

GitLab provides predefined registry variables:

```text
$CI_REGISTRY
$CI_REGISTRY_USER
$CI_REGISTRY_PASSWORD
$CI_REGISTRY_IMAGE
```

The implementation authenticates with:

```bash
echo "$CI_REGISTRY_PASSWORD" |
docker login \
  -u "$CI_REGISTRY_USER" \
  --password-stdin \
  $CI_REGISTRY
```

The password is supplied through standard input rather than as a command-line argument.

This avoids exposing the password directly in the command invocation.

---

## 22. Commit-SHA Image Tagging

The Docker job defines:

```yaml
IMAGE_TAG: $CI_COMMIT_SHA
```

The final image reference is therefore:

```text
$CI_REGISTRY_IMAGE:$IMAGE_TAG
```

which resolves conceptually to:

```text
<registry>/<group>/<project>:<commit-sha>
```

The implementation creates direct traceability:

```text
Git Commit
    ↓
CI_COMMIT_SHA
    ↓
Docker Image Tag
    ↓
Container Registry
```

This avoids relying on a mutable tag such as `latest` for the demonstrated image.

---

## 23. Docker Build Command

The image is built using:

```bash
docker build \
  -f Docker-files/app/multistage/Dockerfile \
  -t $CI_REGISTRY_IMAGE:$IMAGE_TAG \
  .
```

There are two important path concepts:

```text
-f Docker-files/app/multistage/Dockerfile
        ↓
Dockerfile location

.
        ↓
Build context
```

The repository root is the build context.

This allows the Dockerfile to access the source tree through `COPY`.

---

## 24. Dockerfile CI Adaptation

The existing Dockerfile was designed to operate independently and contained instructions for cloning the source from GitHub.

That behavior was unsuitable inside the GitLab pipeline because the GitLab runner already checks out the source into the job workspace.

The CI adaptation replaces the external clone approach with:

```dockerfile
COPY . /app
WORKDIR /app
RUN mvn install
```

The transformation is:

```text
Standalone Dockerfile
---------------------
git clone source
     ↓
external repository

CI Dockerfile
---------------------
COPY . /app
     ↓
pipeline workspace
```

This ensures that the Docker build uses the same source revision already checked out and validated by the pipeline.

---

## 25. Multi-Stage Docker Build

The Dockerfile uses a multi-stage structure.

### Build Stage

```text
Repository Context
       ↓
COPY . /app
       ↓
WORKDIR /app
       ↓
RUN mvn install
       ↓
/app/target/vprofile-v2.war
```

### Runtime Stage

The generated WAR is copied from the build stage:

```dockerfile
COPY --from=build /app/target/vprofile-v2.war <tomcat-webapps-path>
```

The final image therefore contains the Tomcat runtime and application artifact rather than the Maven build environment.

Conceptually:

```text
Stage 1
Maven + Source
      ↓
     WAR
      ↓
Stage 2
Tomcat + WAR
      ↓
Final Image
```

---

## 26. Docker Push

After the image is built, it is pushed:

```bash
docker push $CI_REGISTRY_IMAGE:$IMAGE_TAG
```

The complete Docker job sequence is:

```text
1. Start docker:latest
2. Start docker:dind
3. Wait for Docker daemon
4. Login to GitLab Container Registry
5. Build image
6. Tag with commit SHA
7. Push image
8. Image available in Container Registry
```

The expected registry location is:

```text
Deploy
  → Container Registry
```

---

## 27. Complete Pipeline Implementation

The implementation can be represented as:

```text
                         GitLab Repository
                                │
                                ▼
                          Pipeline Rules
                                │
                                ▼
                            build-job
                                │
                         mvn install
                                │
                         target/*.war
                                │
              ┌─────────────────┴─────────────────┐
              ▼                                   ▼
          test-job                          security-scan
              │                                   │
          mvn test                         Trivy filesystem
              │                                   │
       Checkstyle                           JSON report
              │                                   │
              └─────────────────┬─────────────────┘
                                ▼
                       docker-build-publish
                                │
                         Docker-in-Docker
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

Failure behavior branches separately:

```text
Any Previous Job Failure
          ↓
 notify-on-failure
          ↓
        echo
```

---

## 28. Git Workflow During Implementation

Each pipeline iteration followed the same basic repository workflow:

```text
Modify
  ↓
Save
  ↓
git add
  ↓
git commit
  ↓
git push
  ↓
GitLab Pipeline
  ↓
Inspect Results
```

For example, the security/artifact iteration used:

```bash
git add .gitlab-ci.yml
git commit -m "Add security scan stage and artifacts"
git push
```

The Docker iteration similarly required saving the pipeline and Dockerfile changes before committing and pushing.

This keeps pipeline configuration version-controlled alongside the application workload.

---

## 29. Implementation Troubleshooting

### 29.1 Invalid Stage

Symptom:

```text
job chosen stage security does not exist
```

Cause:

```text
stage: security
```

was used without:

```yaml
- security
```

in the top-level `stages:` list.

Fix:

```yaml
stages:
  - build
  - test
  - security
```

---

### 29.2 Trivy Entrypoint Collision

Symptom:

```text
Trivy usage / unknown command error
```

Cause:

```text
Trivy image entrypoint
        +
script explicitly calling trivy
        =
trivy trivy ...
```

Fix:

```yaml
image:
  name: aquasec/trivy:latest
  entrypoint: [""]
```

---

### 29.3 Pipeline Does Not Run on Feature Branch

If the pipeline still contains:

```yaml
only:
  - main
```

a push to a feature branch will not execute the job.

After migrating to `rules`, execution should instead depend on the explicitly configured conditions.

---

### 29.4 Protected Variable Missing

A protected variable may not be available in a pipeline running from an unprotected branch.

The implementation must therefore align:

```text
Variable protection
        ↕
Branch protection
```

---

### 29.5 Docker Build Context Problem

If the Dockerfile contains:

```dockerfile
COPY . /app
```

the build context must include the required source files.

The demonstrated command therefore uses:

```bash
docker build -f Docker-files/app/multistage/Dockerfile ... .
```

where the final `.` is the repository root.

---

### 29.6 Dockerfile External Clone

If the original Dockerfile still clones the GitHub repository, the Docker build can use a different source revision from the one the pipeline validated.

The CI adaptation removes the external clone and uses:

```dockerfile
COPY . /app
WORKDIR /app
RUN mvn install
```

---

### 29.7 First-Time Pipeline Verification

A new GitLab account may require account verification before pipelines can execute.

If the first pipeline fails specifically because account verification is required:

```text
Verify account
     ↓
Make a new commit
     ↓
Push
     ↓
New pipeline
```

The failed pipeline does not automatically rerun after verification.

---

## 30. Implementation Patterns

### Pattern 1 — Pipeline as Code

```text
.gitlab-ci.yml
      ↓
Version controlled pipeline definition
```

Pipeline configuration changes therefore travel through Git.

### Pattern 2 — Ephemeral Execution

```text
Job
 ↓
Container
 ↓
Execute
 ↓
Destroy
```

Important outputs must explicitly become artifacts or registry objects.

### Pattern 3 — Dependency-Based Parallelism

```text
Build
 ├── Test
 └── Security
```

`needs` expresses actual job dependencies.

### Pattern 4 — Managed Secrets

```text
Repository
   ↓
Reference variable
   ↓
GitLab UI secret
   ↓
Runtime injection
```

Secret values are not stored directly in `.gitlab-ci.yml`.

### Pattern 5 — Commit-Based Traceability

```text
Commit SHA
    ↓
Image Tag
    ↓
Container Registry
```

The published image can therefore be associated with the source revision that produced it.

### Pattern 6 — CI-Aware Docker Context

```text
Runner checkout
      ↓
Repository root
      ↓
Docker build context
      ↓
COPY . /app
```

The container build consumes the source already validated by CI.

### Pattern 7 — Scan → Report → Decision

```text
Source
  ↓
Trivy
  ↓
Report
  ↓
Quality Decision
```

The demonstrated project stops at reporting because `--exit-code 0` is used.

### Pattern 8 — Failure-Driven Notification

```text
Pipeline
   │
   ├── Success → notification skipped
   │
   └── Failure → notification job runs
```

---

## 31. Implementation Boundary

The implementation ends with:

```text
Validated Source
      ↓
Docker Image
      ↓
GitLab Container Registry
```

It does not implement:

```text
Registry
   ↓
Kubernetes
   ↓
Deployment
   ↓
Runtime Validation
   ↓
Rollback
```

Those are future pipeline extensions rather than completed implementation components.

---

## 32. Implementation Summary

The implementation evolved from a simple GitLab build/test pipeline into a multi-stage CI/CD workflow.

The final implementation can be summarized as:

```text
.gitlab-ci.yml
│
├── Pipeline Variables
│
├── Rules
│
├── build-job
│     └── mvn install
│
├── test-job
│     ├── mvn test
│     └── mvn checkstyle:checkstyle
│
├── security-scan
│     └── Trivy filesystem scan
│
├── Artifacts
│     ├── target/*.war
│     └── trivy-results.json
│
├── docker-build-publish
│     ├── docker:dind
│     ├── docker login
│     ├── docker build
│     └── docker push
│
└── notify-on-failure
      └── when: on_failure
```

The resulting implementation demonstrates the progression:

```text
Build
  ↓
Validate
  ↓
Scan
  ↓
Preserve Outputs
  ↓
Containerize
  ↓
Publish
```

The central implementation principle is:

> **Keep the pipeline definition version-controlled, make dependencies explicit, preserve important outputs, integrate security into the workflow, and produce a traceable container image from the same source revision validated by CI.**

[← Back to README](../README.md) | [Architecture →](architecture.md) | [Validation →](validation.md)
