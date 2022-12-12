# Cloud User Meeting: BiBiGrid Hands-on

This tutorial is a reworked/optimized version of the Hands-on session of the **GCB 2019 in Heidelberg**
based on the latest release of BiBiGrid

## Prerequisites

- System base on Linux or OSX or Windows Subsystem for Linux
- required software packages 
  - Python > 3.6  
  - git (required)
  - openssh 
- Openstack API access 
- [OpenstackClient](https://pypi.org/project/python-openstackclient/) (recommended)

## Download BiBiGrid
```shell
git clone https://gitlab.ub.uni-bielefeld.de/bibiserv/bibigrid/bibigrid2.git
cd bibigrid2
```

The following steps assume that you are inside of the bibigrid2 folder. It should contain:

```shell
$ ls
bibigrid2    bibigrid.yml   README.md         resources
bibigrid.sh  documentation  requirements.txt  tests
```

## What will happen...

The goal of this session is to set up a small HPC cluster consisting of 3 nodes (1 master, 2 workers) using BiBiGrid with [Slurm](https://slurm.schedmd.com/quickstart.html) (workload manager), [Network File System](https://linux.die.net/man/5/nfs) (allows file sharing between servers) and [Theia](https://theia-ide.org/docs/user_getting_started/) (Web IDE). This tutorial targets users running BiBiGrid on de.NBI cloud.

1. [Preparation](#preparation)
2. [Configuration](#configuration)
3. [The Cluster](#the-cluster)

## Preparation

### Premade Template

Use the prefilled configuration template [bibigrid_clum2022/resources/bibigrid.yml](resources/bibigrid.yml) as a basis for your personal BiBiGrid configuration. 
Later in this tutorial you will use [OpenStackClient](https://pypi.org/project/python-openstackclient/) or access 
Openstack's dashboard manually to get all necessary configuration information from your project.

Copy the configuration template to `~/.config/bibigrid/`.

### Authentication

In this section you will create an [application credential](https://access.redhat.com/documentation/zh-cn/red_hat_openstack_platform/14/html/users_and_identity_management_guide/application_credentials) and download the autogenerated `clouds.yaml`. `clouds.yaml` contains all required authentication information. Follow the images:

![Navigation](images/ac_screen1.png)

Don't use the input field secret. As you can see its input is not hidden. OpenStack will generate a strong secret for you, if you leave it blank. Pick a sensible expiration date.

![Creation](images/ac_screen2.png)

Safe the downloaded `clouds.yaml` under `~/.config/openstack/` **and** `~/.config/bibigrid/`. That will allow both `OpenstackClient` and BiBiGrid to access it.

<details>
<summary>Why not store BiBiGrids `clouds.yaml` in openstack and save the extra copy?</summary>

In the future BiBiGrid will support more than just one cloud infrastructure. Therefore, using the `~/.config/openstack` folder would be a disadvantage later.
</details>

![Download](images/ac_screen3.png)

If you have `OpenstackClient` installed and `openstack subnet list --os-cloud=openstack` runs without error, you are ready to proceed.

### Virtual Environment

A virtual environment is something that gives you everything you need to run specific programs without altering your installation.

#### Creating a [Virtual Environment](https://docs.python.org/3/library/venv.html)

`python3 -m venv ~/.venv/bibigrid`

#### Sourcing Environments

In order to actually use the virtual environment we need to [source](https://www.theunixschool.com/2012/04/what-is-sourcing-file.html) that environment:

`source ~/.venv/bibigrid/bin/activate`

Following [pip](https://manpages.ubuntu.com/manpages/bionic/en/man1/pip.1.html) installations will only affect the virtual environment. The virtual environment is only `sourced` in the terminal where you executed the source command. Other terminals are not affected.

#### Fulfilling Requirements
You will now install packages required by BiBiGrid within your newly created virtual environment. If you haven't `sourced` your environment yet, please go [back](#sourcing-environments). In order to install all BiBiGrid requirements we simply install from the given requirements file:

`pip install -r requirements.txt`

## Configuration

### Access information

BiBiGrid must know which cloud authentication information to use and later as which user it can access the servers. Therefore, you need to set three keys: [region](https://docs.openstack.org/python-openstackclient/rocky/cli/command-objects/region.html), [availabilityZone](https://docs.openstack.org/nova/latest/admin/availability-zones.html) and [sshUser](https://www.redhat.com/sysadmin/access-remote-systems-ssh). Following the next steps you will be able to update the [premade template](#premade-template).

<details>
<summary>Why are the keys in the template already set?</summary>

Making mistakes while filling the template keys can be annoying at times. We therefore decided to prefill some keys for you, so you can directly see whether the key you found is correct.
</details>

#### region

Determine the [region](https://docs.openstack.org/python-openstackclient/rocky/cli/command-objects/region.html) by running:

```
openstack region list --os-cloud=openstack
```

Set the template's `region` key to the result's `Region` entry.

#### availabilityZone

Determine your [availabilityZone](https://docs.openstack.org/nova/latest/admin/availability-zones.html) by running:

```
openstack availability zone list --os-cloud=openstack
```

If multiple zones are shown, pick default. Set the template's `availabilityZone` key to the result's `Zone Name` entry.

#### sshUser

The [sshUser](https://www.redhat.com/sysadmin/access-remote-systems-ssh) depends on your server image. Since we run on top of Ubuntu 22.04 the ssh-user is `ubuntu`. Set the template's `sshUser` key to `ubuntu`.

### Network

We created a subnet for this workshop. Determine your subnet's `Name` by running:

```
openstack subnet list --os-cloud=openstack
```

Set the template's `subnet` key to the result's `Name` key.

### Instances

BiBiGrid needs to know `type` and `image` for each server. Since those are often identical for the workers, 
you can simply use the `count` key to indicate multiple workers with the same `type` and `image`.

#### Image
Images are virtual disks with a bootable operating system. Choosing an image means choosing the operating 
system of your server.

Since [images](https://docs.openstack.org/image-guide/introduction.html) are often updated, you need to 
look up the current active image using:

```shell
openstack image list --os-cloud=openstack | grep active
```

Since we will use Ubuntu 22.04 you might as well use:

```shell
openstack image list --os-cloud=openstack | grep active | grep "Ubuntu 22.04"
```

Set the template's `image` key of all instances to the result's `ID`  **or** `NAME` entry of the Ubuntu 22.04 row. 
All servers will share the same image.

#### Flavor

Flavors are available hardware configurations.

The following gives you a list of all flavors:

```shell
openstack flavor list --os-cloud=openstack
```

Set the template's `flavor` keys  (`ID` or `NAME`) to flavors of your choice. You can use a different flavor 
for the master and each worker-group.

#### master

```shell
masterInstance:
  type: de.NBI default
  image: [ubuntu-22.04-image-name] or [ubuntu-22.04-image-id]
```

#### worker
```shell
workerInstances:
  - type: de.NBI tiny
    image: [ubuntu-22.04-image-name] or [ubuntu-22.04-image-id]
    count: 2
```

The key `workerInstances` expects a list. Each list element is a `worker group` with an `image` + `type` combination and a `count`.
```shell
workerInstances:
  - type: de.NBI tiny
    image: [ubuntu-22.04-image-name] or [ubuntu-22.04-image-id]
    count: 1
  - type: de.NBI default
    image: [ubuntu-22.04-image-name] or [ubuntu-22.04-image-id]
    count: 1
```

### Waiting for post-launch Service

Some clouds run one or more post-launch services on every started instance, to finish the initialization after an 
instance is available (e.g. to configure local proxy settings or local available repositories). That might interrupt 
Ansible. Therefore, BiBiGrid needs to wait for your post-launch service(s) to finish. For that BiBiGrid needs the 
services' names. Set the key `waitForServices` to the list of services you would like to wait for. For Bielefeld 
this would be `de.NBI_Bielefeld_environment.service`. You should be able to find post-launch service names by 
taking a look at your location's [Computer Center Specific](https://cloud.denbi.de/wiki/) site - if 
post-launch services exist for your location.

```yaml
  waitForServices: 
    - de.NBI_Bielefeld_environment.service
```


### Check Your Configuration
Run `./bibigrid.sh -i [path-to-bibigrid.yml] -ch -v` to check your configuration. `path-to-bibigrid.yml` is 
`bibigrid.yml` if you copied the configuration template to `~/.config/bibigrid/`. The command line argument 
`-v` allows for greater verbosity which will make it easier for you to fix issues.

## The Cluster
`./bibigrid.sh -i bibigrid.yml -c -v` creates the cluster with a more verbose output. Cluster creation time 
depends on the chosen flavor and the overall load of the cloud and will take up to 15 minutes.

### List Running Cluster
Since several clusters can be running simultaneously, it can be useful to list all running clusters:

Execute `./bibigrid.sh -i bibigrid.yml -l`. You will receive a general overview over all clusters started 
in your project.

### Cluster SSH Connection

After a successful setup, BiBiGrid will print some information. For example:

```
Cluster 6jh83w0n3vsip90 with master 123.45.67.890 up and running!
SSH: ssh -i '~/.bibigrid/tempKey_bibi-6jh83w0n3vsip90' ubuntu@123.45.67.890
Terminate cluster: ./bibigrid.sh -i 'bibigrid.yml' -t -cid 6jh83w0n3vsip90
Detailed cluster info: ./bibigrid.sh -i 'bibigrid.yml' -l -cid 6jh83w0n3vsip90
```

You can now establish an SSH connection to your cluster's master by executing the `SSH` line of your `create`'s 
output: 
```shell
ssh -i '~/.bibigrid/tempKey_bibi-6jh83w0n3vsip90' ubuntu@123.45.67.890 
```
But make sure to use the one generated for you by BiBiGrid since 

- cluster-id (here `6jh83w0n3vsip90`), 
- key name (here `~/.config/bibigrid/keys/tempKey_bibi-6jh83w0n3vsip90`) 
- user@IP (here `ubuntu@123.45.67.890`) 

can (and most likely will) differ on every run. Run `sinfo` after logging in. You will see only the master in 
Slurm's list. That is correct. You have successfully logged in.

However, doing everything on the running cluster from a terminal can be quite bothersome. That's were Theia comes in.

### Using Theia Web IDE

[Theia Web IDE's](https://www.theia-ide.org/) many features make it easier to work on your cloud instances. Take a look:

![Theia](images/theia.png)


When enabled, Theia Web IDE is configured to listen on localhost port 8181 on the master instance. Since this address 
is not directly available you have to forward it to your machine using ssh. 

Execute 

```shell
./bibigrid.sh -i bibigrid.yml -ide -cid [cluster-id]
```

to connect to Theia. You may even use `./bibigrid.sh -i bibigrid.yml -ide` since BiBiGrid will attempt to connect to your last created cluster if no cluster-id is given. Theia will be run as `systemd service` on localhost. A Theia IDE tab will be opened in your browser.

#### Hello World, Hello BiBiGrid!

<details>
<summary>Degression: Job Scheduling (Slurm)</summary>

[Slurm](https://slurm.schedmd.com/) is used for job scheduling/workload management. To see all nodes in your cluster execute `sinfo`. You will notice that workers are `idle~`. That means they are `idle` and `~` (powered down). Slurm uses many symbols and words to indicate node states. See [here](https://slurm.schedmd.com/sinfo.html#SECTION_NODE-STATE-CODES) for more about that. To see all running jobs, execute `squeue`. You will notice that no job is currently running.
</details>

After successfully connecting to Theia IDE, we will now run our first job on our cluster. Let's start with a "hello world".

- Open a terminal

- Create a new shell script `nano /vol/spool/helloworld.sh`:

```shell
#!/bin/bash
echo Hello from $(hostname) !
sleep 10
```

- Make `helloworld.sh` executable using [chmod](https://linux.die.net/man/1/chmod): `chmod u+x /vol/spool/helloworld.sh`
- Submit this script as an array job 50 times : `sbatch --array=1-50 --job-name=helloworld /vol/spool/helloworld.sh` (run the job 50 times). The job `helloworld` runs now. It will take a while to finish, but you can already inspect some information while it runs.
- The master will now power up worker nodes (as you described it in `bibigrid.yml`) to assist him with this job. Execute `sinfo` after a few seconds to see the current node status.
- View information about all scheduled jobs by executing `squeue`. You will see your job `helloworld` there.
- You can see `helloworld`'s output using [cat](https://linux.die.net/man/1/cat) `cat /vol/spool/slurm-*.out`.

## Terminate a cluster

Terminating a running cluster is quite simple. Execute `./bibigrid.sh -i bibigrid.yml -t -cid [cluster-id]`. 
You probably already guessed it, `./bibigrid.sh -i bibigrid.yml -t` also does the trick, since BiBiGrid will fall 
back on your last created cluster if no cluster-id is specified.

## Moving Forward: Further Explorations

Congratulations! You have finished BiBiGrid's Hands-on.

You may want to take a look at the "real" `bibigrid.yml` inside BiBiGrid's repository. It has a few more keys. Howeve, everything you learned here stays true.

If you would like to deepen your knowledge maybe give BiBiGrid's [Features](https://gitlab.ub.uni-bielefeld.de/bibiserv/bibigrid/bibigrid2/-/blob/main/documentation/markdown/bibigrid_feature_list.md) or the [Software](https://gitlab.ub.uni-bielefeld.de/bibiserv/bibigrid/bibigrid2/-/blob/main/documentation/markdown/bibigrid_software_list.md) used by BiBiGrid a read.

### I have an issue

Issues can be created [here](https://gitlab.ub.uni-bielefeld.de/bibiserv/bibigrid/bibigrid2/-/issues). However, GitLab only allows users to create issues and since we are running GitLab on `bielefeld.de`, you would need to create a guest account. This is of course not a viable long-term solution and we are actively working on solving [this issue](https://gitlab.ub.uni-bielefeld.de/bibiserv/bibigrid/bibigrid2/-/issues/46).