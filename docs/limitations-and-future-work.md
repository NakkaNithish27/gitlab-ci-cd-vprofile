# Limitations and Future Work

[← Back to README](../README.md) | [← Architecture](architecture.md) | [← Implementation](implementation.md) | [← Validation](validation.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/67df0550-28ea-470a-92f4-1bd3a297c39e" />


## 1. Purpose

This document defines the boundaries of the current GitLab CI/CD project and identifies logical future improvements.

The current project demonstrates:

```text
Source
  ↓
Build
  ↓
Test
  ↓
Security Scan
  ↓
Docker Build
  ↓
GitLab Container Registry
```

The project intentionally stops at the container registry.

It does not attempt to represent a complete production deployment platform.

---

# 2. Application Ownership Limitation

The VProfile application used in this project is an existing application workload.

The project does not claim ownership of:

- VProfile application development
- VProfile business logic
- Original application architecture
- Original application functionality
- Original application test design

The engineering contribution is focused on the delivery workflow around the application.

The boundary is:

```text
Existing VProfile Application
          │
          ▼
   CI/CD Engineering
```

This distinction should remain explicit when presenting the project professionally.

---

# 3. CI/CD Scope Limitation

The current implementation focuses on continuous integration and container image publication.

The pipeline covers:

- source-triggered execution
- Maven build
- automated testing
- Checkstyle execution
- GitLab rules
- merge-request validation
- Trivy filesystem scanning
- artifact preservation
- Docker image construction
- GitLab Container Registry publication
- commit-SHA image tagging
- failure-triggered notification behavior

The implementation does not provide a complete continuous deployment system.

The current boundary is:

```text
Continuous Integration
        +
Container Publication
        │
        ▼
Container Registry
```

rather than:

```text
Continuous Integration
        +
Continuous Delivery
        +
Continuous Deployment
```

---

# 4. No Automated Deployment

The Docker image is published to the GitLab Container Registry, but the project does not automatically deploy that image to a runtime environment.

The current flow ends at:

```text
Docker Image
      ↓
GitLab Container Registry
```

There is no:

```text
Registry
   ↓
Deployment Platform
   ↓
Running Application
```

## Future Work

A future iteration could introduce:

```text
Container Registry
        ↓
Deployment Stage
        ↓
Runtime Platform
        ↓
Application
```

Possible deployment targets could include a container runtime or Kubernetes environment.

The deployment target should be introduced as a separate engineering iteration rather than being represented as completed work in this project.

---

# 5. No Kubernetes Deployment

Kubernetes is outside the current implementation boundary.

The project does not currently include:

- Kubernetes manifests
- Deployments
- Services
- ConfigMaps
- Secrets
- Ingress
- Horizontal Pod Autoscaler
- Kubernetes health probes
- Kubernetes rollout management

The current pipeline stops before Kubernetes.

## Future Work

A future project could extend the workflow:

```text
GitLab CI
    ↓
Docker Image
    ↓
Container Registry
    ↓
Kubernetes
    ↓
Deployment
    ↓
Service
```

This would create a natural next step from CI/CD into container orchestration.

---

# 6. No Infrastructure as Code

Infrastructure provisioning is not part of the current implementation.

The project does not contain:

- Terraform
- CloudFormation
- Ansible-based infrastructure provisioning
- Automated network provisioning
- Automated compute provisioning
- Automated database provisioning

The project therefore focuses on the application delivery pipeline rather than infrastructure lifecycle management.

## Future Work

A future iteration could introduce Infrastructure as Code:

```text
Infrastructure Code
        ↓
Plan
        ↓
Review
        ↓
Apply
        ↓
Infrastructure
```

Terraform could subsequently become part of the broader DevOps workflow.

---

# 7. Trivy Is Not a Blocking Quality Gate

One of the most important limitations is the behavior of the demonstrated Trivy configuration.

The scan uses:

```text
--exit-code 0
```

Therefore:

```text
Vulnerability Found
       ↓
Report Generated
       ↓
Job can still succeed
```

The current implementation provides:

> Security scanning and reporting.

It does not provide:

> An enforced vulnerability quality gate.

## Future Work

A future security-gate implementation could use:

```text
Trivy
  ↓
Severity Filtering
  ↓
Vulnerability Decision
  ↓
Pass / Fail
```

For example:

```text
CRITICAL vulnerability
        ↓
Pipeline fails
        ↓
Docker publication blocked
```

This would turn security scanning into an actual release-control mechanism.

---

# 8. Filesystem Scanning Only

The current Trivy implementation performs a filesystem scan:

```text
Repository Workspace
        ↓
Trivy FS Scan
        ↓
JSON Report
```

The project does not currently demonstrate a dedicated post-build container image vulnerability scan.

## Future Work

A stronger security pipeline could perform:

```text
Source
  ↓
Filesystem Scan
  ↓
Docker Build
  ↓
Container Image
  ↓
Image Vulnerability Scan
  ↓
Registry
```

This would provide security validation at both:

1. source/dependency level
2. container-image level

---

# 9. No Automated Security Policy

Although Trivy generates a vulnerability report, the project does not implement a formal security policy defining:

- acceptable severity levels
- vulnerability exceptions
- remediation deadlines
- approved base images
- security waiver processes
- vulnerability ownership
- policy-as-code

## Future Work

A future implementation could define:

```text
Security Policy
      ↓
Automated Scan
      ↓
Policy Evaluation
      ↓
Pass / Fail
```

This would move the project toward a more mature DevSecOps model.

---

# 10. No Real External Notification Integration

The current failure notification job uses an `echo` command:

```bash
echo "Build or Test job failed..."
```

The implementation demonstrates the trigger mechanism:

```text
Pipeline Failure
      ↓
notify-on-failure
      ↓
Notification Script
```

but does not integrate with a real notification platform.

## Future Work

A future implementation could connect failure events to:

- Slack
- email
- Microsoft Teams
- another approved notification system

The future architecture would be:

```text
Pipeline Failure
       ↓
Notification Job
       ↓
External Notification
       ↓
Engineer
```

---

# 11. No Automated Rollback

The project does not deploy applications, so it also does not provide automated rollback.

There is currently no:

```text
Deployment
    ↓
Health Check
    ↓
Failure
    ↓
Rollback
```

## Future Work

After deployment is introduced, rollback could be designed around:

```text
Current Version
      ↓
New Version
      ↓
Health Validation
      │
      ├── Pass → Continue
      │
      └── Fail → Rollback
```

Container image tags based on commit SHA provide a useful foundation for identifying previous image versions.

---

# 12. No Runtime Health Validation

The current validation ends at successful image publication.

It does not validate:

- application startup
- HTTP health
- service availability
- database connectivity
- runtime dependency availability
- container health
- response correctness

The current validation is therefore:

```text
Build Validation
      +
Pipeline Validation
      +
Image Publication Validation
```

rather than:

```text
Build Validation
      +
Runtime Validation
```

## Future Work

After deployment, a future pipeline could perform:

```text
Deploy
  ↓
Health Check
  ↓
Smoke Test
  ↓
Functional Validation
```

---

# 13. No Production Observability

The project does not include production observability.

There is no implementation for:

- application metrics
- container metrics
- centralized logging
- distributed tracing
- alerting
- dashboards
- SLOs
- SLIs

## Future Work

A mature delivery platform could connect deployment with observability:

```text
Deployment
    ↓
Runtime
    ├── Metrics
    ├── Logs
    ├── Traces
    └── Alerts
```

This would allow deployment success to be evaluated using actual runtime signals.

---

# 14. Docker-in-Docker Tradeoff

The Docker publishing stage uses:

```yaml
image: docker:latest
services:
  - docker:dind
```

This provides a straightforward way to build and push Docker images inside GitLab CI.

However, Docker-in-Docker introduces additional execution complexity and is not the only possible container-build architecture.

The current project does not evaluate alternative image-building approaches.

## Future Work

A future iteration could compare:

```text
Docker-in-Docker
        vs
Alternative Rootless / Daemonless Builders
```

based on:

- security
- isolation
- performance
- caching
- operational complexity
- runner requirements

The current project does not claim that Docker-in-Docker is universally optimal.

---

# 15. Use of `latest` Image Tags

The pipeline uses images such as:

```text
maven:3.9.9-eclipse-temurin-17
docker:latest
aquasec/trivy:latest
```

The Maven image is relatively specific, while `latest` introduces a mutable dependency.

A future environment could prefer more tightly pinned versions or digests.

## Future Work

For stronger reproducibility:

```text
Image Name
    +
Explicit Version
    +
Digest Pinning
```

could be used.

This reduces the possibility that a pipeline changes behavior because an upstream `latest` image changed.

---

# 16. No Advanced Dependency Caching

The current project does not demonstrate advanced Maven dependency caching.

Each pipeline execution can therefore spend time obtaining required dependencies.

## Future Work

A future pipeline could use GitLab cache mechanisms:

```text
Maven Dependencies
       ↓
Cache
       ↓
Subsequent Pipeline
       ↓
Faster Build
```

This could reduce build time while keeping artifacts and caches conceptually separate.

---

# 17. No Advanced Pipeline Templates

The current `.gitlab-ci.yml` is project-specific.

It does not currently demonstrate reusable GitLab CI templates or centralized pipeline components.

## Future Work

Common pipeline functionality could be abstracted:

```text
Reusable CI Template
        │
        ├── Java Build
        ├── Testing
        ├── Security
        └── Docker
```

Multiple projects could then consume standardized pipeline components.

This would be more appropriate as the number of repositories increases.

---

# 18. No Advanced Deployment Strategies

Because deployment is outside the current scope, the project does not demonstrate:

- rolling deployments
- blue/green deployments
- canary deployments
- progressive delivery
- automated traffic shifting

## Future Work

After deployment is introduced, these strategies could be evaluated based on application requirements.

A conceptual progression is:

```text
Basic Deployment
      ↓
Rolling Deployment
      ↓
Controlled Progressive Delivery
      ↓
Automated Rollback
```

---

# 19. No Environment Promotion Model

The current project does not define:

```text
Development
     ↓
Staging
     ↓
Production
```

or environment-specific approvals.

The container image is published to the registry but is not promoted across runtime environments.

## Future Work

A future CI/CD platform could implement:

```text
Build Once
   ↓
Test
   ↓
Security
   ↓
Registry
   ↓
Development
   ↓
Staging
   ↓
Production
```

The same immutable image could be promoted rather than rebuilt for each environment.

The commit-SHA tagging approach already provides a useful foundation for this model.

---

# 20. No Manual Approval Gates

The current project does not demonstrate protected production deployment approvals.

## Future Work

A future production workflow could introduce:

```text
Automated Validation
        ↓
Production Approval
        ↓
Deployment
```

This would be appropriate when production changes require explicit human authorization.

---

# 21. No Pipeline Performance Optimization

The project introduces parallelism between testing and security scanning through `needs`.

However, it does not perform systematic pipeline performance optimization.

There is no detailed measurement of:

- job duration
- queue time
- dependency download time
- Docker build time
- image push time
- cache hit rate

## Future Work

A future iteration could measure:

```text
Pipeline Duration
       ↓
Identify Bottlenecks
       ↓
Optimize
       ↓
Measure Again
```

Possible improvements include:

- dependency caching
- Docker layer caching
- smaller build contexts
- optimized Dockerfiles
- parallel test execution
- reusable CI templates

---

# 22. No Dedicated CI/CD Test Suite

The project validates the pipeline through actual GitLab execution.

It does not have a separate automated test suite for the `.gitlab-ci.yml` configuration itself.

## Future Work

A mature CI/CD repository could validate:

```text
Pipeline YAML
      ↓
Lint
      ↓
Static Validation
      ↓
Pipeline Simulation
```

This could reduce configuration errors before changes reach the GitLab pipeline.

---

# 23. No Application-Level Performance Testing

The project performs Maven testing and Checkstyle but does not conduct:

- load testing
- stress testing
- endurance testing
- scalability testing
- application benchmarking

## Future Work

Once deployment exists, performance validation could be added:

```text
Deploy
  ↓
Load Test
  ↓
Measure
  ↓
Compare Against Threshold
```

---

# 24. No Infrastructure or Runtime Cost Analysis

The project does not evaluate the infrastructure cost of:

- GitLab runners
- Docker builds
- container registry storage
- deployment infrastructure
- runtime infrastructure

## Future Work

A production-oriented extension could include:

```text
Pipeline
   ↓
Resource Usage
   ↓
Cost Analysis
   ↓
Optimization
```

This becomes increasingly important as pipeline frequency and container volume grow.

---

# 25. Future Work Roadmap

The logical evolution of this project can be organized into stages.

## Phase 1 — Strengthen CI

Current:

```text
Build
 ↓
Test
```

Future:

```text
Build
 ↓
Test
 ↓
Quality Checks
 ↓
Caching
```

---

## Phase 2 — Strengthen DevSecOps

Current:

```text
Trivy
 ↓
Report
```

Future:

```text
Filesystem Scan
      ↓
Dependency Policy
      ↓
Container Image Scan
      ↓
Security Gate
```

---

## Phase 3 — Deployment

Current:

```text
Registry
```

Future:

```text
Registry
   ↓
Deployment Platform
   ↓
Application
```

---

## Phase 4 — Runtime Validation

```text
Deployment
    ↓
Health Check
    ↓
Smoke Test
    ↓
Runtime Validation
```

---

## Phase 5 — Delivery Safety

```text
Deployment
    ↓
Progressive Rollout
    ↓
Observability
    ↓
Automated Decision
    ├── Continue
    └── Rollback
```

---

## Phase 6 — Platform Engineering

The broader future architecture could become:

```text
                    Git Repository
                          │
                          ▼
                    GitLab CI/CD
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
           Build        Security      Test
             │            │            │
             └────────────┼────────────┘
                          ▼
                    Container Image
                          │
                          ▼
                    Image Registry
                          │
                          ▼
                    Deployment
                          │
                          ▼
                    Runtime Platform
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          Metrics        Logs        Traces
             │            │            │
             └────────────┼────────────┘
                          ▼
                     Observability
                          │
                          ▼
                    Delivery Decision
```

This represents a natural progression from the current project toward a more complete DevOps delivery platform.

---

# 26. What This Project Successfully Demonstrates

Despite the limitations, the project provides meaningful CI/CD engineering evidence.

It demonstrates the ability to:

- configure GitLab CI/CD
- define pipeline stages
- create containerized CI jobs
- automate Maven builds
- automate tests
- run Checkstyle
- control execution using GitLab rules
- validate merge-request pipelines
- manage CI/CD variables
- handle masked and protected secrets
- define explicit job dependencies
- run independent validation jobs in parallel
- integrate Trivy
- preserve pipeline artifacts
- configure Docker-in-Docker
- authenticate with the GitLab Container Registry
- build Docker images
- tag images using Git commit SHA
- publish images to a registry
- validate pipeline outputs
- reason about CI/CD architecture and boundaries

These are the core capabilities demonstrated by the current implementation.

---

# 27. Final Project Boundary

The current project should be represented accurately as:

```text
┌────────────────────────────────────────────┐
│             GitLab CI/CD                   │
│                                            │
│  Build → Test → Security → Docker → Push   │
│                                            │
└──────────────────────┬─────────────────────┘
                       │
                       ▼
             Container Registry
```

Not as:

```text
GitLab CI/CD
     ↓
Kubernetes
     ↓
Production
     ↓
Observability
     ↓
Automatic Rollback
```

The second architecture represents future work.

The distinction is important because the strength of the project comes from accurately demonstrating what was actually implemented.

---

# 28. Conclusion

The project establishes a practical CI/CD foundation around the VProfile workload.

The current implementation provides:

```text
Source
  ↓
Automated Build
  ↓
Automated Test
  ↓
Security Scan
  ↓
Artifact Preservation
  ↓
Docker Image
  ↓
Commit-SHA Tag
  ↓
GitLab Container Registry
```

The most important limitations are:

1. no automated deployment
2. no Kubernetes
3. no Infrastructure as Code
4. no blocking Trivy security gate
5. no container-image security scan
6. no runtime health validation
7. no production observability
8. no automated rollback
9. no real external notification integration
10. no multi-environment promotion model

These limitations are not failures of the project. They define the current iteration boundary and provide clear directions for subsequent DevOps projects.

The natural next evolution is:

```text
CI/CD
  ↓
DevSecOps
  ↓
Container Deployment
  ↓
Kubernetes
  ↓
Infrastructure as Code
  ↓
Observability
  ↓
Progressive Delivery
```

The central principle for future iterations is:

> **Extend the delivery system incrementally: strengthen validation first, then introduce deployment, runtime verification, infrastructure automation, observability, and safe delivery mechanisms.**

[← Back to README](../README.md) | [← Architecture](architecture.md) | [← Implementation](implementation.md) | [← Validation](validation.md)
