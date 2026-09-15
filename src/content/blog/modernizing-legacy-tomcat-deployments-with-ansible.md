---
title: "Modernizing Legacy Tomcat Deployments with Ansible — Without Moving to Kubernetes"
description: "A practical lab showing how to modernize the delivery of a traditional Java WAR on Apache Tomcat using Gitea Actions, LocalStack, Terraform, and Ansible—without containerizing the application."
pubDate: 2026-09-15T12:00:00-06:00
heroImage: "../../assets/legacy-tomcat-automation.png"
tags:
  - DevOps
  - Ansible
  - Tomcat
  - Java
  - Terraform
  - CI/CD
  - Legacy Modernization
---

When teams talk about modernizing legacy applications, the conversation often jumps immediately to containers, Kubernetes, or a complete application rewrite.

But delivery can be modernized **before** the application architecture is.

In this lab, I kept the application as a traditional Java WAR running on Apache Tomcat and focused on modernizing everything around it:

- infrastructure provisioning,
- server configuration,
- CI/CD,
- artifact versioning,
- configuration management,
- secret resolution,
- deployment,
- health validation,
- and idempotency.

The result is a repeatable deployment process for a legacy-style application without requiring the application itself to become cloud native.

> Legacy application does not have to mean legacy delivery process.

---

## The starting point

A common traditional Tomcat deployment looks something like this:

```text
Build WAR
   |
SSH to server
   |
Copy WAR manually
   |
Edit application.properties
   |
Fix permissions
   |
Restart Tomcat
   |
Check logs
```

This works, but it creates several operational problems:

- deployments depend on manual steps,
- server configuration drifts over time,
- application configuration is difficult to reproduce,
- secrets can end up in scripts or configuration repositories,
- rollback depends on someone remembering the previous state,
- and every server can slowly become unique.

The goal of the lab was not to redesign the application.

The goal was to make this delivery process predictable, automate deployments for legacy applications, and reduce the human factor involved in repetitive and error-prone operational tasks.

Instead of relying on someone to remember the correct WAR, configuration, permissions, restart sequence, or validation steps, the deployment process defines those decisions as code and executes them consistently.

---

# Architecture

The final flow looks like this:

```text
Application Repository
        |
        | tag vX.Y.Z
        v
   Gitea Actions
        |
   WAR + SHA-256
        |
        v
   LocalStack S3
        |
        |
        +------------------------------+
                                       |
Configuration Repository               |
        |                              |
        | tag vX.Y.Z                   |
        v                              |
   Gitea Actions                       |
        |                              |
 config.tar.gz + SHA-256               |
        |                              |
        v                              |
   LocalStack S3                       |
        |                              |
        +---------------+--------------+
                        |
                        v
                Ansible Controller
                 /              \
                /                \
          S3 artifacts      Secrets Manager
                \                /
                 \              /
                  v            v
              Verify SHA-256
                    |
             Render configuration
                    |
                    v
               Tomcat Server
                    |
          controlled WAR deployment
                    |
                    v
       /actuator/health -> UP
```

There are three repositories:

```text
legacy-tomcat-demo-app
legacy-tomcat-demo-config
legacy-tomcat-automation-lab
```

Each one has a different responsibility.

---

# 1. The application repository

The application is a simple Spring Boot WAR using Java 21.

It exposes endpoints such as:

```text
GET /api/customers
POST /api/customers
GET /actuator/health
```

The important part is that it is packaged as:

```text
legacy-tomcat-demo.war
```

and is intended to run in an external Tomcat server.

No Docker image is required.

The repository uses Maven Wrapper, so the CI pipeline can build the application with:

```bash
./mvnw -B clean package
```

---

# 2. CI builds a release, not a deployment

For the lab I used a local Gitea instance with a Gitea Actions runner.

Gitea is only simulating an internal CI platform here. The same architecture could be implemented with:

- GitHub Actions,
- GitLab CI,
- Jenkins,
- Azure DevOps,
- Bitbucket Pipelines,
- or another CI system.

A version tag such as:

```text
v2.0.4
```

triggers the application release workflow.

The workflow:

```text
checkout
   |
Java 21
   |
Maven build + tests
   |
WAR
   |
SHA-256
   |
S3
```

The published release looks like:

```text
s3://legacy-artifacts/
└── legacy-tomcat-demo/
    └── 2.0.4/
        ├── legacy-tomcat-demo.war
        └── legacy-tomcat-demo.war.sha256
```

This separation is important:

**CI produces immutable artifacts. It does not SSH into the application server.**

Deployment is a separate responsibility.

---

# 3. Configuration is versioned independently

Application configuration lives in a different repository:

```text
legacy-tomcat-demo-config/
└── environments/
    ├── dev/
    │   └── application.properties.j2
    ├── qa/
    │   └── application.properties.j2
    └── prod/
        └── application.properties.j2
```

A template might contain:

```properties
spring.application.name=legacy-tomcat-demo

spring.datasource.url=jdbc:h2:file:/opt/legacy-demo/data/legacydb
spring.datasource.driver-class-name=org.h2.Driver

spring.datasource.username={{ app_secrets.db_username }}
spring.datasource.password={{ app_secrets.db_password }}

spring.jpa.hibernate.ddl-auto=update

management.endpoints.web.exposure.include=health,info

application.environment={{ app_environment }}
application.version={{ app_version }}
```

Notice what is **not** stored in Git:

```text
database password
```

The configuration release pipeline packages the templates:

```text
config.tar.gz
config.tar.gz.sha256
```

For example:

```text
s3://legacy-artifacts/
└── legacy-tomcat-demo-config/
    └── 1.0.2/
        ├── config.tar.gz
        └── config.tar.gz.sha256
```

This means an application deployment can use:

```text
Application version:   2.0.4
Configuration version: 1.0.2
Environment:           qa
```

The two releases evolve independently.

---

# 4. Terraform is optional

Terraform is used in the lab because I needed to create the environment itself.

It provisions:

```text
Terraform
   |
   +--> Docker
   |     +--> LocalStack
   |     +--> Gitea
   |     +--> Gitea Actions Runner
   |
   +--> Multipass
          |
          +--> Ubuntu 24.04 VM
```

The VM is abstracted behind a small Terraform compute module.

The module exposes a provider-independent contract:

```text
name
host
ssh_user
ssh_port
```

Today the implementation is Multipass.

A future implementation could be:

```text
AWS EC2
Azure VM
VMware
physical Linux server
```

The Ansible layer does not need to change.

This is an important design point:

> Terraform is not required when the infrastructure already exists.

For an existing customer server, the workflow can simply use another Ansible inventory.

---

# 5. cloud-init does the minimum

The VM is initially prepared with cloud-init.

It only handles the minimum bootstrap required for Ansible:

```text
create ansible user
install SSH public key
configure passwordless sudo
install Python 3
```

It deliberately does **not** install:

```text
Java
Tomcat
application
```

Those belong to configuration management.

---

# 6. Ansible configures the server

Ansible installs Java 21 and Apache Tomcat 11.

The Tomcat layout is:

```text
/opt/tomcat/
├── apache-tomcat-11.0.25/
└── current -> apache-tomcat-11.0.25
```

Tomcat runs as a dedicated user:

```text
tomcat
```

and is managed through systemd.

This gives a clear separation:

```text
Terraform  -> infrastructure
cloud-init -> bootstrap
Ansible    -> server state + deployment
```

---

# 7. Secrets are resolved at deployment time

For the lab I used LocalStack Secrets Manager.

The secret name is:

```text
legacy-tomcat-demo/qa/application
```

with a structure similar to:

```json
{
  "db_username": "legacy_user",
  "db_password": "..."
}
```

Terraform creates the **Secrets Manager resource**, but intentionally does not manage the actual secret value.

That prevents the password from being declared in Terraform configuration or stored in Terraform state.

Ansible resolves the secret from the controller:

```yaml
- name: Load application secrets from Secrets Manager
  ansible.builtin.set_fact:
    app_secrets: >-
      {{
        lookup(
          'amazon.aws.secretsmanager_secret',
          secret_name,
          endpoint_url=artifact_endpoint_url,
          region=aws_region
        ) | from_json
      }}
  no_log: true
```

The normalized `app_secrets` object can later be used by the configuration template.

The application server itself does not need AWS CLI or boto3.

---

# 8. Verify artifacts before deploying them

Both application and configuration releases include SHA-256 files.

Before touching Tomcat, Ansible downloads:

```text
legacy-tomcat-demo.war
legacy-tomcat-demo.war.sha256
```

and verifies the artifact.

The same validation is performed for:

```text
config.tar.gz
config.tar.gz.sha256
```

Conceptually:

```text
artifact
   |
calculate SHA-256
   |
   +------ compare ------ published checksum
                        |
                     match?
                    /     \
                  yes      no
                   |        |
                continue   stop
```

A corrupted or unexpected artifact never reaches Tomcat.

---

# 9. Render the final application configuration

After the configuration bundle is verified, Ansible extracts the template for the requested environment.

For QA:

```text
environments/qa/application.properties.j2
```

Ansible combines:

```text
configuration template
        +
Secrets Manager values
        +
app_version
        +
app_environment
```

and produces:

```text
/opt/legacy-demo/config/application.properties
```

with permissions:

```text
tomcat:tomcat
0640
```

The final properties file exists only on the target server.

It is not published back to:

```text
Git
S3
CI artifacts
```

---

# 10. External Spring configuration through Tomcat

Tomcat uses `setenv.sh` to tell Spring Boot where the external configuration lives.

```bash
export SPRING_CONFIG_ADDITIONAL_LOCATION="file:/opt/legacy-demo/config/"
```

The complete relationship becomes:

```text
systemd
   |
catalina.sh
   |
setenv.sh
   |
SPRING_CONFIG_ADDITIONAL_LOCATION
   |
/opt/legacy-demo/config/application.properties
   |
Spring Boot
```

This keeps the WAR independent from environment-specific configuration.

---

# 11. Controlled WAR deployment

One of the parts I wanted to avoid was restarting Tomcat unnecessarily.

Before deployment, Ansible compares the SHA-256 of the desired WAR with the currently deployed WAR.

There are three possible cases.

## WAR changed

```text
stop Tomcat
     |
copy WAR
     |
remove exploded application
     |
clear application work cache
     |
start Tomcat
```

The cache cleanup is application-specific:

```text
/opt/tomcat/current/work/Catalina/localhost/legacy-tomcat-demo
```

It does not blindly delete the entire Tomcat work directory.

## Only configuration changed

```text
render application.properties
        |
restart Tomcat
```

## Nothing changed

```text
no stop
no deployment
no restart
```

This gives us the desired convergence behavior.

---

# 12. Health validation

After every deployment, Ansible waits for:

```text
GET /legacy-tomcat-demo/actuator/health
```

The deployment succeeds only when the application returns:

```json
{
  "status": "UP"
}
```

A successful lab deployment produced:

```text
Application deployed successfully
Application version: 2.0.4
Configuration version: 1.0.2
Environment: qa
Health status: UP
```

---

# 13. Idempotency

The real test was running the exact same deployment again:

```bash
make deploy \
  APP_VERSION=2.0.4 \
  CONFIG_VERSION=1.0.2 \
  APP_ENV=qa
```

The second execution reported:

```text
WAR changed: false
Configuration changed: false
Tomcat environment changed: false
```

Tomcat was not restarted.

The health check still ran and confirmed that the application remained healthy.

That is the behavior I want from configuration management:

```text
desired state already exists
        |
        v
       ok
```

instead of:

```text
run script again
        |
        v
change everything again
```

---

# 14. A simple deployment interface

The root Makefile hides most of the command complexity.

A deployment becomes:

```bash
make deploy \
  APP_VERSION=2.0.4 \
  CONFIG_VERSION=1.0.2 \
  APP_ENV=qa
```

The same automation can target another inventory:

```bash
make deploy \
  ANSIBLE_INVENTORY=/path/to/customer/hosts.ini \
  APP_VERSION=2.0.4 \
  CONFIG_VERSION=1.0.2 \
  APP_ENV=qa
```

That is how the same Ansible roles can move from the local Multipass lab to existing infrastructure.

---

# What about rollback?

Rollback was intentionally left as the next iteration rather than being hidden inside the first implementation.

Because releases are immutable and versioned, a manual rollback can already be expressed as another deployment:

```bash
make deploy \
  APP_VERSION=2.0.4 \
  CONFIG_VERSION=1.0.2 \
  APP_ENV=qa
```

even if the server is currently running a newer version.

In other words, rollback can be treated as:

> redeploy a previously known-good desired state.

For automatic rollback after a failed health check, Ansible could use `block` and `rescue` to restore the previous application/configuration pair.

A useful deployment history could record:

```yaml
app_version: 2.0.4
config_version: 1.0.2
environment: qa
status: healthy
```

There is one major caveat:

## Database migrations

Rolling back the WAR does not automatically roll back the database.

If a newer application version performs an incompatible schema migration, this state may not be safe:

```text
old application
+
new database schema
```

A production rollback strategy therefore also requires a migration strategy such as backward-compatible migrations, Flyway/Liquibase discipline, or expand/contract schema changes.

---

# What about certificates, JKS, and Java cacerts?

This lab focused on the application delivery path, but the same Ansible model can handle common legacy-server requirements such as:

```text
corporate root CAs
intermediate certificates
Java cacerts
JKS
PKCS12
mTLS certificates
proxy certificates
certificate rotation
```

I would keep that responsibility in a separate role, for example:

```text
roles/
├── java/
├── tomcat/
├── java_security/
└── application/
```

That is another advantage of using configuration management for traditional servers: the automation can reproduce more than just the application binary.

---

# What I would improve next

The core objective of the lab is complete, but a few production-oriented improvements remain:

```text
LocalStack persistence
automatic rollback
deployment history
certificate / truststore management
CI deployment approvals
a second infrastructure provider
database migration strategy
```

One practical issue discovered during the lab was that recreating the LocalStack container without persistence removes local S3 objects and secret values.

Adding a persistent Docker volume would make the local environment more durable.

---

# Final result

The application is still:

```text
Java
  |
WAR
  |
Apache Tomcat
```

Nothing about that was changed.

What changed was everything around it:

```text
manual build
    -> versioned CI release

manual file copy
    -> verified S3 artifact

manual properties editing
    -> versioned templates + secret resolution

manual server setup
    -> Ansible

manual restart
    -> conditional controlled restart

manual verification
    -> health check

unknown server state
    -> idempotent desired state
```

Modernization does not always need to start with a rewrite.

Sometimes the best first step is making the system you already have **repeatable, observable, versioned, and safe to operate**.

For legacy applications, this can have an immediate operational impact: deployments become automated and repeatable, while the human factor is reduced from manually executing every deployment step to reviewing, approving, and triggering a defined process.

The objective is not to remove people from the delivery process. It is to remove the dependency on manual memory and repetitive actions for tasks that automation can execute more consistently.
