# Releases

Every BOSH deployment describes one or more instance groups, upon each is layered one or more job templates, and each job template can had a dependency on packages. The origin of these job templates and packages will become clear now as we introduce BOSH releases, and the top-level deployment manifest attribute `releases`.

Consider this abridged deployment manifest:

```yaml
releases:
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release

instance_groups:
- name: zk
  instances: 5
  jobs:
  - name: zookeeper
    release: zookeeper
  - name: status
    release: zookeeper
```

Each of the five instances in the instance group `zk` will have two job templates installed:

* job template `zookeeper` that is provided by a BOSH release `zookeeper`
* job template `status` that is provided by a BOSH release `zookeeper`

The information about the required BOSH releases is defined in the top-level of the deployment manifest.

## Version Requirement

```yaml hl_lines="3"
releases:
- name: zookeeper
  version: 0.0.7
```

This deployment manifest will require that `zookeeper/0.0.7` has already been installed in your BOSH environment.

You can manually upload a BOSH release using the `bosh upload-release` command:

```
> bosh upload-release https://bosh.io/d/github.com/cppforlife/zookeeper-release?v=0.0.7
```

The URL must resolve to a `tgz` file that was original produced by the `bosh create-release` command. We are currently discussing BOSH releases that were published by other teams, and they used the `bosh create-release` command to do this. Later in the Ultimate Guide to BOSH we too will learn how to create our own BOSH releases.

Approximately, `bosh upload-release https://url/to/someone-else-release-1.2.3.tgz` is similar to using a package manager to install a pre-created package, such as Debian `apt-get install someone-eles-package`. The distinction here is that `bosh upload-release` only populates the external BOSH release into our BOSH environment. We do not use the BOSH release until we run `bosh deploy` and the releases are applied to instances via job templates and packages. So perhaps `bosh upload-release` is more similar to copying an upstream Debian package into your own Debian package cache, and `bosh deploy` is similar to running `apt-get install` from your own package cache.

## External Release

A wonderful feature of a base BOSH deployment manifest can be to include the specific details for `bosh deploy` to discover and install the version that is required. This reduces the additional steps for an operator to bootstrap a new system or upgrade an existing deployment. They can simply run `bosh deploy`. Any missing releases will be automatically fetched and installed into the BOSH environment.

```yaml hl_lines="3 4"
releases:
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release
```

When you run `bosh deploy`, the CLI will see the `git+https` protocol determine that https://github.com/cppforlife/zookeeper-release is a Git repository containing information about all releases. The CLI will look inside the repository for the job templates and packages for the `version: 0.0.7`. The CLI will then download these job templates and packages to your local computer or jumpbox, and then it will upload them to your BOSH environment.

The `bosh deploy` command will use your local `git` configuration and credentials to access Git repositories, thus making it possible to use private Git repositories.

## Finding Existing BOSH Releases

There are many existing BOSH releases that you might like to use, before going down the path of authoring your own BOSH releases later in the Ultimate Guide to BOSH.

One location to discover existing BOSH releases is https://bosh.io/releases.

Another idea is to search Github https://github.com/search?q=bosh+release.

Talk to mysterious strangers at dinner parties.

## What is a BOSH Release?

A BOSH release is a versioned bundle of job templates, and their dependent packages, that describe a complete running system. The developers of a BOSH release benefit by iterating on bespoke application source code, dependency package versions, and configuration file templates all in the one project. The operators of a BOSH release benefit by using a combination of file templates and packages that are exactly the same combination as the developers intended.

We originally encountered [job templates](/instances/#job-templates) and [packages](/instances/#packages) when we were looking inside running BOSH instances. Monit was configured to start and keep processes running. These processes were configured within job templates: how to run the process and how to configure the application within the process. The job templates need software packages to be pre-installed. These packages need to have been pre-compiled from trusted source files.

## Comparing BOSH Releases to Traditional Software Packages

A BOSH release is similar to a collection of traditional software packages, such as Debian APT or MacOS Homebrew packages.

A traditional software package focused on installing a single piece of software. Perhaps it is an executable, perhaps also a library, and some documentation for users and developers. The installed package will have already been compiled into its executable form for the current operating system and underlying host machine architecture.

If this software package has any dependencies, then when you install the software package you will automatically also download and install its dependencies, and their dependencies.

A traditional software package might also configure and run a service. For example, a database package will install the database software, and then it might also install some default configuration files, start the database process using the default process monitoring system, and then initialise the database.

A BOSH release combines all these functions. A BOSH release that is installed into BOSH instances will:

* configure, initialise, and run processes using the Monit process monitoring system
* install all the runtime dependencies via BOSH packages
* all packages will have already been compiled for the target operating system and architecture
* job templates will be aware that there might be 2+ instances that need to form a cluster at runtime

{==

A BOSH release is a single-purpose collection of packages and their job templates.

==}

A traditional database package might configure and run a database, but in reality you would want to manage the specific configuration files for the specific disk volumes, and the other members of the database cluster running on different machines etc.

The `zookeeper` BOSH release exists to run a cluster of Apache ZooKeeper nodes. A traditional software package of Apache ZooKeeper will be authored to install ZooKeeper software.

## Specific Collection of Packages

Each version of the `zookeeper` BOSH release bundles an explicit version of Apache ZooKeeper, like a traditional software package. But a BOSH release also bundles an explicit version of its dependencies. For a `zookeeper` BOSH release, this means it bundles a specific distribution and version of a Java Virtual Machine.

## Caching Upstream Software

A BOSH release contains its own copy of software to ensure the BOSH release can always be re-built. It does not assume that the specific version of Apache ZooKeeper will forever be available for download on demand.

A BOSH release contains its own copy of software in part to isolate BOSH operators from upstream dependencies being hacked. A BOSH release guarantees an operator that the exact same version of source code will be used as the original developers selected.

## Blobs

These copies of explicitly selected software are called "blobs". Blobs guarantee that a BOSH operator, and subsequent developer of a BOSH release, will be downloading and using the same files.

We use the integrity of the BOSH release Git repository to track the blobs used in each BOSH release version.

```
git clone https://github.com/cppforlife/zookeeper-release
git checkout v0.0.7
cd zookeeper-release
cat config/blobs.yml
```

At the time of writing, the blobs used in v0.0.7 of https://github.com/cppforlife/zookeeper-release were:

```yaml hl_lines="1 5"
java/jdk-1.7.0_51.tar.gz:
  size: 138199690
  object_id: 8a697fac-6cc4-443f-7009-b1c62e3d22b8
  sha: bee3b085a90439c833ce18e138c9f1a615152891
zookeeper/zookeeper-3.4.10.tar.gz:
  size: 35042811
  object_id: 868354bd-c49e-41b7-49a9-b110091a82af
  sha: eb2145498c5f7a0d23650d3e0102318363206fba
```

We can see that v0.0.7 of the BOSH release contains upstream versions:

* Apache ZooKeeper v3.4.10
* Java JDK v1.7.0_51

## Origins of Blobs

One omission from BOSH release specifications is that there is no declaration of the original location from where each blob was sourced. Consider the two blobs in the example above:

* `zookeeper/zookeeper-3.4.10.tar.gz` - was this the original v3.4.10 release from the Apache download website?
* `java/jdk-1.7.0_51.tar.gz` - is this Oracle Java or OpenJDK? Were any modifications made from the original downloaded files?

Ideally, a BOSH release blobs will be the original files from their original locations, and the authors will document from where these blobs are being sourced. If patches need to be applied to these projects, then this can be done during the compilation phase of a package.

If your BOSH release does not document any metadata about the origins of each blob, then can always download the blobs and look inside them.

To fetch all the blobs referenced in the current `config/blobs.yml` file:

```
bosh sync-blobs
```

The files referenced in `config/blobs.yml` will be downloaded into the `blobs` folder:

```
blobs
├── java
│   └── jdk-1.7.0_51.tar.gz
└── zookeeper
    └── zookeeper-3.4.8.tar.gz
```

To discover the nature of `blobs/java/java-1.7.0_51.tar.gz` we can use the `tar` application to look at its files. Its `LICENSE` file gives us the biggest clue that it is Oracle Java:

```
Please refer to http://java.com/license
```

Whilst a BOSH release might not retain the metadata about the origins of its blobs, it does guarantee that all developers and operators of a BOSH release will be using the same blobs.

## Compiling Packages
