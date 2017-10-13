# Complete Deployment Manifest

In the preceding sections of the Ultimate Guide to BOSH we have slowly introduced all the concepts of deploying systems with a BOSH environment, and all the sections of a BOSH deployment manifest.

We can now review the entire working deployment manifest for our `zookeeper` example from https://github.com/cppforlife/zookeeper-release. All references to this file in this section will be from a parent folder into which this repository has been cloned:

```bash
git clone https://github.com/cppforlife/zookeeper-release
cat zookeeper-release/manifests/zookeeper.yml
```

```yaml
---
name: zookeeper

releases:
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release

stemcells:
- alias: default
  os: ubuntu-trusty
  version: latest

update:
  canaries: 2
  max_in_flight: 1
  canary_watch_time: 5000-60000
  update_watch_time: 5000-60000

instance_groups:
- name: zookeeper
  azs: [z1, z2, z3]
  instances: 5
  jobs:
  - name: zookeeper
    release: zookeeper
    properties: {}
  - name: status
    release: zookeeper
    properties: {}
  vm_type: default
  stemcell: default
  persistent_disk: 10240
  networks:
  - name: default

- name: smoke-tests
  azs: [z1]
  lifecycle: errand
  instances: 1
  jobs:
  - name: smoke-tests
    release: zookeeper
    properties: {}
  vm_type: default
  stemcell: default
  networks:
  - name: default
```

## Use of Deployment Manifest

The deployment manifest is used by `bosh deploy` command to instruct your BOSH environment to create or update a deployment.

You can provide a full-formed manifest:

```
bosh deploy zookeeper-release/manifests/zookeeper.yml
```

Alternately, you can amend a base manifest with `-o` [Operator files](/deployment-updates/#operator-files) and `-v` [Variables](/deployment-updates/#deployment-manifest-variables). For example, to modify the deployment name of the uploaded deployment manifest to use the current `$BOSH_DEPLOYMENT` value to create an Operator file:

```
export BOSH_DEPLOYMENT=zookeeper-demo
cat > change-deployment-name.yml <<YAML
---
- type: replace
  path: /name
  value: $BOSH_DEPLOYMENT
YAML
bosh deploy zookeeper-release/manifests/zookeeper.yml \
  -o change-deployment-name.yml
```

## Name Attribute

The deployment `name` attribute is to confirm that the `bosh deploy` command is updating the correct deployment. The `name` attribute must match the `BOSH_DEPLOYMENT` variable or the `bosh deploy -d name` value.

If our `name` attribute is set:

```yaml
name: zookeeper
```

Then the `bosh deploy` command must be executed in one of two ways.

1. Using `BOSH_DEPLOYMENT` environment variable:

    ```
    export BOSH_DEPLOYMENT=zookeeper
    bosh deploy zookeeper-release/manifests/zookeeper.yml
    ```

    This is the method used by default in all examples within the Ultimate Guide to BOSH. It makes the `bosh` CLI examples simpler.

2. With explicit `-d` flag option placed anywhere in the command:

    ```
    bosh -d zookeeper deploy zookeeper-release/manifests/zookeeper.yml
    ```

Commonly, BOSH deployment manifests provided by upstream projects will include a default `name`; typically the same name as the project. If you want to modify the deployment name, then look to the example at the top of this chapter where we create `change-deployment-name.yml`.

Once a BOSH deployment has been created successfully, many other `bosh` commands will be available to inspect or interact with your running deployment. They will similarly use these two methods to target a BOSH deployment by name.

## Releases Attribute

* [Releases](/releases/)

```yaml
releases:
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release
```

## Stemcells

* [Stemcells](/stemcells/#stemcells-in-deployment-manifests)

```yaml
stemcells:
- alias: default
  os: ubuntu-trusty
  version: latest
```
