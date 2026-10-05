# CI/CD Self-Study Notes

## Understanding the Three Delivery Practices

Continuous Integration (CI), Continuous Delivery, and Continuous Deployment are related DevOps practices used to automate and improve the software delivery process. Although they are commonly grouped together as CI/CD, each one represents a different stage or level of automation.

The main difference is what happens after code is changed:

- **Continuous Integration (CI):** changes are frequently integrated, built, and tested.
- **Continuous Delivery:** validated changes are automatically prepared for release, while production release can still require approval.
- **Continuous Deployment:** validated changes move to production automatically without a manual release step.

---

## 1. Continuous Integration (CI)

### What CI Means

Continuous Integration is a development approach in which developers regularly merge their changes into a shared source-control repository. The changes are then automatically built and tested so that integration problems can be discovered early.

The basic idea is simple:

**Developer changes → shared repository → automated build → automated tests → feedback**

### Main Parts of CI

**Source Control Repository**

A central system such as Git stores the project's source code and tracks changes made by developers.

**Automated Build**

A new commit can trigger the CI system to build the application automatically.

**Automated Testing**

Tests are executed as part of the build process to check whether the application continues to behave correctly.

**Feedback**

The CI system reports the build and test results so developers can quickly identify problems.

### Advantages of CI

- Integration problems can be discovered earlier.
- Developers spend less time dealing with large batches of conflicting changes.
- Collaboration becomes easier because changes are integrated regularly.
- Faster feedback can contribute to better code quality and shorter development cycles.

---

## 2. Continuous Delivery

### What Continuous Delivery Means

Continuous Delivery takes the automation beyond integration. Code changes are automatically built, tested, and prepared so that they are always in a releasable state.

The important point is that the software **can be released reliably whenever the team decides to release it**.

A simplified flow is:

**Code change → build → testing → release preparation → ready for production**

Production deployment may still require a person to approve the release.

### Main Parts of Continuous Delivery

**Automated Deployment Pipeline**

A sequence of automated stages handles activities such as building, testing, and preparing the application for release.

**Comprehensive Testing**

The pipeline can include different types of tests, such as unit tests, integration tests, and acceptance tests.

**Staging Environment**

A staging environment provides a production-like location where the application can undergo final checks before release.

**Approval Gates**

A manual approval step can be placed before production deployment when the organization wants human confirmation before releasing the software.

### Advantages of Continuous Delivery

- Releases can be prepared with less manual work.
- The risk associated with a release can be reduced through automated validation.
- Teams can make software available for release more quickly.
- The process can make it easier to respond to changing requirements.

---

## 3. Continuous Deployment

### What Continuous Deployment Means

Continuous Deployment extends Continuous Delivery by removing the manual production-release step. When a change successfully passes the required automated checks, the system automatically deploys it to production.

The basic flow becomes:

**Code change → build → tests → deployment → production**

This means production deployment is part of the automated pipeline rather than a separate manual action.

### Main Parts of Continuous Deployment

**Fully Automated Deployment Pipeline**

Automation covers the path from the source-code change through testing and into production deployment.

**Monitoring and Alerting**

The production environment needs monitoring so that problems can be identified quickly after deployment.

**Rollback Mechanisms**

A rollback process provides a way to return to an earlier working version when a deployment causes a failure.

### Advantages of Continuous Deployment

- Features and fixes can reach users immediately after passing the required checks.
- Less manual intervention is needed during releases.
- Teams can deploy more frequently and receive feedback sooner.

---

## 4. CI/CD Pipeline Workflow

A typical automated pipeline can be viewed as a series of stages:

### Stage 1 — Code Commit

Developers commit their changes to the version-control system.

### Stage 2 — Automated Build

The CI server detects the change and builds the application.

### Stage 3 — Automated Testing

Automated tests are executed to validate the new build.

### Stage 4 — Artifact Creation

When the required tests pass, the pipeline creates build artifacts. Examples include application binaries and Docker images.

### Stage 5 — Staging Deployment

The generated artifacts are deployed to a staging environment for additional validation.

### Stage 6 — Acceptance Testing

Further tests are performed in the staging environment to verify that the application meets the expected requirements.

### Stage 7 — Approval

For a Continuous Delivery workflow, a manual approval can be required before the application is released to production.

### Stage 8 — Production Deployment

The validated application is deployed to the production environment. In Continuous Deployment, this stage can happen automatically once the pipeline requirements have been satisfied.

---

## 5. Common CI/CD Tools and Technologies

Different parts of a CI/CD pipeline can be handled by different tools.

| Area | Examples |
|---|---|
| Version Control | Git, Subversion |
| CI/CD Servers | Jenkins, GitLab CI, CircleCI, Travis CI |
| Build Tools | Maven, Gradle, Ant |
| Containerization | Docker, Kubernetes |
| Testing | JUnit, Selenium, TestNG |
| Monitoring | Prometheus, Grafana, Nagios |

These tools can be combined to create an automated development and delivery pipeline.

---

## 6. CI/CD Best Practices

### Commit Changes Regularly

Frequent commits keep changes smaller and make integration problems easier to identify.

### Keep Builds Fast

A slow build delays feedback. Keeping the build process efficient allows developers to discover problems sooner.

### Automate Testing

Automated tests should cover important parts of the application so that changes can be validated consistently.

### Use Feature Flags

Feature flags can be used to control whether particular features are enabled, allowing new functionality to be managed independently of the deployment itself.

### Monitor Continuously

Applications and their environments should be monitored so that problems can be detected and addressed promptly.

### Prepare Rollback Procedures

A deployment should have a recovery plan. If a new version fails, the team should be able to revert to a known working version.

---

## 7. CI vs Continuous Delivery vs Continuous Deployment

| Practice | Main Focus | Production Release |
|---|---|---|
| **Continuous Integration** | Build and test code changes frequently | Not the main focus |
| **Continuous Delivery** | Keep validated software ready for release | May require manual approval |
| **Continuous Deployment** | Automatically release validated changes | Automated |

The three practices can therefore be understood as progressively increasing the amount of automation around software delivery.

---

## Summary

CI/CD practices connect source-code changes with automated validation and software delivery.

**Continuous Integration** focuses on regularly integrating, building, and testing changes.

**Continuous Delivery** takes validated changes and keeps them ready for a production release, with an approval step possible before deployment.

**Continuous Deployment** goes one step further by automatically deploying changes that pass the required checks.

Together, these practices support faster feedback, more repeatable releases, and greater use of automation throughout the software delivery process.
