# Setting up a CI server for a GitHub Action runner Singularity from Linux Ubuntu

After following this guide, you'll have a simple GitHub action workflow on a GitHub repository of your choice. When new commits are made to your repository, the workflow delegates work to a server which runs in a [Singlularity](https://sylabs.io/singularity/) container. You can use this Singularity container to on a HPC cluster as a regular user and you will not need root permissions.

This guide distinguishes between the _client_ and the _server_; the client is your own machine; the server is whichever
machine will run the tests. This document describes the case where the server is a Singularity container running on your own machine.

For guides on how to configure other features in addition to just the runner, go [here](/README.md).

## Prerequisites

1. Install Singularity: https://sylabs.io/guides/3.5/user-guide/quick_start.html#quick-installation-steps

2. Follow a [short tutorial](https://sylabs.io/guides/3.5/user-guide/quick_start.html#overview-of-the-singularity-interface) (Optional)

### Testing your Singularity setup

Check Singularity version
```shell
> singularity version

3.5.3
```

Pull an example image

```shell
singularity pull library://sylabsed/examples/lolcow
```

Start a shell in the Singlarity container

```shell
singularity shell lolcow_latest.sif
```

Run some test commands

```shell
Singularity> id

uid=1000(fdiblen) gid=985(users) groups=985(users),98(power),108(vboxusers),972(docker),988(storage),998(wheel)

Singularity> hostname

archlinux
```

## Server side configuration

### Build image

Now we are ready to build our Singularity image. The following command will use [Definition file](github-actions-runner-singularity.def) to build the image. It will create a system install necessary system packages and dependencies for the runner. In order to create a Singularity image, you will need root permission (or sudo) on your system.

```shell
sudo singularity build github-actions-runner-singularity.sif github-actions-runner-singularity.def
```

This command will generate ``github-actions-runner-singularity.sif`` (SIF stands for Singularity Image Format) image which we will use to set up the runner.

## Client side configuration

### Generate an OAuth token

We're almost ready to use our Docker image to set up a GitHub Runner, but first we need to
generate an OAuth token, as follows:

1. Go to [https://github.com/settings/tokens](https://github.com/settings/tokens) and click the ``Generate new token`` button.
2. Provide your GitHub password when prompted
3. Fill in a description for the token, for example _GitHub runner for github.com/&lt;your organization&gt;/&lt;your repository&gt;_
4. Enable the ``repo`` scope and all of its checkboxes, like so:

    ![Token permissions](/images/token_permissions.png)

5. Click ``Generate`` at the bottom. Make sure to copy its value because we'll need it in the next step

### Run the server

#### Preperation
Before using the Singularity image we need to set some environment variables. The Singularity container will use these environment variables to set up the runner.

```shell
export SINGULARITYENV_PERSONAL_ACCESS_TOKEN="<Github OAuth token>"
export SINGULARITYENV_RUNNER_NAME="<runner name to appear on Github>"
export SINGULARITYENV_RUNNER_WORKDIR="/tmp/actions-runner-repo"
export SINGULARITYENV_GITHUB_ORG="<organization or username>"
export SINGULARITYENV_GITHUB_REPO="<name of the repository>"
```

Create an envionment file which will be user by the runner to save some variables.

```shell
cp env.template env
```

#### Instance mode

Alternatively, you can start it as an instance (service).

```shell
singularity instance start github-actions-runner-singularity.sif github-actions-runner --writable-tmpfs --bind ./env:/opt/actions-runner/.env
```

For more information about Singularity services see [this link](https://sylabs.io/guides/3.5/user-guide/running_services.html).


To list the running instances:

```shell
singularity instance list
```

To stop the running Singularity instance:

```shell
singularity instance stop github-actions-runner
```

To start the Singularity instance again:

```shell
singularity instance start github-actions-runner
```

#### Temporary mode

Now we can run Singularity container with the following command.

```shell
singularity run \
    --writable-tmpfs \
    --bind ./env:/opt/actions-runner/.env \
    github-actions-runner-singularity.sif
```

Singularity containers by-default starts in ``read-only`` mode so you cannot make changes. While setting up the runner, some scripts needs to create a few files so we need a write access. This is achieved by adding ``--writable-tmpfs`` argument.

If you stop the running container or interrupt it by pressing to ``CTRL+C``, the Github actions runner will stop and it will be unregistered from your Github repository.

#### Accessing the logs

The singularity instances save the logs in
`~/.singularity/instances/logs/INSTANCE_NAME` folder.

## Using on a HPC Cluster

In most of the cases, the HPC user does not have root persmissions s it wont be possible to build the Singularity image. The Singularity image can be built locally and copied to
the cluster. We will use `scp` command to copy the singularity image.

```shell
scp github-actions-runner-singularity.sif USERNAME@CLUSTER_IP_ADDRESS:$REMOTE_FOLDER
```

The `$REMOTE_FOLDER` is typically your home folder which you can figure out using the command below.

```shell
echo $HOME
```

In oder to use the Singularity image on a HPC cluster, we first need to create a jobscript. The job script will be handled by the scheduler and eventually it will set up the runner when it is executed.


Example:
```
#!/bin/bash
#SBATCH -t 1:00:00
#SBATCH -n 480

cd $HOME/work

export SINGULARITYENV_PERSONAL_ACCESS_TOKEN="<Github OAuth token>"
export SINGULARITYENV_RUNNER_NAME="<runner name to appear on Github>"
export SINGULARITYENV_RUNNER_WORKDIR="/tmp/actions-runner-repo"
export SINGULARITYENV_GITHUB_ORG="<organization or username>"
export SINGULARITYENV_GITHUB_REPO="<name of the repository>"

srun singularity run \
    --writable-tmpfs \
    github-actions-runner-singularity.sif
```

To submit the job script using ``sbatch``:

```shell
sbatch jobscript
```

## GPU support

See [Singularity documentation](https://sylabs.io/guides/3.5/user-guide/gpu.html)

## Limitations

GitHub owned servers have some pre-installed software (see [here](https://docs.github.com/en/actions/reference/software-installed-on-github-hosted-runners)). Self hosted runners are able to download and use these software. However, this is only possible if self-hosted runner is being run on one of the supported platforms such as Ubuntu. Otherwise, users have to install the required software by themselves.
Also actions based on Docker images will not work as running Docker containers inside a Singularity container does not work.

### Extras

#### Get Singularity image details

Use `singularity inspect` to display details of the image

```shell
> singularity inspect github-actions-runner-singularity.sif*
WARNING: No SIF metadata partition, searching in container...
org.label-schema.build-date: Monday_6_July_2020_16:55:58_CEST
org.label-schema.schema-version: 1.0
org.label-schema.usage.singularity.deffile.bootstrap: library
org.label-schema.usage.singularity.deffile.from: ubuntu:19.10
org.label-schema.usage.singularity.deffile.mirrorurl: http://us.archive.ubuntu.com/ubuntu/
org.label-schema.usage.singularity.deffile.osversion: eoan
org.label-schema.usage.singularity.deffile.stage: build
org.label-schema.usage.singularity.version: 3.5.3
```

#### Accessing Singularity container

If you need access to a shell on the running Singularity container:

```shell
singularity shell \
    --writable-tmpfs \
    github-actions-runner-singularity.sif
```

### What's next

Find instructions for provisioning additional functionality [here](../README.md).
