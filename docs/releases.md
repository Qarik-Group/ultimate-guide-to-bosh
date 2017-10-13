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

```yaml
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

Approximately, `bosh upload-release https://url/to/someone-else-release-1.2.3.tgz` is similar to using a package manager to install a pre-created package, such as Debian `apt-get install someone-eles-package`. The distinction here is that `bosh upload-release` only populates the external BOSH release into our BOSH environment. We do not use the BOSH release until we run `bosh deploy` and the releases are applied to instances via job templates and packages.

## External Release

A wonderful feature of a base BOSH deployment manifest can be to include the specific details for `bosh deploy` to discover and install the version that is required. This reduces the additional steps for an operator to bootstrap a new system or upgrade an existing deployment. They can simply run `bosh deploy`. Any missing releases will be automatically fetched and installed into the BOSH environment.

```yaml
releases:
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release
```

When you run `bosh deploy`, the CLI will assume that https://github.com/cppforlife/zookeeper-release is a `git` repository. The CLI will look inside the repository for the job templates and packages for the `version: 0.0.7`. The CLI will then download these job templates and packages to your local computer or jumpbox, and then it will upload them to your BOSH environment.

The `bosh deploy` command will use your local `git` configuration and credentials to access Git repositories, thus making it possible to use private Git repositories.

## Finding Existing BOSH Releases

There are many existing BOSH releases that you might like to use, before going down the path of authoring your own BOSH releases later in the Ultimate Guide to BOSH.

One location to discover existing BOSH releases is https://bosh.io/releases.

Another idea is to search Github https://github.com/search?q=bosh+release.
