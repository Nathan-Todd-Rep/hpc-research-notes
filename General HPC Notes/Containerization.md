# Containerization

## What Is Containerization?

Containerization is the packaging of software code together with all its dependencies (libraries, frameworks, etc.) so that everything is isolated in its own **container**. The container acts as a bubble around the application, keeping it independent of the environment it runs in.

This means the software can be moved and run consistently on any infrastructure, regardless of the host operating system.

## Why It Matters

- Coding on a single platform made it difficult to share software — code that worked on one machine might not work on another, causing bugs.
- Containers solve this: the same container runs identically everywhere.
- Containers are **lightweight** and **portable** because they share the host OS kernel — no need for a separate OS inside each container.

## Containers vs Virtual Machines

| | Containers | Virtual Machines (VMs) |
|---|---|---|
| Size | Megabytes | Gigabytes |
| OS | Shares the host kernel | Has its own full OS |
| Portability | Very high | High |
| Performance overhead | Very low | Higher |
| Use case | Single app or microservice | Multiple resource-intensive functions |

Both can run in nearly any environment, but containers are much leaner.

## Microservices

Containers are often used to package **microservices** — individual, specialized functions broken out from a larger application. This lets developers work on one part of a system independently, without impacting the rest of the app's performance.

## Container Orchestration

**Container orchestration** is the automation of the deployment, management, scaling, and networking of containers across a system.

---

## Related Notes

- [[Apptainer Singularity]] — the container runtime used on HPC clusters
- [[HPC Intro]] — the cluster environment containers run on
