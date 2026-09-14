---
title: "Reducing Container Attack Surface with Docker Hardened Images"
description: "A practical comparison between a standard Java container image and a Docker Hardened Image using Trivy."
pubDate: 2026-09-14
author: "José Ángel Flores"
tags:
  - Docker
  - DevSecOps
  - Trivy
  - Java
  - Container Security
draft: false
---

Container images are often treated as simple packaging artifacts.

But every package, binary, and tool included in an image increases its potential attack surface.

In this experiment, we compare a traditional Java container image with a Docker Hardened Image using the exact same Spring Boot application.

## What is a hardened container image?

A hardened container image is a container image designed to reduce unnecessary software, privileges, and runtime components that could be abused if the container is compromised.

The idea is simple:

> The less software available inside a production container, the smaller the attack surface.

A traditional base image may include packages, shells, package managers, debugging utilities, and other tools that are useful during development but are not required to run the application in production.

A hardened image attempts to remove or minimize those components.

Typical hardening practices include:

- Reducing the number of installed operating system packages
- Removing unnecessary binaries and utilities
- Avoiding development tools in the final runtime image
- Running applications with reduced privileges
- Using minimal runtime environments
- Keeping the base operating system and runtime dependencies updated
- Reducing the number of components that need to be patched and monitored

This does not mean that a hardened image is automatically secure.

The application running inside the image can still contain vulnerable libraries, insecure configuration, exposed secrets, or exploitable application code.

Container image hardening focuses primarily on reducing the attack surface of the runtime environment.

Application security remains a separate responsibility.

This distinction is important because container security operates at multiple layers:

```text
Application dependencies
        ↓
Application runtime
        ↓
Container image
        ↓
Container runtime
        ↓
Kubernetes / Host
```

Hardening the container image improves one layer of this stack, but it does not replace dependency management, vulnerability remediation, runtime security, or Kubernetes security controls.

## The experiment

For this comparison, I created a small reusable Spring Boot application called **Cloud Native Demo Service**.

The source code used in this experiment is available on GitHub:

[github.com/jalfloresg-cr/cloud-native-demo-service](https://github.com/jalfloresg-cr/cloud-native-demo-service)

The application is intentionally simple so the experiment can focus on the container image rather than application complexity.

We built two images from the same source code:

- `cloud-native-demo-service:standard`
- `cloud-native-demo-service:hardened`

### Build the images

Clone the repository and move into the project directory:

```bash
git clone git@github.com:jalfloresg-cr/cloud-native-demo-service.git
cd cloud-native-demo-service
```

Build the standard image:

```bash
docker build   -f Dockerfile.standard   -t cloud-native-demo-service:standard   .
```

Build the hardened image:

```bash
docker build   -f Dockerfile.hardened   -t cloud-native-demo-service:hardened   .
```

Verify that both images were created:

```bash
docker images cloud-native-demo-service
```

You should now have two local images built from the same application source code and ready for comparison.

### Compare image size

Before running the vulnerability scan, compare the size of both images:

```bash
docker images cloud-native-demo-service
```

You can also print only the image name and size:

```bash
docker image ls   --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"   cloud-native-demo-service
```

Record the result:

| Image | Size |
|---|---:|
| Standard | `<standard-size>` |
| Hardened | `<hardened-size>` |

The image size is useful evidence because both images contain the same application artifact.

If the hardened image is significantly smaller, that strongly suggests that the runtime contains fewer packages, binaries, libraries, and supporting utilities.

That is exactly what we want from a hardened production image: only the components required to run the workload.

However, image size should not be treated as a security metric by itself.

A smaller image is not automatically secure, and a larger image is not automatically insecure. In this experiment, size is supporting evidence that the hardened runtime contains less software, while Trivy gives us a second view by showing the vulnerability findings associated with those components.

Both images contain the exact same application.

The main difference is the runtime base image.

This allows us to isolate the effect that the container runtime image has on the overall vulnerability profile while keeping the application layer unchanged.

## Security scan

Both images were scanned using Trivy, focusing on HIGH and CRITICAL vulnerabilities with known fixes:

```bash
trivy image   --scanners vuln   --severity HIGH,CRITICAL   --ignore-unfixed   cloud-native-demo-service:standard
```

And the hardened image:

```bash
trivy image   --scanners vuln   --severity HIGH,CRITICAL   --ignore-unfixed   cloud-native-demo-service:hardened
```

## Results

The comparison should be evaluated from two perspectives: **how much software is shipped in the runtime image** and **how many vulnerabilities are detected**.

### Image footprint

| Image | Size |
|---|---:|
| Standard | `<standard-size>` |
| Hardened | `<hardened-size>` |

Because the application JAR is the same in both images, the difference in final image size comes primarily from the runtime layer.

A smaller hardened image provides visible evidence that less software is being shipped to production.

### Trivy findings

The hardened image produced the following Trivy summary:

```text
Report Summary

┌──────────────────────────────────────────────────┬────────┬─────────────────┐
│                      Target                      │  Type  │ Vulnerabilities │
├──────────────────────────────────────────────────┼────────┼─────────────────┤
│ cloud-native-demo-service:hardened (alpine 3.22) │ alpine │        0        │
├──────────────────────────────────────────────────┼────────┼─────────────────┤
│ app/app.jar                                      │  jar   │       22        │
└──────────────────────────────────────────────────┴────────┴─────────────────┘
```

The standard image produced:

```text
Report Summary

┌───────────────────────────────────────────────────┬──────────┬─────────────────┐
│                      Target                       │   Type   │ Vulnerabilities │
├───────────────────────────────────────────────────┼──────────┼─────────────────┤
│ cloud-native-demo-service:standard (ubuntu 26.04) │  ubuntu  │        0        │
├───────────────────────────────────────────────────┼──────────┼─────────────────┤
│ app/app.jar                                       │   jar    │       22        │
├───────────────────────────────────────────────────┼──────────┼─────────────────┤
│ usr/bin/pebble                                    │ gobinary │        8        │
└───────────────────────────────────────────────────┴──────────┴─────────────────┘
```

### Interpreting the results

The most important observation is that **both operating system layers reported zero HIGH or CRITICAL vulnerabilities under the scan settings used**.

This means the difference between the two images did not come from Alpine versus Ubuntu vulnerabilities in this particular scan.

The application layer was also identical:

| Target | Standard | Hardened |
|---|---:|---:|
| Operating system | 0 | 0 |
| `app.jar` | 22 | 22 |
| Additional runtime binary | 8 | 0 |
| **Total** | **30** | **22** |

The same `app.jar` was copied into both images, so Trivy correctly reported the same **22 Java dependency findings** in both cases.

The difference came from an additional runtime component in the standard image:

```text
/usr/bin/pebble
```

Trivy identified `pebble` as a Go binary and reported **8 vulnerability findings** associated with components embedded in that binary.

The hardened image did not contain this additional runtime binary, so those findings were not present.

This is a good example of what reducing attack surface means in practice:

> Software that is not shipped in the production image cannot contribute vulnerabilities to that image.

The result is not that the hardened image somehow fixed the application's Java dependencies. It simply shipped **less runtime software** around the application.

That distinction matters.

The 22 findings in the application still need to be addressed through dependency upgrades or other application-level remediation.

Also, a vulnerability scanner finding does not automatically mean that a vulnerability is exploitable in the context of the running application. Reachability, configuration, and runtime usage still matter. However, removing unnecessary components reduces the number of things that need to be analyzed, patched, and monitored.

### What the comparison actually proves

For this experiment, the evidence supports three conclusions:

1. **The hardened image reduced the runtime footprint.**  
   It did not include the additional `pebble` binary detected in the standard image.

2. **The reduction in runtime components removed 8 scanner findings.**  
   The standard image reported 30 findings in total, while the hardened image reported 22.

3. **Container hardening did not change application vulnerabilities.**  
   Both images contained the same Spring Boot JAR and therefore reported the same 22 Java dependency findings.

This is why hardened images should be viewed as one layer of a broader container security strategy rather than as a replacement for application dependency management.

## The important lesson

A hardened container image can significantly reduce the attack surface of the runtime environment.

But it cannot fix vulnerable application dependencies.

In this experiment, vulnerabilities originating from Spring Boot, Tomcat, Jackson, and other Java dependencies remained present in both images.

This highlights an important distinction:

**Container hardening and application dependency management solve different security problems.**

A hardened runtime helps reduce the number of operating system packages, binaries, and utilities available inside the container.

Dependency management, on the other hand, requires upgrading or replacing vulnerable application libraries.

A strong container security strategy needs both.

## Why this matters

Reducing vulnerabilities in the base image is valuable, but vulnerability count alone should not be the only metric.

A smaller and hardened runtime can also provide:

- Fewer installed packages
- Fewer binaries available to an attacker
- Reduced operating system attack surface
- Safer runtime defaults
- Less unnecessary tooling inside production containers

This follows a simple principle:

> If the application does not need a tool at runtime, the container probably should not include it.

## What's next?

This is only the first step of the experiment.

The next phase should address the **22 vulnerabilities that remain inside the application layer**.

That means moving from container hardening to dependency remediation.

For the Java application, the next steps will include:

- Identifying which Maven dependencies introduce the vulnerable components
- Reviewing direct and transitive dependencies
- Upgrading Spring Boot and affected libraries where fixes are available
- Verifying whether vulnerable dependencies are actually required by the application
- Removing unused dependencies where possible
- Rebuilding the application and comparing the Trivy results again

Useful commands for this analysis include:

```bash
mvn dependency:tree
```

and:

```bash
trivy fs   --scanners vuln   --severity HIGH,CRITICAL   .
```

This will help us separate vulnerabilities introduced by the application dependency graph from those introduced by the container runtime.

After addressing the application layer, we will continue extending the software supply chain with:

- SBOM generation
- Cosign image signing
- SBOM attestations
- Vulnerability attestations
- CI/CD security enforcement
- Kubernetes admission policies

The goal is to progressively secure each layer instead of treating the container image as a single security boundary:

```text
Application dependencies
        ↓
Container image
        ↓
CI/CD pipeline
        ↓
Artifact signing and attestations
        ↓
Kubernetes admission controls
```

A hardened image reduces runtime attack surface, but a complete strategy also requires actively managing the vulnerabilities that remain inside the application itself.
