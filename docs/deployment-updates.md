# Deployment Updates

One of the fabulous features of BOSH is the ability to change, scale, and evolve your deployments throughout their multi-year lifespans.

## Scaling deployments

Over time, your long-lived BOSH deployments may need to be scaled up to cope with increased traffic or accrued data. Or you might discover that the initial deployment uses more resources than necessary and you want to downsize the deployment.

You can change the scale of your deployments using the same methodology for making any changes to a deployment:

1. Modify the deployment manifest
2. Deploy the manifest

    ```
    bosh deploy path/to/manifest.yml
    ```

For example, we can change our `zookeeper` cluster down to 3 instance, whilst simultaneously changing the instance size's `vm_type` attribute and increasing the `persistent_disk` size. The abridged `zookeeper-release/manifests/zookeeper.yml` deployment manifest showing the changed attributes:

```yaml
---
name: zookeeper

instance_groups:
- name: zookeeper
  instances: 3
  vm_type: large
  jobs:
  - name: zookeeper
    release: zookeeper
  - name: status
    release: zookeeper
  persistent_disk: 20480

- name: smoke-tests
  ...
```

After making these changes and saving the file, you would then run:

```
> bosh deploy zookeeper-release/manifests/zookeeper.yml
```

The `bosh` CLI will display the various proposed changes. Once you confirm by pressing `y`, the new deployment manifest will be uploaded to your BOSH director, which will perform the changes.

## Modifying the Running Deployment Manifest

If you do not have the original deployment manifest available when you want to modify it, you have the option to download the last deployment manifest that was successfully deployed.

```
bosh manifest > manifest.yml
```

After modifying this file, you could then apply the changes with `bosh deploy`:

```
bosh deploy manifest.yml
```

## Operator Files

It can be common that the deployment manifest comes from a source repository that you do not maintain. Our common example `zookeeper-release/manifests/zookeeper.yml` is a file within the https://github.com/cppforlife/zookeeper-release repository. If you directly modify this file you will not be able to commit the changes to this repository.

Instead, the `bosh deploy` command allows you to apply "operators" to the base manifest; that is, changes or amendments. We use the `bosh deploy -o path/changes.yml` flag for each operator file.

An example operator file that will change the number of `zookeeper` instances from 5 to 3 is:

```yaml
- type: replace
  path: /instance_groups/name=zookeeper/instances
  value: 3
```

Another example - either in a new file or appended to the file above - to change the `persistent_disk` attribute for each `zookeeper` instance:

```yaml
- type: replace
  path: /instance_groups/name=zookeeper/persistent_disk
  value: 20480
```

Finally, to change the `vm_type`:

```yaml
- type: replace
  path: /instance_groups/name=zookeeper/vm_type
  value: large
```

If these three YAML snippets were in a file `zookeeper-scale.yml`, then to apply the operators to our deployment manifest:

```
bosh deploy zookeeper-release/manifests/zookeeper.yml -o zookeeper-scale.yml
```

## Update Sequence

## Update Batches

TODO
```yaml
update:
  canaries: 2
  max_in_flight: 1
  canary_watch_time: 5000-60000
  update_watch_time: 5000-60000
```

## Renaming An Instance Group

Over the lifetime of a deployment you might merge or split out job templates between instance groups. You might then want to rename the instance groups. This can be done using the deployment manifest.

**But**, if you simple change the name of the instance group the BOSH director will not know you wanted to rename an existing instance group. It will destroy the previous instance group, orphan its persistent disks, and then provision a new instance group with new persistent disks.

To **rename** the `zookeeper` instance group in our `zookeeper` deployment manifest, we add the `migrated_from` attribute to our deployment manifest.

```yaml
instance_groups:
- name: zk
  migrated_from: [zookeeper, zookeeper-instances]
  ...
```

When we next run `bosh deploy`, the BOSH director will harmlessly rename any `zookeeper` or `zookeeper-instances` instances to their new name `zk`. On subsequent `bosh deploy` operations, with the instance group now called `zk`, the `migrated_from` attribute will be ignored.
