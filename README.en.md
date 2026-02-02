# DevOps Best Practices

## Table of Contents

- [DevOps Best Practices](#devops-best-practices)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [IaC](#iac)
    - [IaC and CI/CD](#iac-and-cicd)
    - [Terraform State File Security](#terraform-state-file-security)
  - [Environments](#environments)
  - [Containers](#containers)
    - [Versioning and Tags](#versioning-and-tags)
    - [Container Runtime Options](#container-runtime-options)
    - [Container Security](#container-security)
    - [Registry](#registry)
  - [Automated Testing](#automated-testing)
  - [Secrets](#secrets)
  - [DR (Disaster Recovery)](#dr-disaster-recovery)
    - [Databases](#databases)
    - [File Systems](#file-systems)
    - [Object Storage](#object-storage)
    - [Recovery Drills](#recovery-drills)
  - [Scalability](#scalability)
    - [Database Scalability](#database-scalability)
  - [Monitoring](#monitoring)
    - [Metrics](#metrics)
    - [Logs](#logs)
    - [Traces](#traces)
    - [Alerts](#alerts)
    - [Dashboard](#dashboard)
    - [Conventional Stack](#conventional-stack)

## Introduction

This is a summary of my learning of DevOps best practices, based on my practical experience and field learning.

## IaC

All infrastructure should be able to be recreated from scratch using code, ensuring it is identical to the existing one.

Terraform is the recommended tool. Reason: largest ecosystem with the greatest coverage of different providers.

IaC code should be maintained in a Git repository. This ensures change traceability.

Option: GitOps. If using Kubernetes, part of the IaC should reside in files maintained in Git, which describe the desired state in YAML, and synchronized with the cluster through GitOps tools like Argo CD or Flux.

All IaC code (Terraform, GitOps manifests, etc.) should be maintained in Git repositories separate from application code.

This ensures access separation since developers may or may not have access to the IaC configuration of different environments. Additionally, workflows (e.g., GitHub Actions) can have unwanted side effects if executed on IaC or application code.

### IaC and CI/CD

For tools like Terraform, it is desirable that scripts be executed via CI/CD pipelines. This ensures consistency (always the same execution environment) and traceability (results are recorded in the workflow). It is possible to use simulation tools like `terraform plan`/`validate` for execution in PRs before merge and execution of `terraform apply` on the main branch.

In GitOps (Argo CD etc.) the CI/CD flow starts in the application repository, where deployment executes a `git push` of manifests to the GitOps repository and Argo CD synchronizes the manifests with the cluster.

### Terraform State File Security

Terraform stores the current state in a state file for comparison with the existing state when calculating changes to be applied. This file can often contain sensitive data such as passwords, tokens, credentials, etc. It is essential that the CI/CD pipeline configures Terraform to store the state file securely. Recommendation: use S3 backend. Credentials can be configured via GitHub Secrets, GitLab Secrets, etc. as per [Secrets](#secrets).

## Environments

In addition to the production environment, a Staging environment should be created. The main purpose of the Staging environment is to apply changes to an environment identical to production to ensure changes do not break the production environment. As per [IaC](#iac), in all environments changes should go through Git.

The Staging environment should always be kept identical to production, except for sizing differences (e.g., less CPU, memory, storage, etc.) but always with the same components and versions. In particular, for example, if replication or load balancing is used in production, the Staging environment should be configured the same way.

The Staging environment should not be used for frequent changes as in a development environment. For this, create a Dev environment. In this environment, day-to-day changes that are part of the product development process should be made.

When applying changes to the Staging environment, a suite of functional tests should be executed covering all journeys of different user profiles, performance tests, etc., to ensure changes do not break the application. If the test passes, the change should be promoted to the production environment during pre-established time windows. If the test fails, the change in the Staging environment should be reverted so it remains equal to production.

For [IaC](#iac) Git repositories, do not use separate branches for environments. Segment environments by folders or separate repositories. Branches are used for merges and there is no reason to merge between environments. A single branch (trunk-based) should be used. Ephemeral branches can be created only for infrastructure PRs and approval processes, but applying the IaC definition to the environment should always be done from the main branch (`main`).

## Containers

All code should be containerized. This ensures it runs consistently, whether locally or in different environments. The Dockerfile definition should be maintained alongside the application code.

### Versioning and Tags

Via CI/CD (GitHub Actions, GitLab CI, etc.), automate the creation of image tags when Git repository tags are created. This way images have tags synchronized with the code.

Use the Semantic Versioning concept (<https://semver.org/>) for repository tag creation.

Use the convention <https://www.conventionalcommits.org/> for commit messages. It is possible, via GitHub Actions, to automate the creation of image tags and Changelogs when commits are created. Examples:

- <https://github.com/marketplace/actions/semver-conventional-commits>
- <https://github.com/marketplace/actions/changelog-from-conventional-commits>

In [IaC](#iac) definitions, images should be referenced by a pinned tag and not by a dynamic tag (latest). It is possible to use latest in the Dev environment if desired. In Staging and Production environments, images should be referenced by the same tag, except when updating the image to a newer version (first in Staging, then in Production).

### Container Runtime Options

There are two options for container runtime:

- On the cloud provider, examples: AWS ECS, Google Cloud Run, etc.
- Via a Kubernetes cluster, example: AWS EKS, Google Kubernetes Engine, etc.

The choice should be made considering the DevOps team's skills. Managing a Kubernetes cluster, even from managed offerings like EKS and GKE, requires greater team maturity. Reasons:

- The default cluster configuration is delivered without necessary hardening for production such as network segmentation, service account scope limitation, etc.
- Management of cluster upgrades and its different components (ingress, cert-manager, etc.) on an ongoing basis.
- Creation of internal monitoring, logging, etc. infrastructure.

### Container Security

Use a tool such as <https://trivy.dev/> to scan images for vulnerabilities embedded in the CI/CD pipeline, along with image creation from the Dockerfile.

Use "lean" OS bases like Alpine Linux or distroless to reduce the application's attack surface.

### Registry

Generated images need a registry for storage. Using GitHub, GitLab, etc. registry is more practical. Protect the registry with token authentication, which should be synchronized via [Secrets](#secrets).

## Automated Testing

For each API endpoint, unit tests should be created covering all HTTP methods (GET, POST, PUT, DELETE, etc.) and all possible combinations of input parameters, including cases of invalid, non-existent, or default data.

Also create end-to-end (E2E) functional tests where data entry is done by simulating different journeys of different user profiles, through headless browser interaction, and ensuring that data persisted in the database matches expectations for each case.

Include automated tests in the CI/CD pipeline to ensure they pass before promotion to the Staging environment.

Such tests should also be used in PR approvals, ideally also automatically.

## Secrets

Never keep secrets in source code. Options for secret management:

- Secret management system, such as AWS Secrets Manager, etc.
- Secrets in GitHub Secrets, GitLab Secrets, etc.: ideal for IaC secrets like cloud provider credentials or container image registry access.
- Encrypted secrets in GitOps, example: [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)

Via [IaC](#iac), configure containers so that secrets are injected as environment variables and access them this way via application code.

Set up a secret rotation scheme, using IaC automation to synchronize with applications.

## DR (Disaster Recovery)

Persistent data can come from databases, file systems, or object storage.

### Databases

Full database backups should be made according to an automatic schedule and an also automated retention policy. For example:

- 3 most recent daily backups
- 2 most recent weekly backups
- 2 most recent monthly backups
- 5 most recent annual backups

The backup destination should be storage present in at least one geographic location different from the database. For example, a multi-zone S3 bucket.

Backup contents should be protected by credentials separate from those used within the database environment. For example, using [versioning in S3 buckets](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html), one can configure a credential with access only to the current version (for the backup agent residing in the database or production environment) and another, separate and inaccessible by the production environment, with access to previous versions (for recovery).

It is desirable to complement with log backups, for example, with log shipping every 5 minutes. With this, RPO (Recovery Point Objective) can be reduced to minutes.

### File Systems

Using file systems for persistent data is not desirable, as it compromises modern software architectures like [12 Factor App](https://12factor.net/). If unavoidable, follow similar concepts to [Databases](#databases). For media files like images, videos, etc., use object storage.

### Object Storage

Configure object storage as multi-zone to ensure high availability. Use [versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html) to ensure persisted data is protected against loss or corruption and configure separate credentials for the execution environment and backup environment, so that the production environment does not have access to previous bucket versions, ensuring greater protection for eventual recovery.

### Recovery Drills

A recovery drill should be executed periodically (suggestion: quarterly or semi-annually). The Staging environment can be used for this. Execute the production promotion functional tests to validate recovery.

## Scalability

The [12 Factor App](https://12factor.net/) architecture should be used to ensure the application is scalable. The following components are used to build this architecture:

- Load balancer: distributes access across replicas
- Autoscaling: uses a trigger like queue, requests per second, etc. to automatically increase/decrease the number of replicas

### Database Scalability

For databases, use read replicas to distribute load. When necessary, use connection optimization components like [PgBouncer](https://www.pgbouncer.org/) for PostgreSQL or [ProxySQL](https://proxysql.com/) for MySQL.

Such tools should be configured via [IaC](#iac) using cloud provider resources or, if running databases in Kubernetes clusters, via GitOps, through controllers like [CloudNativePG](https://cloudnative-pg.io/).

## Monitoring

The monitoring stack should include:

- Metrics
- Logs
- Traces
- Alerts
- Dashboards

### Metrics

Metrics are indicators like CPU, memory, requests per second, etc. that are exported in temporal samples for later analysis, visualization in graphs, and alert triggering.

Via [IaC](#iac), configure metrics exposure using native cloud provider capabilities along with application-exposed metrics. Metrics should be exposed in a standard like Prometheus or OpenTelemetry.

### Logs

Configure the application to export logs via `stdout` and `stderr` in accordance with [12 Factor App](https://12factor.net/logs). The infrastructure should automatically capture and process such logs transparently to the application.

### Traces

Traces are used to map requests as they flow through different application components and databases.

Via [IaC](#iac), configure trace capture and processing using native cloud provider capabilities along with application-exposed traces. Traces should be exposed in a standard like OpenTelemetry.

### Alerts

Alert configuration should be done via [IaC](#iac) using native cloud provider capabilities along with application-exposed alerts.

Such configuration should include desired channels for receiving alerts according to the tool in use by the company, such as Slack, Teams, etc., and should also be made available through the monitoring Dashboard.

### Dashboard

A monitoring dashboard should be made available for visualization of different monitoring indicators. The universal standard is [Grafana](https://grafana.com/).

### Conventional Stack

It is recommended to use the most mature and consolidated tools. An example of a "cloud-native" stack for use in Kubernetes:

- Prometheus for metrics
- Alertmanager for alerts
- Grafana for dashboards
- Loki for logging
- Tempo for traces

Cloud providers usually offer equivalent and integrated solutions within their own ecosystem. For example, AWS ECS container runtime offers native log integration via CloudWatch and metrics/traces via OpenTelemetry.

Configure all observability wiring between application components via [IaC](#iac).

For Kubernetes, it is recommended to use Helm Charts with bundles that include components already integrated. Examples (components can be used together or separately):

- [Kube Prometheus Stack](https://github.com/prometheus-community/helm-charts/charts/kube-prometheus-stack)
- [Grafana K8s Monitoring](https://github.com/grafana/k8s-monitoring-helm)
