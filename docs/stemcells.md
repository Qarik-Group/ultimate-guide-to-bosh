# Stemcells

Cloud infrastructures require a pre-existing virtual filesystem, also called a machine image, to provision new cloud servers. For example, to provision an AWS EC2 server you will need an AWS AMI.

The BOSH director expects that the cloud servers it provisions will behave in a certain way when it wants to interact with them. For example, the BOSH director expects that each cloud server will have a BOSH Agent installed and running. We will introduce the BOSH Agent below.

Towards this dual requirement - a preexisting machine image, which is pre-populated with the BOSH Agent and other software and configuration - that we now introduce BOSH Stemcells.

## Stemcells in Deployment Manifests

My continuing objective with the Ultimate Guide to BOSH is that you feel good as you are reading each section in sequence. Towards this goal, the abridged deployment manifests and `cloud-config` examples have omitted sections that are actually required by the BOSH director. We now introduce the top-level `stemcells` attribute to our deployment manifests, and the `stemcell` attribute for each instance group.

```yaml
name: zookeeper

releases:
- name: zookeeper
  version: 0.0.7
  url: git+https://github.com/cppforlife/zookeeper-release

stemcells:
- alias: ubuntu
  os: ubuntu-trusty
  version: latest

instance_groups:
- name: zookeeper
  instances: 5
  stemcell: ubuntu

- name: smoke-tests
  lifecycle: errand
  instances: 1
  stemcell: ubuntu
```

Let's look at the `stemcells` section:

```yaml
stemcells:
- alias: ubuntu
  os: ubuntu-trusty
  version: latest
```

Although each BOSH release will have an implicit preference for a stemcell (most BOSH releases are developed/tested/deployed against Ubuntu stemcells), there is no metadata or contract within a BOSH release to help `bosh deploy` fail fast or fail with helpful error messages if you use the wrong stemcell.

In the case of the `zookeeper` deployment manifest, the selection of an `os: ubuntu-trusty` stemcell can be discovered from the project's own [sample deployment manifest](https://github.com/cppforlife/zookeeper-release/blob/6f073fdbbf411babbde11085abb7f43cced8b8d3/manifests/zookeeper.yml#L9-L12). Good BOSH releases or deployment projects will provide sample BOSH deployment manifests.

The selection of `os: ubuntu-trusty` means that the BOSH director must already have an `ubuntu-trusty` stemcell preloaded before running `bosh deploy`. At the time of writing there is no facilities in the BOSH CLI nor BOSH director to automatically discover, download, and install the required stemcell for a deployment manifest.

The `version: latest` means that the deployment will use the latest available stemcell that has been uploaded to the BOSH director.

To discover the available stemcells in your BOSH director:

```
bosh stemcells
```

An example output for a BOSH director with the Google CPI might be:

```
Name                                    Version           OS             CPI  CID
bosh-google-kvm-ubuntu-trusty-go_agent  3445.11           ubuntu-trusty  -    stemcell-04231868...
~                                       3421.11           ubuntu-trusty  -    stemcell-61295c90...
bosh-google-kvm-windows2012R2-go_agent  1200.5.0-build.1  windows2012R2  -    ...packer-1499974558
```

In this example we can see two `ubuntu-trusty` stemcells with the latest version `3445.11` available for deployments. We can also see a `windows2012R2` stemcell has been uploaded and is available for deployments that include BOSH releases targeting Windows.

The `alias: ubuntu` attribute gives the `os` and `version` combination a name that we can now use within `instance_groups`. From our example manifest above, we added `stemcell: ubuntu` to each instance group:

```yaml
instance_groups:
- name: zookeeper
  instances: 5
  stemcell: ubuntu

- name: smoke-tests
  lifecycle: errand
  instances: 1
  stemcell: ubuntu
```

## Updating Stemcells

Anytime that `bosh deploy` is run, `version: latest` will be adjusted to any newer stemcells that have been uploaded to the BOSH director. The BOSH director will display this proposed update before commencing the deployment.

In this example, we have deployed a small (2 instance) zookeeper deployment on AWS using version 3541.9 of the Ubuntu Trusty AWS Xen HVM stemcell.

```
$ bosh stemcells
Name                                     Version  OS             CPI  CID
bosh-aws-xen-hvm-ubuntu-trusty-go_agent  3541.9*  ubuntu-trusty  -    ami-3207964a light

$ bosh deployments
Name       Release(s)       Stemcell(s)                                     Team(s)  Cloud Config
zookeeper  zookeeper/0.0.7  bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3541.9  -        latest
```

However since this stemcell was uploaded, a new version, 3541.10 has been released and so now we will update our stemcell with the new version.

```
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent

Task 102

Task 102 | 15:37:21 | Update stemcell: Downloading remote stemcell (00:00:01)
Task 102 | 15:37:22 | Update stemcell: Extracting stemcell archive (00:00:00)
Task 102 | 15:37:22 | Update stemcell: Verifying stemcell manifest (00:00:00)
Task 102 | 15:37:28 | Update stemcell: Checking if this stemcell already exists (00:00:00)
Task 102 | 15:37:28 | Update stemcell: Uploading stemcell bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3541.10 to the cloud (00:00:08)
Task 102 | 15:37:36 | Update stemcell: Save stemcell bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3541.10 (ami-3fb42c47 light) (00:00:00)

Task 102 Started  Thu Mar 29 15:37:21 UTC 2018
Task 102 Finished Thu Mar 29 15:37:36 UTC 2018
Task 102 Duration 00:00:15
Task 102 done

Succeeded

$ bosh stemcells
Name                                     Version  OS             CPI  CID
bosh-aws-xen-hvm-ubuntu-trusty-go_agent  3541.10  ubuntu-trusty  -    ami-3fb42c47 light
~                                        3541.9*  ubuntu-trusty  -    ami-3207964a light
```

On the next deployment of zookeeper - it will be rebuilt on the newer version of Ubuntu Trusty, 3541.10. This will require a re-compile of the application and then the underlying VMs will be re-created once the new version is compiled.

```
$ bosh deploy -d zookeeper manifests/zookeeper.yml

Using deployment 'zookeeper'

Release 'zookeeper/0.0.7' already exists.

  stemcells:
+ - alias: default
+   os: ubuntu-trusty
+   version: '3541.10'
- - alias: default
-   os: ubuntu-trusty
-   version: '3541.9'

Continue? [yN]: y

Task 103

Task 103 | 15:39:08 | Preparing deployment: Preparing deployment (00:00:00)
Task 103 | 15:39:08 | Preparing package compilation: Finding packages to compile (00:00:00)
Task 103 | 15:39:08 | Compiling packages: golang-1.8-linux/3eac55db0483de642b1be389966327e931db3e3f (00:01:41)
Task 103 | 15:40:49 | Compiling packages: zookeeper/43ee655b89f8a05cc472ca997e8c8186457241c1 (00:00:10)
Task 103 | 15:40:59 | Compiling packages: java/c524e46e61b37894935ae28016973e0e8644fcde (00:00:29)
Task 103 | 15:41:28 | Compiling packages: smoke-tests/ec91e258c41471227a759c2749e7295cb65eff5a (00:00:13)
Task 103 | 15:42:17 | Updating instance zookeeper: zookeeper/587013fa-a927-44b8-9ef4-1a2b73eca415 (0) (canary) (00:03:17)
Task 103 | 15:45:34 | Updating instance zookeeper: zookeeper/d5f3430c-5393-44bf-8030-78732846bacd (1) (canary) (00:03:21)

Task 103 Started  Thu Mar 29 15:39:08 UTC 2018
Task 103 Finished Thu Mar 29 15:48:55 UTC 2018
Task 103 Duration 00:09:47
Task 103 done

Succeeded
```

Once this is complete, we can see that the deployment is now running on stemcell version 3541.10.


```
$ bosh deployments
Name       Release(s)       Stemcell(s)                                      Team(s)  Cloud Config
zookeeper  zookeeper/0.0.7  bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3541.10  -        latest

```



## Finding Stemcells

You can discover stemcells for your CPI at http://bosh.io/stemcells. At the time of writing, there are stemcells published for the following major operating system distributions:

* Ubuntu Linux
* CentOS Linux
* Windows

The Ubuntu stemcells are the most commonly used base images, are the most battle tested in production systems around the world, and seem to the author to have the most security updates pushed out. I would recommend you always use an Ubuntu stemcell unless you have a strong requirement to choose an alternate.

The BOSH release you are deploying will have a specific requirement for either a Linux or Windows stemcell. If the BOSH release specifically requires CentOS Linux, then it will indicate this in its documentation and sample deployment manifests.

## Light Stemcells

On public cloud infrastructures - AWS, GCP, Azure - the BOSH Core Team publish shared machine images that are referenced by the stemcell file.

To quickly confirm what I mean, let's download a stemcell for AWS and look inside it.

```
curl -o stemcell-aws.tgz https://s3.amazonaws.com/bosh-aws-light-stemcells/light-bosh-stemcell-3445.11-aws-xen-hvm-ubuntu-trusty-go_agent.tgz
```

The `stemcell.MF` file within the archive is a YAML file referencing each Amazon Machine Image (AMI) for each region:

```
tar -axf stemcell-aws.tgz stemcell.MF -O
```

The output will look like:

```yaml
---
name: bosh-aws-xen-hvm-ubuntu-trusty-go_agent
version: '3445.11'
bosh_protocol: '1'
sha1: da39a3ee5e6b4b0d3255bfef95601890afd80709
operating_system: ubuntu-trusty
cloud_properties:
  ami:
    us-gov-west-1: ami-4a0d8e2b
    ap-northeast-1: ami-17d61871
    ap-northeast-2: ami-8f409be1
    ap-south-1: ami-7076301f
    ap-southeast-1: ami-5bd9ae38
    ap-southeast-2: ami-eff81e8d
    ca-central-1: ami-31a41d55
    eu-central-1: ami-de19afb1
    eu-west-1: ami-c0cf03b9
    eu-west-2: ami-eddbc889
    sa-east-1: ami-f89ce194
    us-east-1: ami-9a43afe0
    us-east-2: ami-ffab899a
    us-west-1: ami-5493a534
    us-west-2: ami-c03ec3b8
    cn-north-1: ami-296cbc44
```

We can confirm each AMI is a pre-created public AMI. For the `us-east-1` AMI:

[![aws-public-ami](/images/aws/aws-public-ami.png)](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Images:visibility=public-images;search=ami-9a43afe0;sort=name)

For AWS alone, the BOSH Core Team are creating 16 different AMIs in 16 different AWS regions for each AWS light stemcell.

As a BOSH user, you do not need to correctly select the right `ami-1234567` image. The BOSH director knows which region you are using and will use the appropriate public machine image from the list above.

## On-Premise Stemcells

If you are using an on-premise cloud infrastructure such as vSphere or OpenStack then your stemcells cannot reference pre-built machine images. Instead, the BOSH director will have the task of creating machine images within your cloud infrastructure that it can use for provisioning cloud servers.

These stemcells will be substantially larger than "light" stemcells as they contain the entire machine image. On-premise stemcells will be 300+ MB in size, whereas "light" stemcells are tiny 20KB files (discussed in preceding section).

[![bosh-io-stemcell-sizes](/images/bosh-io-stemcell-sizes.png)](http://bosh.io/stemcells)

Your cloud infrastructure BOSH CPI has the responsibility of converting a stemcell into a machine image.

For example, the OpenStack CPI will [interact with OpenStack Glance](https://github.com/cloudfoundry-incubator/bosh-openstack-cpi-release/blob/master/docs/openstack-api-calls.md#all-calls-for-api-endpoint-image-glance) to convert a stemcell into an OpenStack Machine Image.

## Agent

One of the primary reasons for BOSH stemcells, rather than allowing you to bring your own base machine images, is that they have the BOSH agent preinstalled.
