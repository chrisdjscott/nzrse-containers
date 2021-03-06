---
title: "Basics of Singularity"
teaching: 15
exercises: 15
questions:
objectives:
- Download container images
- Run commands from inside a container
- Discuss some popular image registries
keypoints:
- Singularity can run both Singularity and Docker container images
- Execute commands in containers with `singularity exec`
- Open a shell in a container with `singularity shell`
- Download a container image in a selected location with `singularity pull`
- You should not use the `latest` tag, as it may limit workflow reproducibility
- The most commonly used registries are Docker Hub, Quay, Biocontainers and Nvidia GPU Cloud
---


> ## NZ RSE 2020 attendees: let's login
>
> Assuming you have followed NeSI's support [documentation](https://support.nesi.org.nz/hc/en-gb/sections/360000034315) on setting passwords and [configuring local SSH client software](https://support.nesi.org.nz/hc/en-gb/articles/360001016335-Choosing-and-Configuring-Software-for-Connecting-to-the-Clusters) then you should login to Mahuika, e.g., using a [standard terminal configuration](https://support.nesi.org.nz/hc/en-gb/articles/360000625535):
> `ssh mahuika`
{: .callout}


### Get ready for the hands-on

Before we start, let us ensure we have got the required files to run the tutorials. We'll create a directory on NeSI's scratch "nobackup" filesystem for you to work in. NB: these files will be removed within a day of the tutorial.

```
cd ~
mkdir /nesi/nobackup/nesi99991/${USER}
cd /nesi/nobackup/nesi99991/${USER}
```
{: .bash}

If it does not exist already, download the following Github repo. Then `cd` into it, define a couple of handy variables (see below), and finally `cd` into `demos/02_singularity`:

```
git clone https://github.com/nesi/nzrse-containers
cd nzrse-containers
export NZRSE=$(pwd)
export SIFPATH=$NZRSE/demos/sif
cd demos/02_singularity
```
{: .bash}

We also need to initialise your environment to make Singularity available. NeSI maintains up-to-date versions of Singularity as software modules on Mahuika. Load the latest Singularity and check the version. Note also that the `singularity` command includes extensive self-documentation::

```
module load Singularity
singularity version
singularity help
```
{: .bash}

> ## NZ RSE attendees only: cached images
>
> For the NZ RSE tutorial we have prepared some of the bigger images to be downloaded in a specific directory - `/nesi/nobackup/nesi99991/nzrse-containers/demos/sif/`. Create the following symbolic link to be able to use them. Normally downloading the required images will take up to an hour.
>
> ```
> $ ln -s /nesi/nobackup/nesi99991/nzrse-containers/demos/sif $SIFPATH
> ```
> {: .bash}
>
> One more thing: much of this work will be performed interactively on our Slurm cluster, so we need to request a small allocation:
>
> ```
> $ salloc --job-name="SingularityTutorial" --ntasks=4 --time=4:00:00 --account=nesi99991 --reservation=workshop
> ```
> {: .bash}
> ```
> salloc: Granted job allocation 10179453
> salloc: Waiting for resource configuration
> salloc: Nodes wbn027 are ready for job
> ```
> {: .output}
>
> See NeSI's support docs on [Slurm Interactive Sessions](https://support.nesi.org.nz/hc/en-gb/articles/360001316356) for further info.
{: .callout}


### Singularity: a container engine for HPC

[Singularity](https://sylabs.io/singularity/) is developed and maintained by [Sylabs](https://sylabs.io), and was designed from scratch as a container engine for HPC applications, and this is clearly reflected in some of its main features:

* *unprivileged* runtime: Singularity containers do not require the user to hold root privileges to run;

* *root* privileges are required to build container images: users can build images on their personal laptops or workstations, on the cloud, or via a Remote Build service;

* *integration*, rather than *isolation*, by default: same user as host, current directory bind mounted, communication ports available; as a result, launching a container requires a much simpler syntax than Docker;

* native execution of GPU enabled containers.

This tutorial assumes Singularity version 3.0 or higher. Version **3.3.0** or higher is recommended as it offers a smoother, more bug-free experience.


### Container image formats

One of the differences between Docker and Singularity is the adopted format to store container images.

Docker adopts a layered format compliant with the *Open Containers Initiative* (OCI). Each build command in the recipe file results in the creation of a distinct image layer. These layers are cached during the build process, making them quite useful for development. In fact, repeated build attempts that make use of the same layers will exploit the cache, thus reducing the overall build time. On the other hand, shipping a container image is not straightforward, and requires either relying on a public registry, or compressing the image in a *tar* archive.

Since version 3.0, Singularity has developed the *Singularity Image Format* (*SIF*), a single file layout for container images. Among the benefits, an image is simply a very large file, and thus can be transferred and shipped as any other file. Building on this single file format, a number of features have been developed, such as image signing and verification, and (more recently) image encryption. A drawback of this approach is that during build time a progressive, incremental approach is not possible.

Interestingly, Singularity is able to download and run both types of images.

Note that Singularity versions prior to 3.0 used a slightly different image format, characterised by the extension `.simg`. You can still find these around in the web; newer Singularity versions are still able to run them.


### Executing a simple command in a Singularity container

Running a command is done by means of `singularity exec`:

```
singularity exec library://library/default/ubuntu:18.04 cat /etc/os-release
```
{: .bash}

```
INFO:    Downloading library image

NAME="Ubuntu"
VERSION="18.04 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
```
{: .output}

Here is what Singularity has just done:

* downloaded a Ubuntu image from the Cloud Library (this wouldn't happen if the image had been downloaded previously);
* stored it into the default cache directory;
* instantiated a container from that image;
* executed the command `cat /etc/os-release`.

Container images have a **name** and a **tag**, in this case `ubuntu` and `18.04`. The tag can be omitted, in which case Singularity will default to a tag named `latest`.


> ## Using the *latest* tag
>
> The practice of using the `latest` tag can be handy for quick typing, but is dangerous when it comes to reproducibility of your workflow, as under the hood the *latest* image could change over time.
{: .callout}


The prefix `library://` makes Singularity pull the image from the default registry, that is the [**Sylabs Cloud Library**](https://cloud.sylabs.io). Images in there are organised in terms of **users** (`library` in this case) and **user collections** (optional, `default` in the example above). Note that in the particular case of `library/default/`, this specification could be skipped, for instance:

```
singularity exec library://ubuntu:18.04 echo "Hello World"
```
{: .bash}

```
Hello World
```
{: .output}

Here we are also experiencing image caching in action: the output has no more mention of the image being downloaded.


### Executing a command in a Docker container

Now let's try and download a Ubuntu container from the [**Docker Hub**](https://hub.docker.com), *i.e.* the main registry for Docker containers:

```
singularity exec docker://library/ubuntu:18.04 cat /etc/os-release
```
{: .bash}

```
INFO:    Converting OCI blobs to SIF format
INFO:    Starting build...
Getting image source signatures
Copying blob sha256:22e816666fd6516bccd19765947232debc14a5baf2418b2202fd67b3807b6b91
 25.45 MiB / 25.45 MiB [====================================================] 1s
Copying blob sha256:079b6d2a1e53c648abc48222c63809de745146c2ee8322a1b9e93703318290d6
 34.54 KiB / 34.54 KiB [====================================================] 0s
Copying blob sha256:11048ebae90883c19c9b20f003d5dd2f5bbf5b48556dabf06c8ea5c871c8debe
 849 B / 849 B [============================================================] 0s
Copying blob sha256:c58094023a2e61ef9388e283026c5d6a4b6ff6d10d4f626e866d38f061e79bb9
 162 B / 162 B [============================================================] 0s
Copying config sha256:6cd71496ca4e0cb2f834ca21c9b2110b258e9cdf09be47b54172ebbcf8232d3d
 2.42 KiB / 2.42 KiB [======================================================] 0s
Writing manifest to image destination
Storing signatures
INFO:    Creating SIF file...
INFO:    Build complete: /data/singularity/.singularity/cache/oci-tmp/a7b8b7b33e44b123d7f997bd4d3d0a59fafc63e203d17efedf09ff3f6f516152/ubuntu_18.04.sif

NAME="Ubuntu"
VERSION="18.04.3 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.3 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
```
{: .output}

Rather than just downloading a SIF file, now there's more work for Singularity, as it has to both:

* download the various layers making up the image, and
* assemble them into a single SIF image file.

Note that, to point Singularity to Docker Hub, the prefix `docker://` is required.

Also note how Docker Hub organises images only by users (also called *repositories*), not by collections.


> ## What is the *latest* Ubuntu image from the Sylabs Cloud?
>
> Write down a Singularity command that prints the OS version through the *latest* Ubuntu image from Sylabs Cloud Library.
>
> > ## Solution
> >
> > ```
> > $ singularity exec library://ubuntu cat /etc/os-release
> > ```
> > {: .bash}
> >
> > ```
> > [..]
> > NAME="Ubuntu"
> > VERSION="18.10 (Cosmic Cuttlefish)"
> > [..]
> > ```
> > {: .output}
> >
> > It's version 18.10.
> {: .solution}
{: .challenge}


### Open up an interactive shell

Sometimes it can be useful to open a shell inside a container, rather than to execute commands, *e.g.* to inspect its contents.

Achieve this by using `singularity shell`:

```
singularity shell library://ubuntu:18.04
```
{: .bash}

```
Singularity ubuntu_18.04.sif:/home/ubuntu/nzrse-containers/demos/02_singularity>
```
{: .output}

Remember to type `exit`, or hit `Ctrl-D`, when you're done!


### Download and use images via SIF file names

All examples so far have identified container images using their registry name specification, *e.g.* `library/default/ubuntu:18.04` or similar.

An alternative option to handle images is to download them to a known location, and then refer to their SIF file names.

Let's use `singularity pull` to save the image to a specified path (output might differ depending on the Singularity version you use):

```
singularity pull library://ubuntu:18.04
```
{: .bash}

```
WARNING: Container might not be trusted; run 'singularity verify ubuntu_18.04.sif' to show who signed it
INFO:    Download complete: ubuntu_18.04.sif
```
{: .output}

By default, the image is saved in the current directory:

```
ls
```
{: .bash}

```
ubuntu_18.04.sif
```
{: .output}

Also note the trust warning from when we pulled the container image - do not ignore these! Singularity has features and specific services for signing and verifying the source and integrity of containers. Let's check this image:

```
singularity verify ubuntu_18.04.sif
```
{: .bash}

```
Container is signed by 1 key(s):

Verifying partition: FS:
8883491F4268F173C6E5DC49EDECE4F3F38D871E
[REMOTE]  Sylabs Admin <support@sylabs.io>
[OK]      Data integrity verified

INFO:    Container verified: ubuntu_18.04.sif
```
{: .output}

This tells us that the container was signed by Sylabs Admin and that the integrity of the image is ok, i.e., it hasn't been changed since it was built and signed.

Now that we've checked the image we can use the image file simply by:

```
singularity exec ./ubuntu_18.04.sif echo "Hello World"
```
{: .bash}

```
Hello World
```
{: .output}

You can specify the storage location with:

```
mkdir -p sif_lib
singularity pull --dir sif_lib docker://library/ubuntu:18.04
```
{: .bash}

```
INFO:    Using cached image
```
{: .output}

```
ls sif_lib
```
{: .bash}

```
ubuntu_18.04.sif
```
{: .output}


> ## Organise your local container images
>
> Being able to specify download locations for the container images allows you to keep your local set of images organised and tidy, by making use of a directory tree. It also allows for easy sharing of images within your team in a shared resource.
{: .callout}


### Configure cache and pull directory locations

Lots of Singularity settings can be configured by means of environment variables.

The default directory location for the image cache is `$HOME/.singularity/cache`. This location can be inconvenient in shared resources such as HPC centres, where often the disk quota for the home directory is limited. You can redefine the path to the cache dir by setting the variable `SINGULARITY_CACHEDIR`.

Similarly, if you have a preferred location to pull images into you can avoid using the flag `--dir` at runtime, and instead define the variable `SINGULARITY_PULLFOLDER`.


### Reclaim cache space

If you are running out of disk space, you can inspect the cache with this command (add `-v` from Singularity version 3.4 on):

```
singularity cache list -v
```
{: .bash}

```
NAME                     DATE CREATED           SIZE             TYPE
ubuntu_latest.sif        2019-10-21 13:19:50    28.11 MB         library
ubuntu_18.04.sif         2019-10-21 13:19:04    37.10 MB         library
ubuntu_18.04.sif         2019-10-21 13:19:40    25.89 MB         oci

There 3 containers using: 91.10 MB, 6 oci blob file(s) using 26.73 MB of space.
Total space used: 117.83 MB
```
{: .output}

and then clean it up, *e.g.* to wipe everything use the `-a` flag (use `-f` instead from Singularity version 3.4 on):

```
singularity cache clean -a
```
{: .bash}


> ## Contextual help on Singularity commands
>
> Use `singularity help`, optionally followed by a command name, to print help information on features and options.
{: .callout}


### Popular registries (*aka* image libraries)

At the time of writing, Docker Hub hosts a much wider selection of container images than Sylabs Cloud. This includes Linux distributions, Python and R deployments, as well as a big variety of applications.

Bioinformaticians should keep in mind another container registry, [Quay](https://quay.io) by Red Hat, that hosts thousands of applications in this domain of science. These mostly come out of the [Biocontainers](https://biocontainers.pro) project, that aims to provide automated container builds of all of the packages made available through [Bioconda](https://bioconda.github.io).

Nvidia maintains the [Nvidia GPU Cloud (NGC)](https://ngc.nvidia.com), hosting an increasing number of containerised applications optimised to run on GPUs.


> ## Pull and run a Python container ##
>
> How would you pull the following container image from Docker Hub, `python:3-slim`?
>
> Once you've pulled it, enquire the Python version inside the container by running `python --version`.
>
> > ## Solution
> >
> > Pull:
> >
> > ```
> > $ singularity pull docker://python:3-slim
> > ```
> > {: .bash}
> >
> > Get Python version:
> >
> > ```
> > $ singularity exec ./python_3-slim.sif python --version
> > ```
> > {: .bash}
> >
> > ```
> > Python 3.8.0
> > ```
> > {: .output}
> {: .solution}
{: .challenge}
