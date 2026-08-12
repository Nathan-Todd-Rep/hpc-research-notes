# Apptainer / Singularity

## Containers in HPC

A container is a process that has its own isolated view of local resources:
- Filesystem
- User IDs
- Network

A running process sees the real filesystem, but a **container sees a smaller sub-filesystem called an image** as if it were the entire filesystem. Containers promote shareability — you can use images built by others and run them on any machine with the same architecture (x86-64).

### Why Scientists Should Care

- **Reproducibility** — container users are largely unaffected by changes to the cluster environment. If you build software that depends on the current state of the cluster and the cluster changes, your build may break. Containers protect against this.
- **Portability** — build once, run anywhere with the same architecture.

### What Goes Into an Image?

- The OS kernel is **not** duplicated — container images are much smaller than VM images.
- This makes them ideal for shared HPC resources.

## Popular Container Runtimes

LXC, Docker, Singularity (Apptainer), Shifter, Charliecloud, Podman

## Singularity / Apptainer

**Singularity** is an easy-to-use, high-performance container solution produced by Sylabs.
**Apptainer** is a fork of the free version of Singularity, maintained by the Linux Foundation.

Key features:
- Can read and convert any Docker image.
- The filesystem inside is isolated from the host.
- The user inside the container is the **same** as the user outside — important for shared HPC environments.
- Works with HPC cluster technologies like Slurm.

### Important Constraints

- Singularity is **usually only available on compute nodes** — image operations are too CPU-intensive for login nodes.
- Singularity images can be **large on disk** — be aware of your **storage quota** (the limit on disk space per user or research group, applied to home, project, and scratch storage).
- Large image operations may be too intense for the shared filesystem — use a local filesystem for those.

## Getting a Compute Node

To use Singularity you must be on a compute node. Use `srun` to get an interactive session:

```bash
srun --time=120 --mem=4G --pty bash -i
```

| Flag | Meaning |
|---|---|
| `--time=120` | Time limit of 120 minutes |
| `--mem=4G` | Request 4 GB of RAM |
| `--pty` | Attach your shell to the bash session on the compute node |
| `-i` | Interactive mode: bash expects commands from you |

## Setting Up Singularity

Once on a compute node:

```bash
export SINGULARITY_CACHEDIR=$TMPDIR   # point cache at local temp storage
module load WebProxy                   # connect to internet for image fetching
```

---

## Related Notes

- [[Containerization]] — the broader concept that Apptainer implements
- [[HPC Intro]] — the cluster environment Apptainer runs on
- [[MIT Course Overview + The Shell]] — shell commands needed to work with containers
