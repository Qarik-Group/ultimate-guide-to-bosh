# Ultimate Guide to BOSH

[BOSH](https://bosh.io) is an open source tool for release engineering, deployment, lifecycle management, and monitoring of distributed systems.

It's incredible. Huge companies are using. Tiny companies are using it. You too could be using it.

This is the Ultimate Guide to BOSH.

It will place you in the middle of daily life with BOSH and gradually guide you towards understanding, and then deploying your own systems, and then through to deep understanding. You'll become a raving fan.

# TOC

   * [Ultimate Guide to BOSH](#ultimate-guide-to-bosh)
   * [TOC](#toc)
   * [Introduction](#introduction)
      * [WIP](#wip)
      * [Guide to the guide](#guide-to-the-guide)
      * [Joyful operations](#joyful-operations)
      * [Brief history of BOSH](#brief-history-of-bosh)
      * [BOSH in production](#bosh-in-production)
      * [Why write the Ultimate Guide to BOSH?](#why-write-the-ultimate-guide-to-bosh)
      * [Additional sources of information](#additional-sources-of-information)
   * [Why BOSH?](#why-bosh)
      * [What is a running software system?](#what-is-a-running-software-system)
      * [Choose your own deployment level](#choose-your-own-deployment-level)
      * [Assumptions](#assumptions)
      * [Continuous Integration and Continuous Delivery](#continuous-integration-and-continuous-delivery)
   * [Deployments](#deployments)
      * [New deployments](#new-deployments)
      * [New deployments of Zookeeper](#new-deployments-of-zookeeper)
      * [BOSH Architecture, Part 1](#bosh-architecture-part-1)
      * [CPI - the ultimate Cloud Provider Interface abstraction](#cpi---the-ultimate-cloud-provider-interface-abstraction)
      * [Instances](#instances)
      * [SSH](#ssh)
      * [Shell user prompts in examples](#shell-user-prompts-in-examples)
      * [Monit process monitoring](#monit-process-monitoring)
      * [Job templates describe processes](#job-templates-describe-processes)
      * [Job templates](#job-templates)

NOTE: update TOC using `bin/replace-toc`

# Introduction

## WIP

I recently started writing this. If you're actually reading this guide now, please let me know (I'll actively ask you to review bits as I write them) and please "Watch" this repo. Perhaps I can update it via Github Releases so you can get notifications of new sections or updates. Or better jokes.

## Guide to the guide

This Ultimate Guide to BOSH is to be read linearly. Each section will build upon the preceding sections.

This guide will be a single [README.md](README.md) until that just doesn't make sense anymore.

The guide use sample commands and videos to show you real systems instead of expecting that you can deploy systems yourself on day 1.

Later in the guide you will deploy your own BOSH and use it to deploy systems. At that point you will install the `bosh` command-line tool, and you will need to decide which target cloud infrastructure you will use.


## Joyful operations

You're a professional. You're resourceful. You're a king maker. You keep your organisation in the business of winning.

In past lives you might have been called: developer, sysadmin, or devops.

You're always on the look out for better tools, better mental models, and better systems.

I'm going to show you how I do some day-to-day activities using BOSH. You get to decide if you'd like to level up your superhero status and learn how to do this too. Learning is involved. Effort. New tools. New ecosystem. I definitely think its worth it. Let me know what you decide!

Deploy a 5-node cluster of Zookeeper to Amazon AWS:

```
git clone https://github.com/cppforlife/zookeeper-release
cd zookeeper-release
export BOSH_ENVIRONMENT=aws
export BOSH_DEPLOYMENT=zookeeper
bosh deploy manifests/zookeeper.yml
```


Sanity check that the zookeeper cluster is working:

```
bosh run-errand smoke-tests
```

Upgrade to new version of Zookeeper:

```
git pull
bosh deploy manifests/zookeeper.yml
```

Upgrade the base operating system to push out critical security patches:

```
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent
bosh deploy manifests/zookeeper.yml
```

If Amazon AWS deletes one of your VMs, heal your cluster by receating a new VM, reattach the persistent disk, remount it, and restart all the processes to join the Zookeeper node into the cluster:

```
# do nothing, this resurrection will happen automatically
```

List the health of each Zookeeper server, including disks:

```
bosh instances --vitals
```

SSH into one of the Zookeeper servers to check on something:

```
bosh ssh zookeeper/0
```

Run a command on each Zookeeper server and display results:

```
bosh ssh -c '/var/vcap/jobs/zookeeper/bin/ctl status' -r
```

Tear down your Zookeeper cluster:

```
bosh delete-deployment
```

## Brief history of BOSH

I was fortunate to be invited to the VMWare campus in Palo Alto CA on April 11 2012 for the unveiling of "the Outer Shell" that deploys Cloud Foundry. The Outer Shell was called BOSH. This is an acronym for "BOSH Outer SHell". Engineers know one thing: recursion is funny.

I invited myself to the VMware campus for two days to meet the developers of BOSH and Cloud Foundry and came away fascinated by the vast scope of problems that the BOSH team was trying to solve. The BOSH team were kind enough to either answer a question or fix BOSH so the question was void.

In 2012 I was very publicly excited about BOSH on Twitter ([@drnic](https://twitter.com/drnic)) and I gave presentations at meetups and conferences about BOSH. I also began creating new open source projects to help myself and others to use BOSH and to create new BOSH releases. I also created the first [BOSH Getting Started](https://github.com/cloudfoundry-community-attic/LEGACY-bosh-getting-started) guide.

Many of those early presentations and guides were helpful to the many people who've discovered BOSH and are using it to run production systems at huge scales.

In 2013 the BOSH project was taken over by Pivotal engineering and has been gifted to the Cloud Foundry Foundation to secure its long term success as an open source, open community project. Thanks to Pivotal, IBM and other members of the Cloud Foundry Foundation the BOSH project has received huge consistent investment to this today.

There are many people in the history of BOSH who have directly made BOSH what it is, actively sponsored its investment, or evangelised it.

* James Watters, SVP Product at Pivotal, has been the loudest cheerleader of BOSH on the planet
* Ferran Rodenas, Consultant at Stark & Wayne, has been my BOSH friend since 2012
* Dmitriy Kalinin, Product Manager for BOSH, has been driving his incredible vision for BOSH
* Original BOSH team at VMware - Mark Lucovsky, Vadim Spivak, Oleg Shaldybin, Martin Englund - who had the original vision and execution to create the ultimate tool for release engineering, deployment, lifecycle management, and monitoring of distributed systems.

## BOSH in production

BOSH is the core technology to Pivotal Ops Manager and its Pivotal Network delivery system for complex on-premise software systems. BOSH is the deployment technology used behind the scenes for [Pivotal Web Services](https://run.pivotal.io) which runs upon Amazon AWS.

BOSH is the deployment technology used behind the scenes by [IBM BlueMix](https://www.ibm.com/cloud-computing/bluemix/) on its Soft Layer infrastructure.

BOSH is the deployment technology used by GE [Predix](https://www.predix.io/) which runs on various clouds and data centres.

BOSH is the deployment technology used by Swisscom [appCloud](https://developer.swisscom.com/) which runs inside Swisscom data centres.

These are huge companies who have small teams running huge production systems using BOSH.

On the smaller end - our consultancy [Stark & Wayne](https://www.starkandwayne.com)  - uses BOSH to run a variety of our internal systems across vSphere, Amazon AWS and Google Compute. (The rest of our systems run upon Cloud Foundry itself, such as https://www.starkandwayne.com and https://www.starkandwayne.com/blog).

## Why write the Ultimate Guide to BOSH?

BOSH has been my not-so-secret weapon since 2012. Yet you might not yet be using BOSH.

I don't want to write a book.

I wrote a PhD thesis in 2001 and 6 people read it (two supervisors, three judges, and myself). Fortunately this is enough people to be awarded a doctorate. Unfortunately it doesn't really qualify as "sharing knowledge".

I wrote my first blog post in 2006 and when I quickly reached 7 subscribers I knew I'd found my preferred medium for sharing.

I enjoy the continuous publication and feedback loop of blogging, and of sharing open source projects. I enjoy a relaxed writing style.

I don't want to write a "book" book.

I want to share all the wonders of BOSH with you. I want you to use BOSH. I want you to feel great using BOSH. I want you to feel like a superhero. I want you to convince your friends and colleagues to use BOSH. I want you to help me evangelise BOSH.

I also want you to switch to Queen's English, learn more about Australia, and to use the Oxford comma.

## Additional sources of information

In addition to this Ultimate Guide to BOSH, there are some other sources of factual knowledge and tutorials.

[BOSH documentation website](https://bosh.io/docs/) is very thorough and new features of BOSH now regularly appear simultaneously in this documentation site.

Duncan Winn's [Cloud Foundry: The Definitive Guide](http://shop.oreilly.com/product/0636920042501.do) he includes an entire chapter "BOSH All the Things" towards helping you deploy and operate Cloud Foundry.

Stark & Wayne's own blog (["bosh" tag](https://www.starkandwayne.com/blog/tag/bosh/)) has over fifty articles, tutorials and tiny tips on BOSH.

BOSH is an open source project. You can read the source code and learn how it works. A selection of repositories include:

* https://github.com/cloudfoundry/bosh-cli - the `bosh` CLI
* https://github.com/cloudfoundry/bosh - the BOSH director
* https://github.com/cloudfoundry/bosh-agent - the BOSH agent
* https://github.com/cloudfoundry/bosh-deployment - manifests for various permutations of deploying your own BOSH director

# Why BOSH?

First, let's answer the question:

## What is a running software system?

![app-stack](images/handdrawn/app-stack.jpg)

Your bespoke or user-facing application is either a compiled application (Golang) or source code that runs within an interpreter (Ruby or Python) or is compiled and requires an interpreter (JVM languages).

Your bespoke application will be composed of bespoke code plus third party software libraries (RubyGems for Ruby, NPM for Node, Wheels for Python, etc).

Your application will need to be configured (combination of local configuration files, environment variables, service discovery system) to run and connect to any dependent systems.

Your application will require local dependencies to be already installed - its interpreter, linked libraries, executable applications, etc.

Your application and its dependencies all require an operating system. And formatted disks. And networking to be configured.

All this runs in a virtual server/virtual machines (VMs) either in someone else's data centre called "the cloud" (Amazon AWS, Google Compute, Microsoft Azure) or someone else's data centre called "on premise" (but its really normally not in your building, is it?) running virtualisation software (vSphere, OpenStack).

Your applications and databases running on virtual machines will require disks: local or ephemeral disks that might not survive VM downtime or replacement; and persistent networked disks that are independent of each VM and will be available again if you need (or are forced) to replace your VMs.

All of this runs upon physical machines connected to actual storage systems and interconnected by real-world networking.

Servers need to be powered, so you'll need stable affordable electricity. Servers get hot, so you'll need cooling. Servers can be stolen or physically hacked, so you'll need security guards with appropriate lapel badges.

It's incredible that it all works. Click on https://google.com to check that it all works.

Note: The Ultimate Guide to BOSH will include unsolicited sarcasm and humour. With luck, you'll enjoy both the Ultimate Guide BOSH and the humour.

## Choose your own deployment level

You might define "deploying my system" at a different level to other people:

* using an application platform, such as Cloud Foundry or Heroku
* using a container orchestration system provided by someone else, such as Kubernetes, Docker, Amazon ECS
* using virtual machines provided by someone else, such as Amazon AWS, Google Compute, vSphere
* using bare metal machines provided by someone else
* racking bare metal servers or putting Raspberry Pis into the field

From the perspective of your organisation and their goals of efficiently using your time and energy,
hopefully you can start as high up this stack as possible. For example, there is simply nothing faster, more time efficient, and UI consistent as `cf push`-ing an application to any Cloud Foundry. Every system you deploy should have to first justify why it cannot be deployed to Cloud Foundry or Heroku or Google App Engine.

If you do need to "go down the stack" and take responsibility for more then you will need more help. Either your organisation will need to expect less from you and your team, or you'll need more tooling, automation, and education.

## Assumptions

The Ultimate Guide to BOSH assumes you need the latter: you need tooling, automation and education.

It also assumes that you have direct access to your virtualisation/cloud infrastructure - you have suitable AWS credentials, or a Google Compute account or vSphere admin access.

The Ultimate Guide to BOSH assumes you are prepared to learn a new tool, its features, and its quirks.

## Continuous Integration and Continuous Delivery

BOSH slots in very nicely into any continuous deployment systems you might already be using. The `bosh` command-line tool is a perfect abstraction for "please make this happen" that will make it pleasurable to move BOSH deployments into your CI/CD systems.

# Deployments

Let's begin!

The highest level concept of BOSH is the "deployment" of a system. The purpose of BOSH is to continuous run one or more deployments. For example, a cluster of servers that form a Zookeeper cluster is a deployment of the Zookeeper system.

In [Joyful operations](#joyful-operations) we began by creating a deployment:

```
> export BOSH_DEPLOYMENT=zookeeper
> bosh deploy manifests/zookeeper.yml
```

And we finished the lifecycle of that system by deleting the BOSH deployment:

```
> bosh delete-deployment
```

## New deployments

When we ask BOSH to provision a new system, BOSH takes upon the entire responsibility for making this happen:

* BOSH will communicate with your cloud infrastructure API to request new servers/virtual machines (called "instances")
* BOSH manages the base machine image used for each virtual machine (called "stemcells")
* BOSH will allocate an available IP address for each instance
* BOSH will communicate with your cloud infrastructure API to request persistent disk volumes (we will see soon that the `zookeeper.yml` manifest requires a persistent disk volume for each instance in the deployment)
* BOSH will request that the disk volumes are attached to the instances

Once the instances are provisioned and the disks are attached, BOSH then starts communicating with each instance:

* BOSH will format disk volumes if necessary
* BOSH will download the software required (called "packages")
* BOSH will construct configuration files for the packages and commence running the software (called "job templates")
* BOSH provides service discovery information to each job template about the location and credentials of other instances (called "links")

At this point, it becomes the installed software's responsibility to do thing that it needs to do. It now has been given a brand new instance running on a hardened base operating system, with a mounted persistent disk for it to store data, and has been configured with the information for forming a cluster with its peers, and connecting as a client to any other systems.

## New deployments of Zookeeper

Let's revisit each of these actions for the specific case of our 5-instance deployment of Zookeeper running on Amazon AWS.

```
> export BOSH_DEPLOYMENT=zookeeper
> bosh deploy manifests/zookeeper.yml
```

Inside `zookeeper.yml` is the description of an group of five instances, each with a 10GB persistent disk volume (we will review the contents of this file soon).

* BOSH sends requests to Amazon AWS API for five EC2 VMs, using a specified Amazon Machine Image (AMI) as the base file system/operating system
* BOSH will manage the allocation of IPs within the VPC subnet rather than using DHCP (more on networking later)
* BOSH sends requests to AWS for five EBS volumes and then attaches each one to a different EC2 VM

Each of the AWS EC2 VMs will eventually "call home" to BOSH saying that they are awake and ready.

BOSH then begins preparing them for their role of "Zookeeper" instance.

* BOSH downloads special BOSH packages of Apache Zookeeper, plus the Java JDK which is a dependency for running Zookeeper.
* BOSH downloads special BOSH job templates that describe how to configure and run a single node of Zookeeper on each instance
* BOSH provides each Zookeeper job template with the IP address, client port, quorum port and leader election port for every other member of the deployment (these are Zookeeper specific requirements to for a cluster of Zookeeper instances)

## BOSH Architecture, Part 1

In the previous sections I've made reference to a `bosh` CLI but have otherwise danced around the topic of "what is BOSH really?"

From now onwards I will stop simplistically saying "BOSH does a thing" and start to be consistently discerning about which aspect of BOSH is doing something.

Right now think of BOSH as three things:

* BOSH CLI - the `bosh` command being referenced in the earlier examples. The CLI is a client to the:
* BOSH director - an HTTP API that receives requests from the CLI and either communicates directly with instances or with your cloud infrastructure. Communication with your cloud infrastructure is via a:
* Cloud Provider Interface (CPI) - the specific implementation of how a BOSH director communicates with Amazon AWS, Google Compute, vSphere, OpenStack or any other target.

## CPI - the ultimate Cloud Provider Interface abstraction

The CLI, the director and a CPI are the basic components that bring a deployment to life on your target cloud infrastructure.

For our Zookeeper example, we began with:

```
> bosh deploy manifests/zookeeper.yml
```

The BOSH CLI loads the `zookeeper.yml` file from your local machine (which originally came from a [Github repository](https://github.com/cppforlife/zookeeper-release/blob/master/manifests/zookeeper.yml) in the [Joyful operations](#joyful-operations) section above).

The BOSH CLI forwards this file on to the BOSH director.

The BOSH director decides that it is a new deployment (it has a name that the BOSH director does not know yet). The BOSH director decides it needs to provision five new virtual machines and five persistent disks (we will investigate the contents of `zookeeper.yml` soon). The BOSH director delegates this activity to the BOSH CPI for Amazon AWS (where we are attempting to deploy Zookeeper in our example).

The BOSH CPI is a local command line application hosted inside the BOSH director. You will never need to touch it or find it or run it manually. But it can be helpful to understand its nature. A CPI - the abstraction for how an BOSH director can interact with any cloud infrastructure - is just a CLI. The BOSH director - a long-running HTTP API process - calls out to the CPI executable and invokes commands using a JSON payload. When the CPI completes its task - creating a VM, creating a disk, etc - it will return JSON with success/failure information.

For Zookeeper running on Amazon AWS, our BOSH director will be running with the AWS CPI CLI (TLA BINGO - three three letter acronyms in a row) installed on the same server. The combination of the BOSH director and a collocated CPI CLI is the magic of how a BOSH director can be configured to communicate with any cloud infrastructure.  The CPI CLIs can be written in different programming languages than BOSH director, and be maintained by different engineering teams at different companies. It is a wonderfully powerful design pattern.

This will be the last we will reference the CPIs for a long time. They exist. They allow a BOSH director to interact with any cloud infrastructure. There are many of them already implemented (Amazon AWS, Google Compute Platform, Microsoft Azure, VMWare vSphere, OpenStack, IBM SoftLayer, VirtualBox, Warden/Garden, Docker).

And you will mostly never need to know about them.

Here is the command for deploying five Amazon EC2 servers running Zookeeper, backed by Amazon EBS volumes, running inside Amazon VPC networking:

```
> bosh deploy manifests/zookeeper.yml
```

In the Amazon AWS console, your list of EC2 servers (including the BOSH director VM) might look like:

![zookeeper-deployment-aws](images/zookeeper-deployment-aws.png)

Here is the command for deploying five Google Compute VM Instances, backed by Google Compute Disks, running inside GCP networking, installed and configured to be a Zookeeper cluster:

```
> bosh deploy manifests/zookeeper.yml
```

In the Google Compute Platform console, your list of VM instances (including a NAT VM, bastion VM, and BOSH director VM) might look like:

![zookeeper-deployment-google](images/zookeeper-deployment-google.png)

Never used VMWare vSphere before? Here is the command for deploying a five ESXi virtual machines using a concept of persistent disks, on any cluster of physical servers in the world. And they will be Zookeeper:

```
> bosh deploy manifests/zookeeper.yml
```

In VMWare vCenter your deployment will not specifically look like anything. vSphere is a crazy mess to me.

For sure there are distinctions in deploying any system to any infrastructure that need to be made, but the command above is valid and will work once we have a running BOSH director configured with a CPI. That's fantastic.

## Instances

A deployment is made up of instances. Normally instances represent long-running servers on your cloud infrastructure. They can also represent "errands" - one-off tasks that are run inside of temporary servers.

The BOSH CLI makes it easy to see the list of instances for a deployment and their basic health status with the `bosh instances` command:

```
> bosh instances
Using environment '10.0.0.4' as client 'admin'

Task 1808. Done

Deployment 'zookeeper'

Instance                                          Process State  IPs
smoke-tests/dd931466-4329-46cc-971e-34aeee3f8baf  -              -
zookeeper/146229c3-648a-4280-b725-692b0092eae9    running        10.0.0.9
zookeeper/5b879e13-48b1-40f3-b6a7-ae18ef375bcb    running        10.0.0.7
zookeeper/680367f6-2303-4416-877f-8f1d68d4d978    running        10.0.0.6
zookeeper/aa88d2b6-4a94-4801-b0f4-b82d7b6c05b8    running        10.0.0.5
zookeeper/bc988e19-a8e6-41c4-bc2d-3cad00306aef    running        10.0.0.8

6 instances

Succeeded
```

This is my 5-node cluster of Zookeper, running on one cloud infrastructure or another.

We can see that all the `zookeeper` instances are `running`. There is a 6th instance `smoke-tests` which does not have a Process State. It is an errand instance which has no VM running for it at the time the instances were listed.

For now we will focus on the long-running instances, and return to errands later.

Each instance has at least one assigned IP address. The `zookeeper.yml` manifest did not need to allocate these IP addresses, rather if left this assignment to the BOSH director, the CPI, and the cloud infrastructure. It is possible to statically assign IP addresses in a deployment manifest, but ideally there are few reasons to do so. It is not a fun part of a human's day to keep track of IP allocations.

With the BOSH CLI we can also start to introspect what is running on each instance with `bosh instances --ps`:

```
Instance                                          Process    Process State  IPs
smoke-tests/dd931466-4329-46cc-971e-34aeee3f8baf  -          -              -
zookeeper/146229c3-648a-4280-b725-692b0092eae9    -          running        10.0.0.9
~                                                 zookeeper  running        -
zookeeper/5b879e13-48b1-40f3-b6a7-ae18ef375bcb    -          running        10.0.0.7
~                                                 zookeeper  running        -
zookeeper/680367f6-2303-4416-877f-8f1d68d4d978    -          running        10.0.0.6
~                                                 zookeeper  running        -
zookeeper/aa88d2b6-4a94-4801-b0f4-b82d7b6c05b8    -          running        10.0.0.5
~                                                 zookeeper  running        -
zookeeper/bc988e19-a8e6-41c4-bc2d-3cad00306aef    -          running        10.0.0.8
~                                                 zookeeper  running        -
```

Each `zookeeper` instance is running a process called `zookeeper`. Very educational information.

Let's look at another BOSH deployment that collocates more processes and is more interesting. The following deployment is for the Stark & Wayne CI system https://ci.starkandwayne.com.

```
Deployment 'concourse'

Instance                                        Process       Process State   IPs
db/af2a4827-82fd-45f1-b37b-cb7d3c6f4ab9         -             running         10.57.111.7
~                                               postgresql    running         -
haproxy/93ee7e45-8190-4f53-86b2-b1ea211f5cf9    -             running         10.57.111.6
                                                                              184.98.185.163
~                                               haproxy       running         -
web/66bac8cf-3af8-4f7e-b587-b82856f47cdc        -             running         10.57.111.8
~                                               atc           running         -
~                                               tsa           running         -
web/bed6bd94-eecc-4147-9653-5d4bdf40dcf9        -             running         10.57.111.9
~                                               atc           running         -
~                                               tsa           running         -
worker/194ac3c7-0a07-4681-ade9-afbf0e47a1a9     -             running         10.57.111.15
~                                               baggageclaim  running         -
~                                               beacon        running         -
~                                               garden        running         -
```

In this deployment we have 4 different instance groups: `db`, `haproxy`, `web` (there are two instances), and `worker` (I've show one of them above but our deployment of [Concourse](https://concourse.ci/) has many `worker` instances).

Each `worker` instance is running three processes: `baggageclaim`, `beacon`, and `garden`.

The `haproxy` instance has two IP addresses. The latter `184.98.185.163` is a public IP on the Internet. All the other IPs are private to the vSphere data centre. This `haproxy` instance is an inbound HTTP load balancer and has a statically assigned public IP for the benefit of configuring the external CloudFlare service which sits in front receiving https://ci.starkandwayne.com traffic.

The labels for processes above come directly from inside the running instances. We will now look inside the `worker` instance and match up where this information comes from.

## SSH

To access a shell session on any instance we can use `bosh ssh`:

```
> bosh ssh worker/194ac3c7-0a07-4681-ade9-afbf0e47a1a9
```

If you don't know the long UUID for an instance, and you just want to SSH into any instance in the instance group then use a numbered index such as `/0` or `/1`.

```
> bosh ssh worker/0
```

If your deployment only has a single instance then you can omit the label altogether:

```
> bosh ssh
```

If you attempt this latter command but your deployment has more than one instance you will get a red error message similar to:

```
Running SSH:
  Interactive SSH only works for a single host at a time
```

After successfully running a `bosh ssh` command you will be presented with a shell prompt like:

```
worker/194ac3c7-0a07-4681-ade9-afbf0e47a1a9:~$
```

The start of the prompt indicates that you are inside `worker/194ac3c7...`. The `~` segment means you are in the home folder (as an aside, `cd ~` will take you to the home folder on linux machines).

Each time you open an SSH shell using `bosh ssh` you will be allocated a new user account:

```
$ whoami
bosh_5510a7b92da9475
```

To investigate the processes running we will need to change to the root user:

```
worker/194ac3c7-0a07-4681-ade9-afbf0e47a1a9:~$ sudo su -
worker/194ac3c7-0a07-4681-ade9-afbf0e47a1a9:~#
```

The suffix of the prompt subtly changed from `$` to `#` to represent we are now a root user.

## Shell user prompts in examples

We now have three different shell users in examples: your local machine, a dynamically created non-root user on an instance, and the root user on an instance.

I would like to help make it obvious at all times from which machine/user a command is being run in the examples within the Ultimate Guide to BOSH.

When an example is being run from a local developer machine (that is, not inside a BOSH instance) I will prefix the commands with the greater-than symbol `>`. For example, `bosh` commands are run from outside of BOSH instances.

```
> bosh deployments
```

From inside a BOSH instance I will abbreviate the prompt to `$` in future for non-root user, and `#` for the root user. Like any good person I will try hard to stick to being a non-root user.

```
> bosh ssh
$ whoami
bosh_5510a7b92da9475
$ sudo su -
# whoami
root
```

But right now, we need to be the root user to access Monit.

## Monit process monitoring

```
# monit summary
The Monit daemon 5.2.5 uptime: 8d 3h 26m

Process 'beacon'                    running
Process 'baggageclaim'              running
Process 'garden'                    running
System 'system_localhost'           running
```

The three `Process` entries above match directly to the three `Processes` items in the `bosh instances --ps` output in the preceding section. This is where this information comes from.

Inside each VM we delegate to [Monit](https://mmonit.com/monit/) to start, stop, and monitor the health of processes.

Monit plays a small but vital role within every BOSH deployment running on Linux servers. Monit has been the process monitoring heart of BOSH instances since before BOSH was publicly open sourced in 2012. And ever since 2012, every single Product Manager of BOSH has said "we will replace Monit with something else".

If you learn about using BOSH on Windows you will discover that instead of Monit we use native Windows Services to start, stop, repair processes. But for Linux, its Monit.

The good news is that Monit has an [extensive set of configuration](https://mmonit.com/monit/documentation/monit.html#THE-MONIT-CONTROL-FILE) that you might wish to use in future to describe good process behaviour.

Let's regroup and reestablish what we know:

`bosh instances --ps` displays a list of processes and their `running` or otherwise state. This information comes directly from Monit running on each instance.

In daily life you will run `monit summary` or `bosh instances --ps` as part of debugging.

## Job templates describe processes

Earlier in [New deployments](#new-deployments) I introduced the terminology of a "job template":

> BOSH will construct configuration files for the packages and commence running the software (called "job templates")

Job templates are where we configure Monit processes, which in turn runs processes, which in turn results in a running system (of Zookeeper or Concourse or whatever our deployment is designed for). Job templates also configure the processes, describe how to start a process, and how to stop a process.

First, let's finish tying up the story of Monit processes.

Each BOSH instance has a folder `/var/vcap/jobs` containing one or more job templates.

```
# ls /var/vcap/jobs/
baggageclaim  garden  groundcrew
```

This Concourse `worker` instance is configured to run three job templates called `baggageclaim`, `garden`, and `groundcrew`.

These three job template names `baggageclaim garden groundcrew` are ALMOST the same as the Monit process names from above `baggageclaim garden beacon`. If they ever match, it is for convenience.

Each job template folder contains a single `monit` file, which in turn contains zero or more `check process <process name>` instructions.

```
# tail /var/vcap/jobs/*/monit
==> /var/vcap/jobs/baggageclaim/monit <==
check process baggageclaim
  with pidfile /var/vcap/sys/run/baggageclaim/baggageclaim.pid
  start program "/var/vcap/jobs/baggageclaim/bin/baggageclaim_ctl start"
  stop program "/var/vcap/jobs/baggageclaim/bin/baggageclaim_ctl stop"
  group vcap

==> /var/vcap/jobs/garden/monit <==
check process garden
  with pidfile /var/vcap/sys/run/garden/garden.pid
  start program "/bin/sh -c '/var/vcap/jobs/garden/bin/garden_ctl start'"
  stop program "/var/vcap/jobs/garden/bin/garden_ctl stop"

  if failed host 127.0.0.1 port 7777
    with timeout 5 seconds for 12 cycles
    then restart

  group vcap

==> /var/vcap/jobs/groundcrew/monit <==
check process beacon
  with pidfile /var/vcap/sys/run/groundcrew/beacon.pid
  start program "/var/vcap/jobs/groundcrew/bin/beacon_ctl start"
  stop program "/var/vcap/jobs/groundcrew/bin/beacon_ctl stop"
  group vcap
```

We see above that the `groundcrew` job template's `monit` file describes `check process beacon`. This is where the `beacon` name comes from. If we ever observe a problem with the `beacon` process, we now know it is configured inside the `groundcrew` job template.

As an aside, `/var/vcap/jobs/garden/monit` above shows an advanced example of a Monit control file. It includes additional rules for when to automatically trigger a restart of the `garden` process. Monit will monitor port `7777` and if it cannot connect to the application locally then it will restart the process.

Looking again at `groundcrew` job template's `monit` file:

```
check process beacon
  with pidfile /var/vcap/sys/run/groundcrew/beacon.pid
  start program "/var/vcap/jobs/groundcrew/bin/beacon_ctl start"
  stop program "/var/vcap/jobs/groundcrew/bin/beacon_ctl stop"
  group vcap
```

When Monit needs to start `beacon` it will invoke `/var/vcap/jobs/groundcrew/bin/beacon_ctl start`. That is, it invokes `/var/vcap/jobs/groundcrew/bin/beacon_ctl` and passes `start` as the first and only argument.

At this point the Monit `check process beacon` expects that the Linux Process ID (PID) of a running Linux process will be placed at `/var/vcap/sys/run/groundcrew/beacon.pid`. Monit will continuously watch that this Linux process is running. If it isn't - that is, if the process dies unexpectedly - then Monit will restart to process. That is, it will invoke `/var/vcap/jobs/groundcrew/bin/beacon_ctl start` again.

This is Monit's primary role - to monitor processes (by their PID) and ensure they are restarted if faulty or missing.

When Monit needs to stop `beacon` it will invoke `/var/vcap/jobs/groundcrew/bin/beacon_ctl stop`.

## Job templates

The `groundcrew` job template will always be located at `/var/vcap/jobs/groundcrew`. The `garden` job template will always be located at `/var/vcap/jobs/garden`.

As it happens, within our `zookeeper` example deployment each `zookeeper` instance includes a job template called `zookeeper`. This means there will be a `/vcap/vcap/jobs/zookeeper` job template folder. This folder contains a `monit` control script with `create program zookeeper`. Consistency of names - using `zookeeper` as the name of the instance group, job template, and monit process - is convenient once you understand what is going on and is a common pattern that you will see.

In summary, all job templates - the configuration of how software is configured and executed - are located on every BOSH instance around the world in the same location: `/var/vcap/jobs/`

**Conventions like this radically lower the mental challenges of support/debugging on running production systems.** You will discover BOSH has many pleasant conventions across all deployments.

Job templates must contain a `monit` file, but that `monit` file can be empty if the job template does not require any processes to be run.

Job templates will also be able to provide any configuration files used by the running processes. Some software requires configuration files. Or software might be configured with environment variables. Job templates will be written to suite the software it is configuring to run.

Let's look at the files within the `zookeeper` job template on a `zookeeper` deployment instance:

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
    ├── java -> /var/vcap/data/packages/java/050ac564c5b047616ea42d0ea451c9c753b5637c
    └── zookeeper -> /var/vcap/data/packages/zookeeper/be0fe89fd073e3bfd23d464cf3acf3bfb9bc8029
```
