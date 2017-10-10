# Complete Deployment Manifest

In the preceding sections of the Ultimate Guide to BOSH we have slowly introduced all the concepts of deploying systems with a BOSH environment, and all the sections of a BOSH deployment manifest.

We can now review the entire working deployment manifest for our `zookeeper` example from https://github.com/cppforlife/zookeeper-release

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

Alternately, you can amend a base manifest with `-o` [Operator files](/deployment-updates/#operator-files) and `-v` [Variables](/deployment-updates/#deployment-manifest-variables). For example, to modify the deployment name of the uploaded deployment manifest to use the current `$BOSH_DEPLOYMENT` value:

```
export BOSH_DEPLOYMENT=zookeeper-demo
cat > change-deployment-name.yml <<YAML
---
- type: replace
  path: /name
  value: ((deployment-name))
YAML
bosh deploy zookeeper-release/manifests/zookeeper.yml \
  -o change-deployment-name.yml \
  -v deployment-name=$BOSH_DEPLOYMENT
```

## Deployment Name

```yaml
name: zookeeper
```
