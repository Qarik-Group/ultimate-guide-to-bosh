# Disks

One of the first demonstrations of BOSH I ever saw in April 2012 was "let's resize a disk". It is an incredibly powerful demonstration of BOSH. Managing disk sizes can be non-trivial on most cloud infrastructures; with BOSH it is a fully managed service:

As a user, it is just two steps:

1. change the persisent disk size or select a different persisent disk type in the deployment manifest
1. run `bosh deploy` to request the BOSH director orchestrate the changes

The BOSH director will now orchestrate the following sequence for you:

1. provision a second, larger persistent disk
1. attach it to the instance
1. format and mount the disk as an additional temporary mounted volume
1. stop all the processes
1. copy all the data from the older volume to the new volume (this can be time consuming for large amounts of data)
1. unmount the older volume
1. remount the new volume to `/var/vcap/store`
1. restart all the processes
1. detach the older persistent disk
1. delete the older, orphaned persistent disk in 5 days

You merely had to run `bosh deploy` and your BOSH environment does everything else above.

Growing your infrastructure has never been easier.

We first looked at disks in the section [Persistent Volumes](/instances/#persistent-volumes).

Each instance of an instance group can have a fully managed persistent disk (see [Multiple Persistent Disks](/disks/#multiple-persistent-disks) to move to multiple disks). It will be mounted at `/var/vcap/store` and is shared across all job templates collocated on the same instance.

This section will discuss persistent disk types, the provisioning and mounting sequence, and how the BOSH director fully manages the resizing of persistent disks between deployments.

## Persistent Disks and CPIs

Each cloud infrastructure and the BOSH Cloud Provider Interface (CPI) will have its own implementation of persistent disks.

The important implementation requirement of the CPI and the cloud infrastructure is that the persistent disk can be detached from one cloud server and reattached to another cloud server. These disk volumes are not necessarily located on the same host machine as the cloud server to which they are attached, rather will be located somewhere else on the network. Hence we can term them "network disk volumes".

For example, the AWS CPI will provision AWS Elastic Block Storage (AWS EBS). The vSphere CPI has more work to do to implement persistent disks as vSphere does not have a native network disk volume concept. The Docker CPI and Garden CPI, which manage local Linux containers rather than virtual machines, will implement persistent disks upon the host machine.

A BOSH persistent disk exists for the life of the instance group, not only during the lifespan of individual cloud servers. This is similar to the relationship between BOSH instances and the underlying cloud servers provided by the cloud infrastructure. The BOSH instance and BOSH persistent disk are concepts that exist for the lifetime of the deployment, but underneath they are implemented by cloud servers and network disk volumes that might be provisioned and destroyed to support resizing or upgrades.

## Persistent disks and volume mounts

If an instance has a single persistent disk then the BOSH director will organise for the disk to be formatted and mounted.

Each persistent disk is formatted with filesystem type `ext4` by default. [Alternate filesystem types](https://bosh.io/docs/persistent-disk-fs.html) are available.

Each persistent disk is mounted at `/var/vcap/store`.

### Pitfalls of forgetting persistent disks

At the time of writing, BOSH job templates do not have a way to communicate with the BOSH director that they **require** a persistent disk. With or without an external `/var/vcap/store` mount, these job templates will attempt to write data within this folder structure. Potentially large amounts of data. This scenario will result in system failures without a persistent disk mounted.

Consider a job template that writes data to `/var/vcap/store/zookeeper/mydb.dat`.

If there is a persistent disk mounted at `/var/vcap/store` then the file `/var/vcap/store/zookeeper/mydb.dat` will be safely stored upon this persistent disk. If the disk starts to fill up then it is a simple matter to resize the persistent disk (see the opening demonstration in [Disks](/disks)).

If there is no persistent disk mounted at `/var/vcap/store`, then the file `/var/vcap/store/zookeeper/mydb.dat` will be be stored upon the root volume `/`. The root volume is typically small (large enough only for the system's packages), ephemeral (changes to the root volume will be lost when the cloud server is recreated), and fixed in size (root volumes are typically not resized during the life of a deployment). Eventually the root volume will fill up and the instance's processes and perhaps system processes will begin to fail.

If your instances are ever experiencing failure, run `df -h` to check that your disks have not filled up. If the root volume `/` is at 100% then you have probably forgotten to include a persistent disk; or a job template has a mistake in it and is writing files outside of `/var/vcap/store` or `/var/vcap/data` volumes.

## Simple Persistent Disk

The simplest technique for allocating a persistent disk in a deployment manifest is the `persistent_disk` attribute of an `instance_group`. Consider this abbreviated deployment manifest:

```yaml
instance_groups:
- name: zookeeper
  instances: 5
  persistent_disk: 10240
```

BOSH will manage five different persistent disks and their association with the five `zookeeper` instances in this deployment. BOSH will use its CPI to request a `10240` Mb persistent disk from the cloud infrastructure. That is, on AWS it will provision five 10GB AWS EBS volumes and request that AWS attach each one to the five different AWS EC2 servers.

Continuing with the AWS EBS example, the AWS CPI has to choose an [EBS Volume Type](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html). At the time of writing, the AWS CPI defaults all disks to General Purpose SSD (`gp2`). Other CPIs will have their own defaults:

* Google CPI defaults to `pd-standard` https://bosh.io/docs/google-cpi.html#disk-types
* AWS CPI defaults to `gp2` https://bosh.io/docs/aws-cpi.html#disk-pools
* vSphere CPI defaults to `preallocated` https://bosh.io/docs/vsphere-cpi.html#disk-pools
* OpenStack CPI defaults to `SSD` https://bosh.io/docs/openstack-cpi.html#disk-pools

## Persistent Disk Types

The benefits of `persistent_disk` attribute are that it is simple (just a number of megabytes) and cloud infrastructure agnostic.

The downside is the inability to customise the cloud infrastructure attributes of the disk. For this we switch from `persistent_disk` to the `persistent_disk_type` attribute.

```yaml
instance_groups:
- name: zookeeper
  instances: 5
  persistent_disk_type: large
```

In the updated example above, we are now using a text label `large` to reference a new item `disk_types` from your `bosh cloud-config`:

```yaml
disk_types:
- name: default
  disk_size: 3000
  cloud_properties: {}
- name: large
  disk_size: 50_000
  cloud_properties: {}
```

The `name` attribute is the reference label used by `persistent_disk_type` in deployment manifests. The `disk_size` is in megabytes.

The default `cloud_properties` for each item in `disk_types` is the same as for the `persistent_disk` section above. The linked URLs to documentation describe the cloud infrastructure options.

In our modified `zookeeper` deployment manifest above, our `persistent_disk_type: large` currently maps to a 50,000 MB disk with default `cloud_properties` for the cloud infrastructure.

If the `name: large` attributes are subsequently modified in `bosh cloud-config` then our deployment will not be immediately affected. Our deployment will pick up the changes when we next run `bosh deploy`.

### Available Disk Types

You can use the `bosh` CLI to discover the `disk_types` and their labels on your BOSH director:

```
bosh int <(bosh cloud-config) --path /disk_types
```

The example output might be similar to:

```yaml
- disk_size: 3000
  name: default
- disk_size: 50000
  name: large
- disk_size: 1024
  name: 1GB
- disk_size: 5120
  name: 5GB
- disk_size: 10240
  name: 10GB
- disk_size: 102400
  name: 100GB
```

In this example, the optional `cloud_properties` attribute was not included.

To curate the lists of `disk_types` shared amongst your deployments you will need to update your `cloud-config`. We will discuss this later in the section [Cloud Config Updates](/cloud-config-updates).

## Orphaned Disks

The BOSH director does not immediately delete disks that are no longer needed. These disks are marked as orphaned and will be garbage collected after 5 days.

To save money or to reclaim disk space on the host system you can manually delete them.

The get a list of all orphaned disks:

```
bosh disks --orphaned
```

After deleting our `zookeeper` deployment, or resizing all its disks, the output might look like:

```
Disk CID          Size    Deployment  Instance               AZ  Orphaned At
disk-5d968d18...  10 GiB  zookeeper   zookeeper/0d382e23...  z1  ...
disk-bc5945ca...  10 GiB  zookeeper   zookeeper/0acaf2d8...  z2  ...
disk-bac28797...  10 GiB  zookeeper   zookeeper/a94bb2b7...  z1  ...
disk-bd67097c...  10 GiB  zookeeper   zookeeper/97c88778...  z2  ...
disk-f7e5d3cf...  10 GiB  zookeeper   zookeeper/e4d84929...  z3  ...
```

### Reattach orphaned disks

The primary purpose of orphaning disks is to allow you to recover from the error of accidentally deleting your deployments or from erroneous deployment changes.

If we made an error with our `zookeeper` deployment, then we would need to reattach five different disks.

To reattach an orphaned disk to a running instance:

* run `bosh stop name/id` command to stop instance (or multiple instances) for repair
* run `bosh attach-disk name/id disk-cid` command to attach disk to given instance
* run `bosh start name/id` command to resume running instance workload

The `bosh attach-disk` command will replace any currently attached disk, thus orphaning it.

The `attach-disk` command can also attach available disks found in the cloud infrastructure. They don’t have to be listed in the `bosh disks --orphaned` list.

### Delete orphaned disks

You can delete individual disks using their "Disk CID":

```
bosh delete-disk disk-3b40021c...
```

If you want to delete every orphaned disk, then this little snippet might come in handy:

```
bosh disks --orphaned | grep disk- | awk '{print $1}' | xargs -L1 bosh delete-disk -n
```

## Multiple Persistent Disks

BOSH deployment manifests can request multiple persistent disks, apply any disk formatting, and mount them to any path. This topic is discussed in a great blog post by Chris Weibel [Mounting Multiple Persistent Disks with BOSH](https://www.starkandwayne.com/blog/bosh-multiple-disks/).
