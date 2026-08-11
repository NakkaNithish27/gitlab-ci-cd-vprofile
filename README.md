# GitLab CI/CD Pipeline — VProfile Workload

A GitLab CI/CD workflow that automates application build and validation, integrates vulnerability scanning, preserves pipeline artifacts, and builds and publishes a Docker image to the GitLab Container Registry.

## Overview

This project demonstrates an end-to-end **GitLab CI/CD workflow** built around the **VProfile Java application workload**.

The pipeline evolved from a basic build-and-test workflow into a controlled delivery pipeline:

```text
Source Change
     │
     ▼
GitLab CI/CD
     │
     ▼
   Build
     │
     ├───────────────┐
     ▼               ▼
   Test        Security Scan
     │               │
     └───────┬───────┘
             ▼
       Docker Build
             │
             ▼
   GitLab Container Registry
```

The project focuses on the **CI/CD engineering around the application workload**, rather than application development.

## Application Ownership Boundary

The VProfile application was used as the workload for this practical.

I did **not** develop the original VProfile application or its business logic.

My engineering work focuses on the delivery workflow around the existing workload, including:

- GitLab CI/CD pipeline configuration
- Build and test automation
- Pipeline variables and secrets
- Branch and event-based execution rules
- Merge-request validation workflow
- Trivy filesystem vulnerability scanning
- Artifact preservation
- Job dependency and parallel execution
- Docker-in-Docker configuration
- Docker image construction
- Commit-SHA image tagging
- GitLab Container Registry publication
- Pipeline validation

This distinction is intentional:

```text
Existing Application Workload
          ≠
CI/CD Engineering Around the Workload
```

## Engineering Objective

The objective was to create a controlled delivery workflow that moves the existing application through:

```text
Source Change
     ↓
Build
     ↓
Testing
     ↓
Security Scan
     ↓
Docker Build
     ↓
Container Registry
```

The workflow also uses GitLab rules to control execution based on branch and pipeline source.

## Architecture

The pipeline uses a dependency model in which the build completes before the independent testing and security jobs execute:

```text
                 ┌───────────────┐
                 │    Testing    │
                 └───────┬───────┘
                         │
                         │
┌─────────┐              ▼
│  Build  │───────► Docker Build & Publish
└────┬────┘              ▲
     │                   │
     └──────► Security ──┘
               Scan
```

Testing and security scanning both depend on the build job.

The Docker publishing stage occurs after the validation work.

For the detailed architecture:

**[Architecture →](docs/architecture.md)**

## My Engineering Contribution

### CI/CD Pipeline

- Created and evolved the GitLab pipeline configuration.
- Configured build and test jobs using Docker-based execution.
- Automated Maven build, test, and Checkstyle execution.
- Configured pipeline variables and GitLab-managed variables.
- Demonstrated protected, masked, and hidden secret handling.

### Pipeline Control

- Replaced the basic branch restriction with GitLab `rules`.
- Configured execution for relevant branch and pipeline events.
- Validated merge-request-driven pipeline execution.
- Used `needs` to allow testing and security scanning to run independently after the build.

### Security

- Integrated Trivy filesystem vulnerability scanning.
- Generated a machine-readable JSON vulnerability report.
- Preserved the scan report as a pipeline artifact.
- Configured failure notification behavior.

### Artifacts

- Preserved the generated WAR file as a GitLab CI artifact.
- Preserved the Trivy JSON report as a GitLab CI artifact.

### Docker

- Configured Docker-in-Docker for image construction inside GitLab CI.
- Authenticated to the GitLab Container Registry using GitLab-provided CI variables.
- Adapted the Docker build process to use the repository workspace as its build context.
- Used a multi-stage Docker build.
- Tagged the Docker image using the Git commit SHA.
- Published the resulting image to the GitLab Container Registry.

For implementation details:

**[Implementation →](docs/implementation.md)**

## Key Engineering Decisions

### Separate Testing and Security Validation

Testing and security scanning represent different validation concerns.

Both depend on the build rather than depending on one another:

```text
Build
 ├──► Test
 └──► Security Scan
```

This avoids unnecessarily serializing independent validation work.

### Artifact Persistence

GitLab CI jobs execute in ephemeral environments.

The build output and security report are therefore explicitly declared as artifacts so they remain available after job execution.

### Tool-Specific Entrypoint Handling

The Trivy image uses Trivy as its Docker entrypoint.

The pipeline overrides that entrypoint so the CI script can invoke the Trivy command directly.

### Commit-SHA Image Tagging

The Docker image is tagged using the Git commit SHA.

This creates a direct relationship between the published image and the source revision that produced it.

### Container Registry Publication

The final pipeline stage converts the validated application workload into a Docker image and publishes that image to the GitLab Container Registry.

## Validation

The project validates the workflow through:

- Pipeline execution
- Build job execution
- Test and Checkstyle execution
- Rules behavior
- Trivy scan execution
- Artifact generation
- Docker image construction
- Registry publication
- Commit-SHA image tagging

The validation strategy and evidence mapping are documented here:

**[Validation →](docs/validation.md)**

> **Important:** The demonstrated Trivy configuration uses `--exit-code 0`. Therefore, the security scan produces a vulnerability report but does not act as a blocking vulnerability gate in the final demonstrated configuration.

## Project Boundaries

This project demonstrates **GitLab CI/CD and container image publishing around an existing application workload**.

It does not demonstrate:

- Development of the VProfile application
- VProfile business-logic implementation
- Application architecture ownership
- Infrastructure as Code
- Terraform-based provisioning
- Kubernetes deployment
- GitOps implementation
- Automated application deployment
- Automated rollback
- Zero-downtime deployment
- Production observability
- Enterprise-grade CI/CD platform engineering
- A blocking Trivy vulnerability gate

Future improvements and architectural evolution are documented here:

**[Limitations & Future Work →](docs/limitations-and-future-work.md)**

## Technologies

- GitLab
- Git
- GitLab CI/CD
- YAML
- Docker
- Docker-in-Docker
- Maven
- Java
- Trivy
- GitLab Container Registry

## Repository Navigation

| Document | Purpose |
|---|---|
| [Architecture](docs/architecture.md) | Pipeline architecture, dependencies, artifact flow, and major decisions |
| [Implementation](docs/implementation.md) | How the pipeline was configured and assembled |
| [Validation](docs/validation.md) | Validation strategy, results, and evidence mapping |
| [Limitations & Future Work](docs/limitations-and-future-work.md) | Project boundaries and logical next iterations |

## Evidence

Personal execution evidence should be added only when available and should demonstrate the completed environment rather than course material.

High-signal evidence includes:

- Successful pipeline graph
- Trivy artifact
- Docker build/push job
- GitLab Container Registry image
- Commit-SHA traceability
- Rules behavior where useful

[← Back to top](#gitlab-cicd-pipeline--vprofile-workload)
