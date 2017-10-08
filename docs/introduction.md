# Introduction

## WIP

I recently started writing this. If you're reading this guide now, please let me know (I'll actively ask you to review bits as I write them) and please "Watch" this repo. Perhaps I can update it via Github Releases so you can get notifications of new sections or updates. Or better jokes.

## Guide to the Guide

This Ultimate Guide to BOSH is to be read linearly. Each section will build upon the preceding sections.

The guide use sample commands and videos to show you real systems instead of expecting that you can deploy systems yourself on day 1.

Later in the guide you will deploy your own BOSH and use it to deploy systems. At that point, you will install the `bosh` command-line tool, and you will need to decide which target cloud infrastructure you will use.


## Joyful Operations

You're a professional. You're resourceful. You're a king maker. You keep your organization in the business of winning.

In past lives you might have been called: developer, sysadmin, or devops.

You're always on the lookout for better tools, better mental models, and better systems.

I'm going to show you how I do some day-to-day activities using BOSH. You get to decide if you'd like to level up your superhero status and learn how to do this too. Learning is involved. Effort. New tools. New ecosystem. I definitely think it's worth it. Let me know what you decide!

Deploy a 5-node cluster of Zookeeper to Amazon AWS:

```
git clone https://github.com/cppforlife/zookeeper-release
cd zookeeper-release
export BOSH_ENVIRONMENT=aws
export BOSH_DEPLOYMENT=zookeeper
bosh deploy manifests/zookeeper.yml
```


Sanity check that the ZooKeeper cluster is working:

```
bosh run-errand smoke-tests
```

Upgrade to new version of ZooKeeper:

```
git pull
bosh deploy manifests/zookeeper.yml
```

Upgrade the base operating system to push out critical security patches:

```
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent
bosh deploy manifests/zookeeper.yml
```

If AWS deletes one of your VMs, heal your cluster by receating a new VM, reattach the persistent disk, remount it, and restart all the processes to join the ZooKeeper node into the cluster:

```
# do nothing, this resurrection will happen automatically
```

List the health of each ZooKeeper server, including disks:

```
bosh instances --vitals
```

SSH into one of the ZooKeeper servers to check on something:

```
bosh ssh zookeeper/0
```

Run a command on each ZooKeeper server and display results:

```
bosh ssh -c '/var/vcap/jobs/zookeeper/bin/ctl status' -r
```

Tear down your ZooKeeper cluster:

```
bosh delete-deployment
```

## About the author

I did not invent BOSH. A group of engineers from the Cloud Foundry project at VMWare conceived of BOSH and built the first iteration of BOSH. BOSH is architecturally still very similar to its original public incarnation in 2012.

Nor have I ever worked on BOSH as a full-time job. The BOSH project is sponsored heavily by full-time employees from Pivotal, as well as IBM, and other Cloud Foundry Foundation members. Your joy and happiness from BOSH is in huge thanks to VMWare, Pivotal, IBM, Stark & Wayne and the vast community of contributors who have contributed in large and small to make it fantastic.

My initial role in the history of BOSH was small - I was one of the first people outside VMWare to be publicly excited about BOSH. My relationship with BOSH and its community has snowballed ever since.

In the years before BOSH, I was the VP of Technology at Engine Yard. Engine Yard was a "devops as a service" or "platform as a service" company. Or more crassly, it was a web hosting company.

Several years earlier, the Engine Yard platform had been hastily prototyped on the new AWS platform using an also-new configuration management tool called Chef. The Engine Yard platform worked and many customers' entire business ran successfully upon it to this day. The internal cost for allowing customers to continually run their applications - even if they didn't want to upgrade and maintain their code bases - was tremendous. Upgrading the Engine Yard platform was difficult. The surface area of the implicit contract we made with customers - the base operating system, available packages, the version of Chef - was vast. Every change we wanted to make needed to be considered from the perspective of 2000 different web applications, run by 2000 different development teams, running 2000 different business.

Some of our internal systems were not as automated as others. We did not have a nice way to publish new AMIs to AWS. Since we didn't publish new AMIs, we also didn't initially have a way to share them with customers. We had spent a year preparing, curating, and releasing the second-ever edition of our base AMI, when I saw BOSH for the first time. The demonstration I was shown was incredible - a small change was made to a YAML file, they run `bosh deploy`, and in the vCenter window it could be seen that all the VMs were being progressively destroyed and replaced by new ones built upon a new base machine image, with newly compiled packages for that machine image. It was also replacing VMs with bigger ones. And it was resizing the persistent disks for the databases. BOSH was incredible.

## Brief History of BOSH

I was fortunate to see that demo. I had been invited to the VMWare campus in Palo Alto, CA, on April 11, 2012, for the unveiling of "The Outer Shell" that VMWare internally had been using to deploy Cloud Foundry. The Outer Shell was called BOSH. This is an acronym for "BOSH Outer Shell." All engineers know that recursion is funny.

I invited myself to the VMware campus for two days to meet the developers of BOSH and Cloud Foundry and came away fascinated by the vast scope of problems that the BOSH team was trying to solve. The BOSH team were kind enough to either answer a question or fix BOSH so the question was void.

In 2012, I was very publicly excited about BOSH on Twitter ([@drnic](https://twitter.com/drnic)) and I gave presentations at meetups and conferences about BOSH. I also began creating new open source projects to help myself and others to use BOSH and to create new BOSH releases. I also created the first [BOSH Getting Started](https://github.com/cloudfoundry-community-attic/LEGACY-bosh-getting-started) guide.

Many of those early presentations and guides were helpful to the many people who've discovered BOSH and are using it to run production systems at huge scales.

In 2013, the BOSH project was taken over by Pivotal engineering and has been gifted to the Cloud Foundry Foundation to secure its long term success as an open source, open community project. Thanks to Pivotal, IBM and other members of the Cloud Foundry Foundation, the BOSH project has received huge consistent investment to this day.

There are many people in the history of BOSH who have directly made BOSH what it is, actively sponsored its investment, or evangelised it.

* James Watters, SVP Product at Pivotal, has been the loudest cheerleader of BOSH on the planet
* Ferran Rodenas, has been my BOSH friend since 2012 and who created some of the greatest BOSH community contributions, including [docker-boshrelease](https://github.com/cloudfoundry-community/docker-boshrelease)
* James Bayer, VP Project Management at Pivotal, was the creator of the Clam logo for BOSH. I love the clam.
* Dmitriy Kalinin, Product Manager for BOSH, has been driving his incredible vision for BOSH
* Original BOSH team at VMware - Mark Lucovsky, Vadim Spivak, Oleg Shaldybin, Martin Englund - who had the original vision and execution to create the ultimate tool for release engineering, deployment, lifecycle management, and monitoring of distributed systems.
* Stark & Wayne staff - I've been ever so lucky to have started an organisation that has attracted so many wonderful team members around the world, who've gone on to continuously expand the BOSH ecosystem with releases, tools, and 200+ blog posts.

## BOSH in Production

BOSH is the core technology to Pivotal Ops Manager and its Pivotal Network delivery system for complex on-premise software systems. BOSH is the deployment technology used behind the scenes for [Pivotal Web Services](https://run.pivotal.io) which runs upon AWS.

BOSH is the deployment technology used behind the scenes by [IBM BlueMix](https://www.ibm.com/cloud-computing/bluemix/) on its Soft Layer infrastructure.

BOSH is the deployment technology used by GE [Predix](https://www.predix.io/) which runs on various clouds and data centres.

BOSH is the deployment technology used by Swisscom [appCloud](https://developer.swisscom.com/) which runs inside Swisscom data centres.

These are huge companies who have small teams running huge production systems using BOSH.

On the smaller end - our consultancy [Stark & Wayne](https://www.starkandwayne.com)  - uses BOSH to run a variety of our internal systems across vSphere, AWS and Google Compute. (The rest of our systems run upon Cloud Foundry itself, such as https://www.starkandwayne.com and https://www.starkandwayne.com/blog).

## Why Write the Ultimate Guide to BOSH?

BOSH has been my not-so-secret weapon since 2012. Yet, you might not yet be using BOSH.

I don't want to write a book.

I wrote a PhD thesis in 2001 and six people read it (two supervisors, three judges, and myself). Fortunately this is enough people to be awarded a doctorate. Unfortunately, it doesn't really qualify as "sharing knowledge."

I wrote my first blog post in 2006 and when I quickly reached seven subscribers; I knew I'd found my preferred medium for sharing.

I enjoy the continuous publication and feedback loop of blogging, and of sharing open source projects. I enjoy a relaxed writing style.

I don't want to write a "book" book.

I want to share all the wonders of BOSH with you. I want you to use BOSH. I want you to feel great using BOSH. I want you to feel like a superhero. I want you to convince your friends and colleagues to use BOSH. I want you to help me evangelise BOSH.

I also want you to switch to Queen's English, learn more about Australia, and to use the Oxford comma.

## Additional Sources of Information

In addition to this Ultimate Guide to BOSH, there are some other sources of factual knowledge and tutorials.

[BOSH documentation website](https://bosh.io/docs/) is very thorough and new features of BOSH now regularly appear simultaneously in this documentation site.

Maria Shaldibina's [A Guide to Using BOSH](https://mariash.github.io/learn-bosh/) is a short step-by-step tutorial that includes provisioning BOSH within a local VirtualBox machine and the common commands for deploying systems.

Duncan Winn's [Cloud Foundry: The Definitive Guide](http://shop.oreilly.com/product/0636920042501.do) includes an entire chapter, "BOSH All the Things," guided toward helping you deploy and operate Cloud Foundry.

Stark & Wayne's own blog (["bosh" tag](https://www.starkandwayne.com/blog/tag/bosh/)) has over fifty articles, tutorials, and tiny tips on BOSH.

BOSH is an open source project. You can read the source code and learn how it works. A selection of repositories include:

* https://github.com/cloudfoundry/bosh-cli - the `bosh` CLI
* https://github.com/cloudfoundry/bosh - the BOSH director
* https://github.com/cloudfoundry/bosh-agent - the BOSH agent
* https://github.com/cloudfoundry/bosh-deployment - manifests for various permutations of deploying your own BOSH director
