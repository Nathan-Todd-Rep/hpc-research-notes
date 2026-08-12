# HPC Introduction

## What Is HPC?

An aggregation of computing power put together to solve problems that are too complex for a normal computer, or that would otherwise take too long. HPC systems can simulate and analyze huge volumes of data.

## Common Challenges

- Producing more data than your infrastructure can handle — this could leave you waiting weeks or months for results.

## How It Works

- A group of computers is called a **cluster**. Each computer in a cluster is called a **node**.
- Each node has an operating system, multiple cores, storage, and networking.
- Example: a smaller cluster could have 16 nodes with 64 cores (4 cores per processor).

The three building blocks of HPC:

| Building Block | Role |
|---|---|
| Compute | Processing and running jobs |
| Storage | Holding data and results |
| Networking | Connecting nodes together |

---

## Related Notes

- [[MIT Course Overview + The Shell]] — shell skills needed to work on an HPC cluster
- [[Linux refresher]] — Linux is the OS on every HPC node
- [[Apptainer Singularity]] — containers used for reproducible HPC software
- [[Inkly Codebase Notes Start]] — an AI assistant built for HPC clusters
