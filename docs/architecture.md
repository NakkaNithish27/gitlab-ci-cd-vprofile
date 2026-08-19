# Architecture

[← Back to README](../README.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/772cf543-422a-4b9d-8298-7f66b00c677b" />


## 1. Architecture Overview

This project implements a GitLab CI/CD workflow around an existing VProfile Java application workload.

The architecture separates:

```text
Application Workload
        ≠
CI/CD Engineering
```

The VProfile application is the workload used by the pipeline. The engineering focus of this project is the automated delivery workflow surrounding that workload rather than development of the application itself.

At a high level:

```text
                 ┌──────────────────────┐
                 │   GitLab Repository  │
                 │  VProfile Workload   │
                 └──────────┬───────────┘
                            │
                            ▼
                     Pipeline Trigger
                            │
                            ▼
                     ┌─────────────┐
                     │    Build    │
                     └──────┬──────┘
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
          ┌─────────────┐       ┌──────────────┐
          │    Test     │       │   Security   │
          │ Maven +     │       │    Trivy     │
          │ Checkstyle  │       │ Filesystem   │
          └──────┬──────┘       └──────┬───────┘
                 │                     │
                 └──────────┬──────────┘
                            ▼
                    ┌─────────────────┐
                    │ Docker Build &  │
                    │     Publish     │
                    └────────┬────────┘
                             │
                             ▼
                  GitLab Container Registry
```

The final pipeline transforms source code into a versioned container image stored in GitLab's Container Registry.

---

## 2. Application Ownership Boundary

The architecture must distinguish the application from the CI/CD engineering performed around it.

```text
┌───────────────────────────────────────────┐
│        Existing VProfile Application      │
│                                           │
│  Application source / business logic      │
│  supplied as the workload                 │
└────────────────────┬──────────────────────┘
                     │
                     │ CI/CD operates around
                     │ this workload
                     ▼
┌───────────────────────────────────────────┐
│          GitLab CI/CD Engineering         │
│                                           │
│  Build → Test → Security → Docker → Push  │
└───────────────────────────────────────────┘
```

The VProfile source originated from the existing `hkhcoder/vprofile-project` repository and was transferred into the GitLab repository for the practical.

Therefore, this project does not claim ownership of the application's original business logic or implementation.

The engineering boundary is:

- source integration into GitLab
- CI/CD pipeline configuration
- automated build and testing
- variables and secrets
- execution rules
- security scanning
- artifact persistence
- Docker image construction
- container registry publication
- pipeline validation

---

## 3. GitLab Platform and Runner Architecture

GitLab separates pipeline orchestration from job execution.

Conceptually:

```text
                 GITLAB PLATFORM
                      │
       ┌──────────────┼───────────────┐
       │              │               │
       ▼              ▼               ▼
 Pipeline         Job State       Logs / Artifacts
 Definition       Tracking        Visualization
       │
       │ sends job instructions
       ▼
                    RUNNER
                      │
                      ▼
                Job Execution
```

GitLab acts as the controller while a runner performs the actual job commands.

The project uses Docker-based job execution. Each job specifies an appropriate container image, allowing the required tooling to be provided by the job environment.

---

## 4. Pipeline Architecture

The pipeline began as two stages:

```text
Build
  ↓
Test
```

The build job runs Maven to produce the application artifact, while the test job executes Maven tests and Checkstyle.

The pipeline was subsequently extended with security scanning, artifacts, notification behavior, and Docker image publishing.

The final stage structure is:

```yaml
stages:
  - build
  - test
  - security
  - docker
  - notify
```

The Docker stage is positioned after security, while the notification job provides failure-driven behavior.

Conceptually:

```text
┌─────────┐
│  Build  │
└────┬────┘
     │
     ├──────────────┐
     │              │
     ▼              ▼
┌─────────┐   ┌───────────┐
│  Test   │   │ Security  │
└────┬────┘   └─────┬─────┘
     │              │
     └──────┬───────┘
            ▼
     ┌──────────────┐
     │ Docker Build │
     │  & Publish   │
     └──────┬───────┘
            │
            ▼
    Container Registry
```

---

## 5. Job Dependency Architecture

GitLab stages provide the broad pipeline ordering, while `needs` establishes explicit job dependencies.

The build job is the common prerequisite:

```text
                 ┌──────────────┐
                 │    Build     │
                 └──────┬───────┘
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
       ┌───────────┐        ┌──────────────┐
       │   Test    │        │   Security   │
       │           │        │     Scan     │
       └───────────┘        └──────────────┘
```

Both testing and security scanning declare a dependency on the build job:

```yaml
needs: [build-job]
```

Because neither depends on the other, GitLab can start them as soon as the build completes rather than unnecessarily serializing them.

This produces a fan-out pattern:

```text
             Build
               │
        ┌──────┴──────┐
        ▼             ▼
      Test        Security
```

The Docker publishing job then forms the downstream transition after the validation stage.

---

## 6. Build Architecture

The build boundary is:

```text
VProfile Source
      │
      ▼
 Maven Build
      │
      ▼
target/*.war
```

The build job executes Maven inside the GitLab CI job environment.

The initial pipeline uses:

```yaml
script:
  - mvn install
```

The resulting WAR file becomes a pipeline artifact so it can survive beyond the lifetime of the build container.

The important architectural boundary is:

```text
Ephemeral Build Environment
          ≠
Persisted Build Artifact
```

---

## 7. Testing Architecture

Testing is implemented as a separate validation job.

Conceptually:

```text
Build
  │
  ▼
Test Job
  │
  ├── mvn test
  └── mvn checkstyle:checkstyle
```

This separates application compilation from application validation.

A failed command returns a non-zero status and causes the corresponding job to fail.

The project therefore treats:

```text
Build Success
     ≠
Validation Success
```

as separate pipeline concerns.

---

## 8. Pipeline Rules and Trigger Architecture

Not every source event should necessarily produce the same pipeline behavior.

The project uses GitLab `rules` to control when pipeline jobs execute.

The rules work with pipeline context such as:

- branch
- pipeline source
- merge request events
- manual/web triggers
- scheduled execution

The practical demonstrates behavior for:

```text
main branch push
       ↓
   pipeline

feature branch push
       ↓
no pipeline

merge request
       ↓
   pipeline
```

The rules therefore create an execution-control boundary between a source event and actual pipeline execution.

Conceptually:

```text
                    Source Event
                         │
                         ▼
                       Rules
                    /    |    \
                   /     |     \
               Push    MR     Other
                │       │
                ▼       ▼
             Execute  Execute
```

The rules are part of the delivery architecture rather than merely YAML syntax.

---

## 9. Variables and Runtime Configuration

Variables provide the mechanism through which configuration, secrets, and runtime context enter pipeline jobs.

The project uses multiple categories:

```text
                 GitLab CI/CD Variables
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   Pipeline-defined   UI-defined    Predefined
      variables       variables      variables
          │              │              │
       config         secrets        runtime
```

Pipeline-defined variables are appropriate for non-sensitive configuration.

UI-defined variables provide additional controls such as visibility, masking, protection, and environment scope.

GitLab also provides predefined variables automatically, including registry-related variables used by the Docker publishing stage.

The architecture therefore separates:

```text
Repository Configuration
        ≠
Secret Configuration
        ≠
GitLab Runtime Context
```

---

## 10. Security Scanning Architecture

The security stage introduces Trivy into the CI workflow.

The scan operates against the repository filesystem:

```text
Repository Workspace
        │
        ▼
      Trivy
        │
        ▼
trivy-results.json
```

The demonstrated command scans both operating-system and library vulnerability types and writes the result as JSON:

```bash
trivy fs --format json --exit-code 0 \
  --vuln-type os,library \
  --output trivy-results.json .
```

The result is preserved as a GitLab CI artifact.

The architecture therefore introduces security assessment before Docker image publication.

```text
Build
  ↓
Security Scan
  ↓
Docker Build
  ↓
Registry
```

### Security boundary

The demonstrated implementation should be understood accurately:

```text
Trivy Scan
    ↓
Vulnerability Report
    ↓
Artifact
```

The configured `--exit-code 0` means the scan produces a report without acting as a blocking vulnerability gate in the final demonstrated configuration.

Therefore:

> **Security scanning is implemented; an enforced vulnerability quality gate is not.**

---

## 11. Artifact Architecture

GitLab CI job containers are ephemeral.

Files created inside them would otherwise disappear when the job finishes.

Artifacts provide persistence:

```text
                 Job Container
                      │
             ┌────────┴────────┐
             ▼                 ▼
        target/*.war    trivy-results.json
             │                 │
             └────────┬────────┘
                      ▼
               GitLab Artifacts
```

The project preserves two important outputs:

### Build artifact

```text
target/*.war
```

### Security artifact

```text
trivy-results.json
```

Both are configured to be retained as job artifacts, including when the associated job finishes unsuccessfully.

This establishes:

```text
Job Workspace
     ≠
Longer-Lived Pipeline Evidence
```

---

## 12. Docker Build Architecture

The Docker stage transforms the validated workload into a deployable container image.

```text
Validated Repository
        │
        ▼
   Docker Build
        │
        ▼
   Docker Image
```

The Docker job uses:

```yaml
image: docker:latest
services:
  - docker:dind
```

This creates two containers for the Docker job:

```text
┌──────────────────────────────────────────────┐
│                Docker Job                    │
│                                              │
│  Main Container          Sidecar Container   │
│  ┌─────────────────┐    ┌─────────────────┐ │
│  │ docker:latest   │───►│ docker:dind     │ │
│  │                 │    │                 │ │
│  │ Docker CLI      │    │ Docker daemon   │ │
│  └─────────────────┘    └─────────────────┘ │
└──────────────────────────────────────────────┘
```

The main container provides the Docker CLI.

The `docker:dind` sidecar provides the Docker daemon that performs the actual image build and push operations.

This is an important architectural distinction:

```text
Docker CLI
    ≠
Docker daemon
```

The CLI sends commands to the daemon, while the daemon performs the Docker operations.

---

## 13. Docker-in-Docker Communication

The demonstrated configuration sets:

```yaml
DOCKER_TLS_CERTDIR: ""
```

This disables the TLS certificate setup between the Docker CLI and the `dind` service for this ephemeral job environment.

The practical therefore uses:

```text
docker:latest
     │
     │ Docker commands
     ▼
docker:dind
     │
     ├── Pull base images
     ├── Execute Dockerfile
     ├── Build layers
     └── Push image
```

The Docker service starts before the job's Docker commands execute, and GitLab waits for the service to become available.

---

## 14. Dockerfile and Build Context Architecture

The Dockerfile is located at:

```text
Docker-files/app/multistage/Dockerfile
```

The Docker build command uses:

```bash
docker build \
  -f Docker-files/app/multistage/Dockerfile \
  -t $CI_REGISTRY_IMAGE:$IMAGE_TAG \
  .
```

There are two distinct concepts here:

```text
Dockerfile location
        ≠
Build context
```

The `-f` option identifies the Dockerfile.

The final `.` identifies the repository root as the build context.

This allows the Dockerfile to use the source code already checked out by GitLab rather than cloning the repository again.

The CI-adapted build flow is:

```text
GitLab Runner checks out repository
              │
              ▼
          Build Context
              │
              ▼
         COPY . /app
              │
              ▼
         WORKDIR /app
              │
              ▼
         RUN mvn install
```

The practical specifically removes the original Git-clone approach because the source is already present in the CI workspace.

---

## 15. Multi-Stage Docker Architecture

The Dockerfile uses a multi-stage build.

Conceptually:

```text
Stage 1 — Build
────────────────────────
Source
  ↓
COPY . /app
  ↓
WORKDIR /app
  ↓
mvn install
  ↓
/app/target/vprofile-v2.war
          │
          ▼
Stage 2 — Runtime
────────────────────────
Tomcat
  +
WAR
  ↓
Final Docker Image
```

The first stage contains the Maven build environment.

The second stage copies the generated WAR into the Tomcat runtime image.

This separates the build environment from the final runtime image.

---

## 16. Container Registry Architecture

The GitLab Container Registry provides the image storage boundary.

```text
Docker Build
     │
     │ docker push
     ▼
GitLab Container Registry
     │
     ▼
Versioned Container Image
```

GitLab provides predefined CI/CD variables for registry interaction:

```text
$CI_REGISTRY
$CI_REGISTRY_USER
$CI_REGISTRY_PASSWORD
$CI_REGISTRY_IMAGE
```

The full image repository path is supplied through `$CI_REGISTRY_IMAGE`.

The registry therefore acts as:

```text
Image Build
    ↓
Image Storage
```

It is not the application runtime.

The demonstrated project stops at the registry; it does not implement deployment of the image to Kubernetes or another runtime platform.

---

## 17. Registry Authentication Architecture

The Docker job authenticates using:

```bash
echo "$CI_REGISTRY_PASSWORD" |
docker login \
  -u "$CI_REGISTRY_USER" \
  --password-stdin \
  $CI_REGISTRY
```

The authentication flow is:

```text
CI_REGISTRY_PASSWORD
        │
        │ stdin
        ▼
docker login
   │       │
   │       └── CI_REGISTRY_USER
   │
   └────────── CI_REGISTRY
        │
        ▼
Docker credentials
        │
        ▼
docker push
```

Using `--password-stdin` avoids passing the password as a visible command-line argument.

---

## 18. Image Tagging and Traceability

The Docker image uses the Git commit SHA as its tag:

```yaml
IMAGE_TAG: $CI_COMMIT_SHA
```

The resulting image reference follows the pattern:

```text
$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

Conceptually:

```text
Git Commit
    │
    │ SHA
    ▼
Docker Image Tag
    │
    ▼
Registry Image
```

This creates a source-to-image traceability relationship.

Given an image tag, the corresponding Git commit can be identified.

---

## 19. End-to-End Artifact Flow

The complete project can be understood as a sequence of transformations:

```text
┌──────────────────────┐
│ VProfile Source Code │
└──────────┬───────────┘
           │
           ▼
      Maven Build
           │
           ▼
     WAR Application
           │
           ├──────────────────► GitLab Artifact
           │
           ▼
     Test + Checkstyle
           │
           ▼
      Trivy Scan
           │
           └──────────────────► JSON Artifact
           │
           ▼
     Docker Build
           │
           ▼
      Docker Image
           │
           │ commit SHA tag
           ▼
GitLab Container Registry
```

This is the central engineering flow of the project.

---

## 20. Failure Notification Architecture

The pipeline contains a notification job configured with:

```yaml
when: on_failure
```

The notification job is therefore reactive:

```text
Previous Job
     │
     ├── Success → notification skipped
     │
     └── Failure → notification executes
```

When the complete pipeline succeeds, the notification job is expected to be skipped.

This demonstrates conditional pipeline behavior without requiring complex custom logic.

---

## 21. Architectural Decisions

### 21.1 Build Before Validation

The build job is the prerequisite for testing and security scanning.

```text
Build
  ├──► Test
  └──► Security
```

This reflects the dependency that validation operates on the resulting repository/build context.

### 21.2 Parallel Independent Validation

Testing and security scanning do not depend on one another.

Using `needs` allows them to execute in parallel after the build completes.

### 21.3 Security Before Docker Publication

The Docker stage follows the security stage in the declared pipeline structure.

Conceptually:

```text
Build
  ↓
Validation
  ↓
Docker Image
  ↓
Registry
```

This establishes security scanning as part of the pre-publication workflow.

### 21.4 Artifact Persistence

The WAR and Trivy report are explicitly preserved because job execution environments are ephemeral.

### 21.5 Commit-SHA Image Tags

Using the commit SHA makes the image traceable to a specific source revision rather than relying only on a mutable tag such as `latest`.

### 21.6 CI-Aware Dockerfile

The Dockerfile was adapted to use the source already checked out by GitLab rather than performing a second external clone.

This keeps the Docker build aligned with the source revision that the pipeline is validating.

### 21.7 GitLab Container Registry

The project uses GitLab's integrated registry rather than introducing a separate external image registry. This keeps the demonstrated CI workflow within the GitLab ecosystem.

---

## 22. Architectural Boundaries

This architecture intentionally ends at:

```text
Source
  ↓
Build
  ↓
Test
  ↓
Security Scan
  ↓
Docker Image
  ↓
GitLab Container Registry
```

It does **not** extend into:

```text
Container Registry
        ↓
Kubernetes
        ↓
Running Application
```

The project therefore does not establish:

- Kubernetes deployment
- GitOps
- Terraform-based infrastructure provisioning
- Automated application deployment
- Automated rollback
- Zero-downtime deployment
- Production runtime orchestration
- Production observability
- Application development ownership
- A blocking Trivy vulnerability quality gate

---

## 23. Future Architectural Evolution

The current architecture provides a natural foundation for future iterations.

### Current

```text
GitLab Repository
       ↓
Rules / Trigger
       ↓
Build
       ↓
Test + Security Scan
       ↓
Docker Build
       ↓
GitLab Container Registry
```

### Possible future evolution

```text
GitLab Repository
       ↓
Rules / Trigger
       ↓
Build
       ↓
Test
       ↓
Security Policy Gate
       ↓
Container Image Scan
       ↓
Image Registry
       ↓
Deployment Platform
       ↓
Runtime Validation
       ↓
Observability
```

Potential improvements include:

- converting the informational Trivy scan into an enforced vulnerability gate
- adding container-image vulnerability scanning
- introducing a deployment stage
- deploying the published image to a container runtime
- adding runtime validation
- introducing deployment rollback mechanisms
- introducing infrastructure-as-code in a future project iteration

These capabilities are **future work**, not completed components of this project.

---

## 24. Architecture Summary

The project can be compressed into one architectural model:

```text
                       GITLAB
                         │
                 Repository + CI/CD
                         │
                         ▼
                       Rules
                         │
                         ▼
                       Build
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
            Test              Security Scan
              │                     │
              └──────────┬──────────┘
                         ▼
                   Docker Build
                         │
                  Docker-in-Docker
                         │
                         ▼
                  Docker Image
                         │
                  Commit SHA Tag
                         │
                         ▼
              GitLab Container Registry
```

The core architectural principle is:

> **Validate the source, preserve meaningful outputs, package the validated workload into a traceable container image, and publish that image to an integrated registry.**

[← Back to README](../README.md)
