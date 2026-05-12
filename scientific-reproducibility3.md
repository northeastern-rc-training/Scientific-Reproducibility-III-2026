<img src="images/ac-grid-5-red.svg" alt="drawing" width="900"/>

## Scientific Reproducibility III (container images)

Welcome to the final session of the Research Computing Spring 2026 training series. In this training we will be learning more about Scientific Reproducibility, with a focus on software and container images, which can greatly increase the reproducibility and collaboration of your research efforts.

Today this presentation will cover:

1. [What is a container?](#what-is-a-container)
2. [What are some advantages of containers?](#what-are-some-advantages-of-containers)
3. [How to run a container (on CPU or GPU).](#how-to-run-a-container)
4. [Binding directories to your container.](#binding-directories-to-your-container)
5. [How to build a container.](#how-to-build-a-container)
6. [How to pull your container to the cluster.](#how-to-pull-your-container-to-the-cluster)
7. [Some tips and tricks.](#tips-and-tricks)
8. [How to get help](#how-to-get-help)

## What is a container?

[#what-is-a-container](#what-is-a-container)

A container is a lightweight, portable, self-contained unit of software that packages up an application and all its dependencies — libraries, configuration files, binaries, and runtime — into a single file or image. This means the software inside the container will behave the same way regardless of what machine it is running on.

[![drawing](images/container_diagram.svg)](images/container_diagram.svg)

Think of a container as a sealed lunchbox: everything your application needs for lunch is already inside. It doesn't matter whose kitchen you use — the contents are the same.

On HPC clusters like Explorer, we use **[Apptainer](https://apptainer.org/)** (formerly known as Singularity) as our container platform. Unlike Docker (which is common on laptops and cloud systems), Apptainer is designed specifically for shared HPC environments where users do not have root (administrator) privileges. Apptainer containers run with your user permissions, making them safe for multi-user clusters.

Container images on the cluster are typically stored as a single `.sif` file (Singularity Image Format).

> **Note:** You may see containers referred to as Docker, Singularity, or Apptainer images in different contexts. On Explorer, we use Apptainer, but Apptainer is fully compatible with Docker images and can pull directly from Docker Hub.

## What are some advantages of containers?

[#what-are-some-advantages-of-containers](#what-are-some-advantages-of-containers)

Containers offer several compelling advantages for scientific computing:

**Reproducibility**

Because the entire software environment is captured inside the container image, your results are tied to a specific, versioned software stack. A collaborator running your container six months from now will get the exact same results you did today — even if the underlying system has been updated.

**Portability**

A container built on your laptop can be run on Explorer, on another HPC cluster, or in the cloud without modification. You write your software environment once and run it anywhere.

**No dependency conflicts**

Have you ever needed two programs that require incompatible versions of the same library? Containers solve this problem because each container carries its own isolated set of dependencies.

**No root required at runtime**

On a shared HPC cluster you do not have administrator privileges. Conda environments and module installs are great for many cases, but containers allow you to install software as if you did have full control — you just do the build step on a machine where you do have permissions (such as your own laptop or a cloud VM), and then bring the finished `.sif` file to Explorer.

**Shareability**

A `.sif` file is a single file that can be shared with collaborators, uploaded to a container registry, or archived with your data for long-term reproducibility. You can point a reviewer directly to the container image used to produce your results.

**Access to community-built images**

Registries like [Docker Hub](https://hub.docker.com/), [Quay.io](https://quay.io/), [NVIDIA NGC](https://catalog.ngc.nvidia.com/), and [BioContainers](https://biocontainers.pro/) host thousands of pre-built, community-maintained images for popular scientific software — saving you the effort of building from scratch.

## Binding directories to your container

[#binding-directories-to-your-container](#binding-directories-to-your-container)

By default, a container runs in an isolated filesystem and cannot see the files on your host system (the cluster). To give your container access to data stored in your `/home`, `/scratch`, or `/projects` directories, you need to **bind** those directories into the container.

This is done with the `--bind` (or `-B`) flag.

**Basic syntax:**

```
apptainer exec --bind /source/on/host:/destination/in/container myimage.sif mycommand
```

**Example — binding your projects directory:**

```
apptainer exec --bind /projects/myproject:/data myimage.sif python /data/myscript.py
```

Here, `/projects/myproject` on the cluster is made available inside the container at the path `/data`.

**Binding multiple directories at once:**

```
apptainer exec \
  --bind /projects/myproject:/data \
  --bind /scratch/s.caplins:/scratch \
  myimage.sif python /data/myscript.py
```

**Using environment variables:**

Apptainer also respects the `APPTAINER_BIND` environment variable, which can be convenient in scripts:

```
export APPTAINER_BIND="/projects/myproject:/data,/scratch/s.caplins:/scratch"
apptainer exec myimage.sif python /data/myscript.py
```

> **Note:** Your `/home` directory is automatically bound into the container by default on Explorer. However, `/projects` and `/scratch` are **not** bound by default and must be specified explicitly. Always use absolute paths when binding.

> **Tip:** If you need the same paths inside and outside the container, you can use a simplified bind with just one path: `--bind /projects/myproject` mounts it at the same path inside the container.

## How to run a container

[#how-to-run-a-container](#how-to-run-a-container)

Apptainer provides several ways to interact with a container. The three most common subcommands are:

| Command | Description |
| --- | --- |
| `apptainer exec` | Run a specific command inside the container |
| `apptainer shell` | Open an interactive shell inside the container |
| `apptainer run` | Execute the container's default runscript |

### Running a container on CPU

[#running-a-container-on-cpu](#running-a-container-on-cpu)

---

First, request a compute node (do not run containers on the login node):

```
srun --pty bash
```

**Execute a single command:**

```
apptainer exec --bind /projects/myproject:/data myimage.sif python myscript.py
```

**Open an interactive shell inside the container:**

```
apptainer shell --bind /projects/myproject:/data myimage.sif
```

You will see your prompt change to `Apptainer>`, indicating you are now inside the container. Type `exit` to leave.

**Run the container's default runscript:**

```
apptainer run myimage.sif
```

**Using a container in an sbatch script (CPU):**

```
#!/bin/bash
#SBATCH -J container_job
#SBATCH -p short
#SBATCH -N 1
#SBATCH -n 8
#SBATCH --mem=32GB
#SBATCH --time=04:00:00
#SBATCH --mail-user=s.caplins@northeastern.edu
#SBATCH --mail-type=ALL

# Run the container
apptainer exec \
  --bind /projects/myproject:/data \
  /projects/myproject/containers/myimage.sif \
  python /data/myscript.py
```

Save this script (e.g., `run_container.sh`) and submit it with:

```
sbatch run_container.sh
```

### Running a container on GPU

[#running-a-container-on-gpu](#running-a-container-on-gpu)

---

Containers are especially powerful for GPU workloads because they allow you to bundle a specific version of CUDA and cuDNN into the image itself — no need to worry about which CUDA modules are installed on the system.

To give the container access to the host GPU, add the `--nv` flag to your Apptainer command:

```
apptainer exec --nv --bind /projects/myproject:/data myimage.sif python train.py
```

**Interactive GPU session with a container:**

```
srun --partition=gpu-interactive --gres=gpu:v100-sxm2:1 \
  --ntasks=4 --mem=16GB --time=02:00:00 --nodes=1 --pty bash
```

Once on the compute node:

```
apptainer exec --nv --bind /projects/myproject:/data myimage.sif python train.py
```

**GPU container job via sbatch:**

```
#!/bin/bash
#SBATCH -J gpu_container_job
#SBATCH -p gpu
#SBATCH -N 1
#SBATCH --gres=gpu:v100-sxm2:2
#SBATCH --cpus-per-task=8
#SBATCH --mem=64GB
#SBATCH --time=12:00:00
#SBATCH --mail-user=s.caplins@northeastern.edu
#SBATCH --mail-type=ALL

apptainer exec --nv \
  --bind /projects/myproject:/data \
  /projects/myproject/containers/pytorch_image.sif \
  python /data/train.py --epochs 50 --batch-size 64
```

> **Tip:** NVIDIA maintains an excellent collection of pre-built GPU-optimized container images (PyTorch, TensorFlow, RAPIDS, and more) on the [NGC Catalog](https://catalog.ngc.nvidia.com/). These are a great starting point for deep learning workflows and are regularly updated with optimized CUDA builds.

> **Note:** The `--nv` flag passes through the host GPU drivers to the container. The CUDA version inside the container must be compatible with the GPU driver version on the host node. Check compatible driver versions in the [CUDA release notes](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/).

## Binding directories to your container

[#binding-directories-to-your-container](#binding-directories-to-your-container)

By default, a container runs in an isolated filesystem and cannot see the files on your host system (the cluster). To give your container access to data stored in your `/home`, `/scratch`, or `/projects` directories, you need to **bind** those directories into the container.

This is done with the `--bind` (or `-B`) flag.

**Basic syntax:**

```
apptainer exec --bind /source/on/host:/destination/in/container myimage.sif mycommand
```

**Example — binding your projects directory:**

```
apptainer exec --bind /projects/myproject:/data myimage.sif python /data/myscript.py
```

Here, `/projects/myproject` on the cluster is made available inside the container at the path `/data`.

**Binding multiple directories at once:**

```
apptainer exec \
  --bind /projects/myproject:/data \
  --bind /scratch/s.caplins:/scratch \
  myimage.sif python /data/myscript.py
```

**Using environment variables:**

Apptainer also respects the `APPTAINER_BIND` environment variable, which can be convenient in scripts:

```
export APPTAINER_BIND="/projects/myproject:/data,/scratch/s.caplins:/scratch"
apptainer exec myimage.sif python /data/myscript.py
```

> **Note:** Your `/home` directory is automatically bound into the container by default on Explorer. However, `/projects` and `/scratch` are **not** bound by default and must be specified explicitly. Always use absolute paths when binding.

> **Tip:** If you need the same paths inside and outside the container, you can use a simplified bind with just one path: `--bind /projects/myproject` mounts it at the same path inside the container.


## How to build a container

[#how-to-build-a-container](#how-to-build-a-container)

Building a container requires root (administrator) access, which is not available on Explorer. You have two options for building containers:

- **Build locally** on your own laptop or workstation (requires Docker or Apptainer installed).
- **Build remotely** using the Apptainer Remote Builder or a cloud VM.

### Writing a Definition File (`.def`)

[#writing-a-definition-file](#writing-a-definition-file)

An Apptainer definition file (also called a "def file") is a text recipe that describes how to build your container image. It is analogous to a Dockerfile for Docker.

Here is a simple example that creates a Python environment with `numpy` and `pandas`:

```
Bootstrap: docker
From: python:3.11-slim

%post
    pip install numpy pandas scipy matplotlib

%runscript
    echo "Container is ready. Python version:"
    python --version

%labels
    Author s.caplins@northeastern.edu
    Version 1.0

%help
    This container provides a Python 3.11 environment
    with numpy, pandas, scipy, and matplotlib installed.
```

Key sections of a definition file:

| Section | Purpose |
| --- | --- |
| `Bootstrap` / `From` | The base image to start from (e.g., Docker Hub image) |
| `%post` | Commands to run during the build (install software, configure environment) |
| `%runscript` | Commands executed when you use `apptainer run` |
| `%environment` | Environment variables set at runtime |
| `%labels` | Metadata (author, version, etc.) |
| `%help` | Help text shown with `apptainer run-help` |

### Building with Docker (on your laptop)

[#building-with-docker](#building-with-docker)

If you have Docker installed locally, you can build a Docker image and convert it to a `.sif` file:

```
# Build a Docker image
docker build -t myimage:1.0 .

# Convert Docker image to Apptainer .sif (requires Apptainer installed locally)
apptainer build myimage.sif docker-daemon://myimage:1.0
```

Alternatively, push to Docker Hub and pull from there (see the next section).

### Building with Apptainer locally

[#building-with-apptainer-locally](#building-with-apptainer-locally)

If you have Apptainer installed on your own machine (Linux) with root access, you can build directly from a definition file:

```
sudo apptainer build myimage.sif myimage.def
```

### Using the Apptainer Remote Builder

[#using-the-apptainer-remote-builder](#using-the-apptainer-remote-builder)

Apptainer provides a free [Remote Build Service](https://cloud.sylabs.io/builder) that lets you build containers in the cloud — no local root access required. Create a free account at [cloud.sylabs.io](https://cloud.sylabs.io/) and then:

```
apptainer build --remote myimage.sif myimage.def
```

> **Note:** Building large or complex images on the Remote Builder can take a few minutes. If your build fails, carefully read the error messages — they usually point directly to the problem (a missing package, a typo in a library name, etc.).

## How to pull your container to the cluster

[#how-to-pull-your-container-to-the-cluster](#how-to-pull-your-container-to-the-cluster)

Once you have built or found a container image, you need to get it onto Explorer. There are a few ways to do this.

### Pulling from Docker Hub

[#pulling-from-docker-hub](#pulling-from-docker-hub)

Apptainer can pull Docker images directly from [Docker Hub](https://hub.docker.com/) and convert them to `.sif` format on the fly. First, get a compute node so you are not pulling on a login node:

```
srun --pty bash
```

Then pull the image:

```
apptainer pull docker://ubuntu:22.04
```

This will produce a file called `ubuntu_22.04.sif` in your current directory.

You can also specify a custom output name:

```
apptainer pull --name myubuntu.sif docker://ubuntu:22.04
```

**Pulling a specific tool — for example, GATK from Docker Hub:**

```
apptainer pull docker://broadinstitute/gatk:4.5.0.0
```

**Pulling a GPU-optimized image from NVIDIA NGC:**

```
apptainer pull docker://nvcr.io/nvidia/pytorch:24.01-py3
```

### Pulling from the Sylabs Cloud Library

[#pulling-from-the-sylabs-cloud-library](#pulling-from-the-sylabs-cloud-library)

```
apptainer pull library://lolcow
```

### Transferring a locally built `.sif` file

[#transferring-a-locally-built-sif-file](#transferring-a-locally-built-sif-file)

If you built the container on your laptop, you can transfer the `.sif` file to Explorer using `scp` via the transfer node:

```
scp myimage.sif s.caplins@xfer.discovery.neu.edu:/projects/myproject/containers/
```

Or use the [OOD Files application](https://ood.explorer.northeastern.edu) to upload files up to 10 GB.

> **Tip:** We recommend storing your `.sif` files in your `/projects` directory so they are available to all members of your lab and are not subject to the `/home` 75 GB quota.

> **Tip:** Keep your `.def` (definition) files alongside your `.sif` files in your project directory. The `.def` file is a complete record of how the container was built and is important for reproducibility. Committing the `.def` file to a version-controlled repository (like GitHub) is excellent scientific practice.

## Tips and tricks

[#tips-and-tricks](#tips-and-tricks)

### Setting a default bind path

[#setting-a-default-bind-path](#setting-a-default-bind-path)

Rather than typing `--bind` every time, you can set the `APPTAINER_BIND` environment variable in your `~/.bashrc` (though note the general caution about `.bashrc` from the software module session):

```
export APPTAINER_BIND="/projects/myproject:/data,/scratch/s.caplins:/scratch"
```

### Setting a cache directory

[#setting-a-cache-directory](#setting-a-cache-directory)

When Apptainer pulls or builds images, it stores temporary files in a cache directory, which by default is in your `/home`. If you are pulling large images this can quickly fill your home quota. Set the cache to your `/scratch` instead:

```
export APPTAINER_CACHEDIR="/scratch/s.caplins/.apptainer_cache"
mkdir -p $APPTAINER_CACHEDIR
```

Add this to a job script or your environment before running large pulls.

### Inspecting a container

[#inspecting-a-container](#inspecting-a-container)

You can view the metadata and help text embedded in a container without running it:

```
apptainer inspect myimage.sif
apptainer run-help myimage.sif
```

### Using `--cleanenv`

[#using-cleanenv](#using-cleanenv)

By default, Apptainer passes your shell environment variables into the container. In some cases this can cause unexpected behavior (for example, if you have conda initialized in your environment). The `--cleanenv` flag starts the container with a clean environment:

```
apptainer exec --cleanenv myimage.sif python myscript.py
```

### Testing your container interactively first

[#testing-your-container-interactively-first](#testing-your-container-interactively-first)

Before submitting a long sbatch job, always test your container interactively on a compute node. This helps you catch errors quickly:

```
srun --pty bash
apptainer shell --bind /projects/myproject:/data myimage.sif
# test your commands here
exit
```

### Overlay files for writable containers

[#overlay-files-for-writable-containers](#overlay-files-for-writable-containers)

By default, `.sif` container images are read-only. If you need to install additional packages or write files inside the container filesystem (rather than to a bound directory), you can use an overlay:

```
# Create a 500 MB writable overlay
apptainer overlay create --size 500 myoverlay.img

# Use it with your container
apptainer exec --overlay myoverlay.img myimage.sif pip install newpackage
```

> **Note:** Overlay files are useful for quick experimentation, but for reproducibility we always recommend building a new container with the additional software baked in rather than relying on an overlay.

### BioContainers for bioinformatics tools

[#biocontainers-for-bioinformatics-tools](#biocontainers-for-bioinformatics-tools)

If you work in bioinformatics, [BioContainers](https://biocontainers.pro/) provides community-maintained containers for thousands of bioinformatics tools. Each tool version has its own container, making it easy to use and cite an exact software version. You can search for images at [biocontainers.pro](https://biocontainers.pro/) and pull them directly:

```
apptainer pull docker://biocontainers/samtools:v1.9-4-deb_cv1
```

### Containers in a workflow manager

[#containers-in-a-workflow-manager](#containers-in-a-workflow-manager)

Workflow managers like [Nextflow](https://www.nextflow.io/) and [Snakemake](https://snakemake.readthedocs.io/) have built-in support for running individual steps inside specified containers. This is one of the most powerful patterns for reproducible scientific pipelines, as each tool in your workflow can run in its own container. Ask the RC team if you would like help setting this up.

## How to get help

[#how-to-get-help](#how-to-get-help)

Email the Research Computing team at [rchelp@northeastern.edu](mailto:rchelp@northeastern.edu).

Come to [office hours](https://rc.northeastern.edu/getting-help/) hosted on Zoom.

Or [book a consultation](https://rc.northeastern.edu/getting-help/) with an RC team member.

Review our [documentation on containers](https://rc-docs.northeastern.edu/en/latest/containers/index.html) for more detailed guides and examples.

Thank you for joining the Research Computing Spring 2026 Training Series!

---

*For questions or support, contact the Research Computing team at [rchelp@northeastern.edu](mailto:rchelp@northeastern.edu)*