# Deployment Updates

TODO

## Update Sequence

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
