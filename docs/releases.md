# Releases

Every BOSH deployment manifest describes a set of cloud servers, their disks, and the software to be installed, configured, and run.

Stated again using BOSH terminology, a BOSH deployment manifest describes one or more instance groups, resulting in instances, upon each is layered one or more job templates, and each job template can have a dependency on packages. The origin of these job templates and packages will become clear now as we introduce BOSH releases, and the top-level deployment manifest attribute `releases`.

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

This deployment manifest will require that a BOSH release with the name `zookeeper` and version `0.0.7` be already uploaded to your BOSH environment.

## Upload Releases

You can manually upload a BOSH release using the `bosh upload-release` command:

```
> bosh upload-release https://bosh.io/d/github.com/cppforlife/zookeeper-release?v=0.0.7
```

This URL must ultimately resolve to a file originally produced by the `bosh create-release` command. Later in the Ultimate Guide to BOSH you will learn how to create and publish your own BOSH releases.

Approximately, `bosh upload-release https://url/to/someone-else-release-1.2.3.tgz` is similar to using a package manager to install a pre-created package, such as Debian `apt-get install someone-elses-package`.

A distinction is that `bosh upload-release` only populates the external BOSH release into our BOSH environment; it does not unpack the release into any running deployment instances. We do not use a newly uploaded BOSH release until we run `bosh deploy`.

So perhaps `bosh upload-release` is more similar to copying an upstream Debian package into your own Debian package cache, and `bosh deploy` is similar to running `apt-get install` against your own package cache.

## Automatically Upload Releases

A wonderful feature of a base BOSH deployment manifest can be to include the specific details for `bosh deploy` to discover and install the version that is required. This reduces the additional steps for an operator to bootstrap a new system or upgrade an existing deployment. They can simply run `bosh deploy`. Any missing releases will be automatically fetched and installed into the BOSH environment.

```yaml hl_lines="3 4"
releases:
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release
```

When you run `bosh deploy`, the CLI will see the `git+https` protocol determine that https://github.com/cppforlife/zookeeper-release is a Git repository containing information about all releases. The CLI will look inside the repository for the job templates and packages for `version: 0.0.7`. The CLI will then download these job templates and packages to your local computer or jumpbox, and then it will upload them to your BOSH environment.

The `bosh deploy` command will use your local `git` configuration and credentials to access Git repositories, thus making it possible to use private Git repositories to describe BOSH releases.

## Finding Existing BOSH Releases

There are many existing BOSH releases that you might like to use. Here are some suggestions for discovering them.

You can discover existing BOSH releases at https://bosh.io/releases.

Try searching Github https://github.com/search?q=bosh+release.

Look thru the [Ask for Help](/ask-for-help/suggestions/) suggestions to find other BOSH users or consultants who can help.

Talk to mysterious strangers at dinner parties.

## What is a BOSH Release?

A BOSH release is a versioned bundle of job templates, and their dependent packages, that describe a complete running system. The developers of a BOSH release benefit by iterating on bespoke application source code, dependency package versions, and configuration file templates all in the one project. The operators of a BOSH release benefit by using a combination of file templates and packages that are exactly the same combination as the developers intended.

We originally encountered [job templates](/instances/#job-templates) and [packages](/instances/#packages) when we were trawling through the inside of BOSH instances. Monit was configured to start and keep processes running. These processes were configured within job templates: how to run the process and how to configure the application within the process. The job templates need software packages to be pre-installed. These packages need to have been pre-compiled from trusted source files.

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

If your BOSH release does not document any metadata about the origins of each blob, you can always download the blobs and look inside them.

To fetch all the blobs referenced in the current `config/blobs.yml` file:

```
bosh sync-blobs
bosh blobs
```

The files referenced in `config/blobs.yml` will be downloaded into the `blobs` folder:

```
Path                               Size     Blobstore ID                          Digest
java/jdk-1.7.0_51.tar.gz           132 MiB  8a697fac-6cc4-443f-7009-b1c62e3d22b8  bee3b085a90439c833ce18e138c9f1a615152891
zookeeper/zookeeper-3.4.10.tar.gz  33 MiB   868354bd-c49e-41b7-49a9-b110091a82af  eb2145498c5f7a0d23650d3e0102318363206fba
```

To discover the nature of `blobs/java/java-1.7.0_51.tar.gz` we can use the `tar` application to look at its files. Its `LICENSE` file gives us the biggest clue that it is Oracle Java:

```
Please refer to http://java.com/license
```

Whilst a BOSH release might not retain the metadata about the origins of its blobs, it does guarantee that all developers and operators of a BOSH release will be using the same blobs.

## Versions

Perhaps you've noticed that our continuing example `zookeeper` BOSH release is at version v0.0.7 but it bundles Apache ZooKeeper v3.4.10. Perhaps this worries you. Deeply. Why is a BOSH release for running a cluster of Apache ZooKeeper versioned so independently from the version of Apache ZooKeeper it is running?

You're not a bad person for being a versionoholic. And I'm not here to make you feel bad for a problem you are imagining. Although I do notice I'm using negative words to describe your condition. "Your condition," HA! I did it again.

Traditional software packages themselves have not always had a simple job of versioning themselves in alignment with the upstream software they are bundling. Sometimes the version number must leak into the name of the package itself. Consider Python. In most packaging distributions, such as Debian, if you install a package called `python` you will get Python v2.7.10 or similar. If you install `python3` then you will get v3.X.Y. The forking of Python packaging into two packages `python` and `python3` was because packaged applications that depended on a `python` package assumed version v2+ and generally were flexible about which minor version of Python 2 they run upon. But those same packaged applications did not run upon Python 3. So the maintainers of Python packages made a decision to protect `python` v2+ applications by bundling Python 3 in an independently named package `python3`.

Without knowing this abridged history of Python language and the downstream packaging communities, the names and version numbers of the two families of Python packages is confusing at a glance. If you install `python3` package you get a Python version that is v3+, but there may be no `python2` package, and `python` package does not install v1+ but rather installs v2+.

BOSH release version number suffer from a similar situation, but it is aggravated because BOSH releases bundle many software projects which each have their own independent version sequence. For example, earlier we discovered that the `zookeeper/0.0.7` BOSH release contained Oracle Java 1.7.0_51 and Apache ZooKeeper 3.4.10.

Any change to the BOSH release will result in an upward change to the version number of the BOSH release.

If the current version of a BOSH release is `0.0.7`, then the developers must decide that the next release version will be: `0.0.8`, `0.1.0`, `1.0.0`, or perhaps something else.

Changing the bundled version of Java or ZooKeeper might mean the developers decide one a small version change, say `0.0.8`, or a larger version change, say `0.1.0` or `1.0.0`.

Fixing the `zookeeper` job template might result in a small version change, say to `0.0.8`. Drastically changing the names of operator-provided properties to the `zookeeper` job template might suggest a larger version increase, say `0.1.0` or `1.0.0`.

The version number of a BOSH release is part of the communication strategy of its developers to you the operators who want to upgrade their production systems. But the version number is just one part. With each new BOSH release version, hopefully its developers are including release notes or a changelog, are updating documentation, as well as updating the base deployment manifest and operator files to help operators perform their upgrades with confidence.

A BOSH release, and its accompanying deployment manifests, operator files, and documentation, is not about packaging software like traditional software packages. A BOSH release is a point-in-time description of a running system. As such, its developers will have an independent version number from any of the upstream bundled software it includes.

## Transparent Packaging

As a user of a traditional software packaging distribution, you will typically download a pre-compiled version of the package for your operating system and architecture. For example, if you used a traditional package system to install ZooKeeper you would not expect to see the ZooKeeper Java source code be downloaded and then compiled. From your perspective the "ZooKeeper package" is the resulting pre-compiled package that you download. But from the package author's perspective, the pre-compiled package you download is a byproduct of the package distribution system you are using. Traditional software package distribution systems have been so wonderful that you've probably never needed to worry about how they work, nor perhaps who is doing the work.

This opaque two-sided system -- package authors and the distribution system on one side, and the packaging-consuming users on the other side -- has some downsides. The consumers do not know what is being installed. The consumers do not know how to modify what is being installed. The consumers do not know who is maintaining the packages (if they are independent of the upstream software they are packaging). The consumers are subsequently not actively encouraged to help with the long-term maintenance of packages. These are not insurmountable problems, but every barrier to discovery and participation reduces their likelihood.

At the time I discovered BOSH, I was working for Engine Yard which maintained its own Gentoo distribution. I worked there for two years and never figured out the tool chain nor the social rules associated with creating or fixing our Gentoo packages.

Not all package systems are awful. The rapid rise of the MacOS [Homebrew](https://brew.sh) package system comes from the openness of how each package is defined -- a simple file written in Ruby programming language -- and how new packages can be contributed to the collective distribution system.

In the era of Docker and the `Dockerfile`, a growing number of people have started bypassing packaging systems for the explicit declarations of how they want to compile and install software. This scenario is growing because there are many public `Dockerfile`s with examples of explicitly compiling and installing software. One of the primary benefits of Docker and Dockerfiles is to breakdown the walls between packaging/distribution and operations. Packaging and using software is becoming the same role.

BOSH releases also help with transparency and enablement. After you've used a BOSH release, you can quickly discover how the BOSH release is constructed, and then contribute fixes and improvements. The entire tool chain for maintaining a BOSH release is at your finger tips with the `bosh` CLI.

## Comparing BOSH Packages to Traditional Software Packages

There are obvious similarities and some subtle distinctions between the packages within a BOSH release and traditional software packages. Discussing the similarities and differences might help you.

When a BOSH package is installed, during `bosh deploy`, all of its files will be placed within a single folder `/var/vcap/data/packages/package-name/some-long-guid`. Remember, `/var/vcap/data` is a large ephemeral disk, not the possibly small root disk. The job templates do not reference this folder directly. Instead a symlink `/var/vcap/packages/package-name/` will be created. Job templates will need to setup the `$PATH` variable to access the executables within each package.

When a traditional software package is installed, say using `apt-get install package-name`, its files will be placed throughout the file system. The location of files is encoded within the package, and so it cannot benefit from knowledge about large ephemeral disks on the target server.

A BOSH package will be installed with specific dependency packages from the same BOSH release. The BOSH release author curates an explicit pairing of packages that have been tested together. You will deploy exactly the same combination. Your probability of success is higher, and the author's ability to provide support is higher.

A traditional software package can specify very loose dependencies. An operator who updates a software package might not be aware of desirable updates to dependencies. The author of a traditional software package that allows very loose dependencies might never be able to truly test their software against all permutations of dependencies, and the dependencies of those dependencies, and so on.

A BOSH package is transparently authored within its BOSH release, and collocated with the source files and upstream blobs that it uses.

A traditional software package is convenient to install, but can be very difficult to find the source for the packaging scripts and the specific upstream files that went into the installable package.

## Comparing BOSH Releases to Docker Images

A popular tool for describing, distributing, and running software is [Docker](https://www.docker.com/). There is a lot to like about Docker. You can iterate quickly to build and test your Docker images. You can easily upload your Docker image to a central repository. Operators will be running the same Docker image that the author tested.

This is the same delightful experience as shared with BOSH release authors and the operators who deploy the releases.

One distinction between curating BOSH releases and Docker images is responsibility for the base operating system.

A BOSH release makes very light assumptions about the underlying base operating system (provided by a BOSH stemcell). It might assume that `gcc` is installed, and perhaps a few other system libraries. But fundamentally a BOSH release will package all the primary software and dependencies that are used. The author of a BOSH release only needs to monitor the security vulnerabilities of the primary software and dependencies. All BOSH release authors do not need to monitor and patch their BOSH releases for vulnerabilities in the shared base operating system layer. Instead the BOSH stemcells are continously maintained by the BOSH core team.

It is the operators responsibility to ensure that all vulnerabilities are patched in the stemcell and BOSH releases, and that BOSH deployments are updated to use the new stemcells and releases as soon as possible. Fortunately for operators, the `bosh deploy` command makes this very easy for everyone in the team.

A Docker image bundles a base operating system. An operator looking to deploy a Docker image could look at the original `Dockerfile` and discover the intended base operating system:

```dockerfile
FROM ubuntu:16.04
```

This Docker image is built upon Canonical Ubuntu 16.04.

Later in the `Dockerfile` we might see that the "latest" packages are being installed:

```dockerfile
RUN apt-get update && apt-get install curl -y
```

Unfortunately, this brief description does not communicate the relative recency of neither the base operating system nor the packages. It only indicates that the author/publisher used the latest versions at the time they built that layer of the Docker image.

It is the operators responsibility to ensure that all vulnerabilities are patched as quickly as possible within Docker images, and that running containers are shutdown and restarted to use the new Docker images.

But this responsibility is bottlenecked by each Docker image author's effective stewardship to monitor all vulnerabilities at all layers of the operating system, packaging system, and their own bespoke software.

## Discovering the Source of a BOSH Release

If you are debugging a BOSH deployment which includes BOSH releases authored by other teams, it can be relatively simple to discover the location of the source code.

Either fetch the entire BOSH deployment manifest from your BOSH environment:

```
bosh manifest
```

Or fetch only the `releases` section:

```
bosh int <(bosh manifest) --path /releases
```

The output of the latter might be:

```yaml hl_lines="3"
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release
```

The `url` value here gives us our answer: https://github.com/cppforlife/zookeeper-release

Alternately, the output might include `url` values that infer the location of the Git repository:

```yaml hl_lines="3 7"
- name: binary-buildpack
  sha1: c5ba6b6d99b972ec34dece478302351d8b4f6bbc
  url: https://bosh.io/d/github.com/cloudfoundry/binary-buildpack-release?v=1.0.14
  version: 1.0.14
- name: capi
  sha1: 94da536a79b95bf9b723d30ab42a944938cf2e76
  url: https://bosh.io/d/github.com/cloudfoundry/capi-release?v=1.43.0
  version: 1.43.0
```

Fortunately, the two `url` values transparently include the original source location of each BOSH release: https://github.com/cloudfoundry/binary-buildpack-release, and https://github.com/cloudfoundry/capi-release respectively.

We can clone these Git repositories to see their contents. Even better, using the same `bosh` CLI we can create and test variations of the BOSH releases.

## Building and Testing a BOSH Release

TODO

```
bosh create-release --force
bosh upload-release --rebase
bosh deploy manifests/zookeeper.yml -o manifests/dev.yml
```

## Compiling Packages

One of the built-in features of a BOSH environment is to compile packages on demand.
