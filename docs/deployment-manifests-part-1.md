# Deployment Manifests, Part 1

To provision a new deployment, we provide a deployment manifest to the BOSH director. To make modifications to an existing deployment, we provide the BOSH director with a modified deployment manifest.

A deployment manifest is the explicit declaration of what software needs to run, with specific configuration properties, on each different instance.

**The same deployment manifest deployed today should produce the same running system if you deployed it again in five years time.**

Our first example deployment manifest will be `zookeeper.yml`, which I've been referring to throughout the Ultimate Guide to BOSH so far (the [original file](https://github.com/cppforlife/zookeeper-release/blob/207c9d79eb12399dffe6df7f89abd854d4888f3e/manifests/zookeeper.yml) at time of writing). Below is a subset of the manifest that references the concepts covered so far:

```yaml
---
name: zookeeper

releases:
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release

instance_groups:
- name: zookeeper
  instances: 5
  jobs:
  - name: zookeeper
    release: zookeeper
  - name: status
    release: zookeeper
  persistent_disk: 10240

- name: smoke-tests
  lifecycle: errand
  jobs:
  - name: smoke-tests
    release: zookeeper
```

BOSH deployment manifests use [YAML](http://yaml.org/) markup language. YAML is a relatively human-readable format though your friendship with YAML will be stretched when deployment manifests grow beyond a 100 lines or more.

The goal of a deployment manifest is to describe a deployment that will be reproducible again and again in the future. Three years from now we do not want to accidentally see different packages or configuration files being installed unless we explicit upgraded to them.

In the example manifest above, the top level sections of this YAML file are:

* `name` is the unique name of this deployment within a BOSH director. When a BOSH director receives subsequent deployment manifests with the same `name` it will assume it is an upgrade of the existing deployment.
* `releases` lists the specific BOSH release versions that are to be used, which almost means the specific sets of job templates and packages.
* `instance_groups` lists the sets of instances that will run the same job templates/packages as each other. Instance groups will be deployed as long running instances by default. The configuration `lifecycle: errand` means they will instead be errands (to be discussed later).

## Sizing a Deployment, Part 1

In our example subset manifest above, there are two attributes we can change to resize the infrastructure used for our deployment:

* `instances: 5` will set or change the number of instances used
* `persistent_disk: 10240` will set or change the size of the persistent volume attached to each instance

For a brand new deployment, setting these attributes and invoking `bosh deploy` will establish the initial CPI requests to the cloud infrastructure.

For an existing deployment, changing these attributes and invoking `bosh deploy` will perform a transformation of your system from its current cloud infrastructure servers and disks to the new requirements - more or fewer servers, and/or smaller or large persistent disks.

We will review how the BOSH director performs these transformations of existing deployments later. It's pretty fabulous.

## Explicit Declaration in Manifests

If we keep this manifest the same, we will always get the same deployment of instances, job templates, and packages year after year. This is achieved by our explicit declaration of `releases`.

In the example above, we explicitly require `zookeeper/0.0.7` BOSH release. The `0.0.7` version number is only relative to preceding versions of the same BOSH release, not to any upstream packages. At the time of writing, the `zookeeper` BOSH release being used was packaging Apache ZooKeeper v3.4.10 ([list of source blobs](https://github.com/cppforlife/zookeeper-release/blob/207c9d79eb12399dffe6df7f89abd854d4888f3e/config/blobs.yml)).

If you deployed `zookeeper/0.0.7` every day for a year, you would always be deploying Apache ZooKeeper v3.4.10 as it is packaged inside the BOSH release. It would always use the same Monit control script, the same Monit start/stop wrapper script, and the same configuration templates.

In the deployment manifest, we decide which instances install which job templates/packages within the `instance_groups` section.

The `zookeeper.yml` example above specifies as single long-running group of instances:

```yaml
instance_groups:
- name: zookeeper
  instances: 5
  jobs:
  - name: zookeeper
    release: zookeeper
  persistent_disk: 10240
```


This group of instances will be known within the deployment by its name `zookeeper`. This instance group name is coincidentally the same name as the entire deployment. Whilst, deployment names are unique across deployments within the same BOSH director, instance group names only need to be unique within a deployment. You can have many different deployments in one BOSH director with an instance group called `zookeeper`, but they would all need different deployment names.

Each instance in the group will have a 10GB persistent disk (the plain number `10240` is in MB).

The software, configuration, and start/stop scripts running on each `zookeeper` instance is described by the `jobs:` section of an instance group.

```yaml
jobs:
- name: zookeeper
  release: zookeeper
  properties: {}
```

This `jobs` section declares that it will install a job template called `zookeeper` from the release named `zookeeper`. The consistency of naming the deployment, instance group, release, and job templates all `zookeeper` is mentally efficient eventually.

But right now, you might find it confusing.

## Immutable Manifest Attributes

Let's rename as many attributes in this manifest as we can and discuss which attributes we cannot modify.

```yaml
---
name: zookeeper-deployment

releases:
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release

instance_groups:
- name: zookeeper-instances
  instances: 3
  jobs:
  - name: zookeeper
    release: zookeeper
  - name: status
    release: zookeeper
  persistent_disk: 20480

- name: smoke-tests-errand
  lifecycle: errand
  jobs:
  - name: smoke-tests
    release: zookeeper
```

Compare this manifest to the earlier version and see that we have modified the following attributes:

* deployment name changed to `zookeeper-deployment`
* instance group `zookeeper` renamed to `zookeeper-instances`
* we've reduced the cluster size from 5 instances to 3
* we've resized the persistent disk from 10GB to 20GB
* errand `smoke-tests` renamed to `smoke-tests-errand`

These were the only attributes in our manifest subset that were easily modifiable or renameable.

Conversely, the following attributes of the manifest were not easily modifiable:

* the `name` of each item in the `releases` section comes from the BOSH release we use; we cannot rename or alias them within the manifest
* the `version` and `url` are related to each other to describe which BOSH release to use; if we want to upgrade to a newer release we would change these
* within `jobs` sections of instance groups the `release` must match one of the names in the top-level `releases` section. At the top-level these are immutable, so they are correspondingly immutable within the `jobs` sections of instance groups.
* within `jobs` sections of instance groups, the `name` of a job template is derived from the BOSH release itself (more on this soon), and cannot be renamed or aliased within a deployment manifest.


## Cloud Config, Part 1

The number of instances and the size of persistent disks are attributes that are not specific to which cloud infrastructure you are using. Conversely, the specific details about the size of the servers is specific to each cloud infrastructure. On vSphere, you will want to specify the explicit allocation of CPUs, RAM, and ephemeral disk. Whereas, when using Amazon EC2 you might choose between a list of [Instance Types](https://aws.amazon.com/ec2/instance-types/) such as `m4.large` or `t2.medium`. Similarly, GCP [Machine Types](https://cloud.google.com/compute/docs/machine-types) have a list of predefined sizes you can provision, albeit with a different set of names such as `n1-standard-1` and different attributes from Amazon EC2.

This crossover from the deployment manifest to specific cloud infrastructure requirements is not placed in the deployment manifest. Instead, we will have provided our BOSH director with "Cloud Config" to educate it about how we wish our deployment manifests to be applied to our cloud infrastructure.

Each BOSH director has a Cloud Config specification. We can download and view a BOSH director's Cloud Config:

```
bosh cloud-config
```

There are several areas of configuration but for now let's look at one - `vm_type` - the specification of the size of servers. An example of a `bosh cloud-config` for GCP might look like:

```yaml
vm_types:
- name: default
  cloud_properties:
    machine_type: n1-standard-2
    root_disk_size_gb: 20
    root_disk_type: pd-ssd
```

For Amazon EC2 it might include:

```yaml
vm_types:
- name: default
  cloud_properties:
    instance_type: t2.medium
    ephemeral_disk:
      size: 20_000
```

For vSphere it might include:

```yaml
vm_types:
- name: default
  cloud_properties:
    cpu: 2
    ram: 4096
    disk: 20_000
```

Cloud Config is where BOSH becomes specific to each different cloud infrastructure. Each `vm_type` above includes configuration that is only meaningful to its CPI. Their configurations are similar (2 CPUs) but different (Google `n1-standard-2` has 7.5GB RAM, AWS `t2.medium` has 4GB RAM).

The CPI-specific customisations are placed in the `cloud_properties` section of each `vm_type`.

It is likely that you will want to use various `vm_types` for your deployments. You might want small instance sizes for low CPU/low RAM activities, and large CPU/large RAM instances for other activities. Each will need an entry in your shared `vm_types` list within `bosh cloud-config`.

In our example `zookeeper.yml` we can now add a required `vm_type` attribute into each of the `instance_groups` to reference which instance size we want to use from our `cloud-config`:

```yaml
---
name: zookeeper-deployment

instance_groups:
- name: zookeeper-instances
  instances: 5
  vm_type: default
  jobs:
  - name: zookeeper
    release: zookeeper
  persistent_disk: 10240

- name: smoke-tests-errand
  instances: 1
  lifecycle: errand
  vm_type: default
  jobs:
  - name: smoke-tests
    release: zookeeper
```

The name `default` is not particularly meaningful. In the deployment manifest above there is no indication of the amount of resources each `zookeeper` cloud server will be allocated on the target cloud infrastructure. 1GB RAM or 16GB? How many CPUs?

As a counter example, the community method for deploying Cloud Foundry assumes that your `cloud-config` contains multiple `vm_types` that are named ([see `cf-deployment.yml`](https://github.com/cloudfoundry/cf-deployment/blob/master/cf-deployment.yml)):

* `minimal`
* `small`
* `small-highmem`
* `sharedcpu`

These are more descriptive than `default` but you would still need to investigate your `bosh cloud-config` to see the specific details.

## Instance Groups Form Clusters

Without knowing how Apache ZooKeeper works, it is fair to assume that the ZooKeeper processes running on each of the five instances in our example deployment are communicating with each other. Yet, in our example deployment manifests, we have not explicitly described any relationships between them.

This is not by omission. BOSH deployment manifests allow you to ignore explicit networking configuration as much as possible. Instead, the mapping of your Cloud Infrastructure networking to BOSH is configured in the `bosh cloud-config`.

In the subsequent section [Networking](/networking), I will introduce computer networking and reduce it to the parts you will need to know to help BOSH to help you deploy, scale, upgrade your distributed systems.

But first, let's look at how each ZooKeeper process is configured to know where its cluster peers are located.

In the section [Job Templates](/instances/#job-templates), we discussed that all files for running and configuring processes are inside the `/var/vcap/jobs` subfolders. Each subfolder is a job template provided by a BOSH release.

Another look within the `zookeeper` job template on a `zookeeper` deployment instance:

```
> bosh ssh zookeeper/0
$ cd /var/vcap/jobs/zookeeper
$ tree
.
├── bin
│   ├── ctl
│   └── pre-start
├── config
│   ├── configuration.xsl
│   ├── log4j.properties
│   ├── myid
│   └── zoo.cfg
├── monit
└── packages
    ├── java
    └── zookeeper
```

To recap, `/var/vcap/jobs/zookeeper/monit` describes how to start/stop `zookeeper` and what process ID (PID) to watch to ensure that ZooKeeper is still running. Monit will invoke `/var/vcap/jobs/zookeeper/bin/ctl start` to start or restart the local `zookeeper` process.

An abridged version of `/var/vcap/jobs/zookeeper/bin/ctl` to start ZooKeeper looks like:

```bash
export ZOOCFGDIR=/var/vcap/jobs/zookeeper/config
exec chpst -u vcap:vcap \
  /var/vcap/packages/zookeeper/bin/zkServer.sh start-foreground
```

The `$ZOOCFGDIR` environment variable is special to the `/var/vcap/packages/zookeeper/bin/zkServer.sh` script. This script will look for `zoo.cfg` in the `$ZOOCFGDIR` folder. Notice that `$ZOOCFGDIR` is a folder within the job template above: `config/zoo.cfg`.

The contents of `/var/vcap/jobs/zookeeper/config/zoo.cfg` include the following configuration that allows the `zookeeper/0` instance to discover the other four instances:

```
server.0=10.0.0.5:2888:3888
server.1=10.0.0.6:2888:3888
server.2=10.0.0.7:2888:3888
server.3=10.0.0.8:2888:3888
server.4=10.0.0.9:2888:3888

clientPort=2181
```

At a glance, Apache ZooKeeper will expect to communicate with its peer nodes on ports `2888` and `3888`, and the five cloud servers running the zookeeper processes have IP addresses `10.0.0.5` thru `10.0.0.9`. Client applications that want to use our Apache ZooKeeper system will communicate via port `2181`.

This `config/zoo.cfg` configuration is meaningful to Apache ZooKeeper and the `zkServer.sh` start script. The contents of the configuration file are not meaningful to BOSH, but the file was created by BOSH.

The deployment manifest `zookeeper.yml` did not need to explicitly allocate the five IP addresses above (also found by running `bosh instances`), nor did the manifest need to explicitly document them for the `zookeeper` job template. Instead, all the IP addresses for all members of the `zookeeper` instance group were automatically provided to the `zookeeper` job template before `monit` attempted to start any processes. We will look at how to write your own job templates in your own BOSH releases in later sections.

The more urgent piece of information you will want to know why were these five `zookeeper` instances allocated those five IP addresses, why are the five underlying cloud servers allowed to talk to each other, is anything special required for them to communicate over ports `2888` and `3888` but prevent other systems from accessing these ports, and how are client applications allowed to access these five cloud servers over port `2181`.

And if you're confused at all by that last paragraph, then you are ready for the next section.
