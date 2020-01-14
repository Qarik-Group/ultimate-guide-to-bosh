# Introduction

## WIP

I recently started writing this. If you're reading this guide now, please let me know (I'll actively ask you to review bits as I write them) and please "Watch" this repo. Perhaps I can update it via Github Releases so you can get notifications of new sections or updates. Or better jokes.

## Guide to the Guide

This Ultimate Guide to BOSH is to be read linearly. Each section will build upon the preceding sections.

The guide uses sample commands and videos to show you real systems instead of expecting that you can deploy systems yourself on day 1.

Later in the guide you will deploy your own BOSH and use it to deploy systems. At that point, you will install the `bosh` command-line tool, and you will need to decide which target cloud infrastructure you will use.

If you'd like to share a section with someone, wave your mouse cursor over the heading and a permanent link icon will appear. You can click it or copy it. It is my invisible gift to you so you can give to others.

If you'd like to fix some spelling, some grammar, or help out with the guide in some way, there is an "edit" button at the top right of each page. This will link you to the Github project page and let you submit a pull request. Good things happen to people who submit pull requests.

Each text block has a copy-and-paste icon. Click it to copy the text block into your clipboard.

## Joyful Operations

You're a professional. You're resourceful. You're a king maker. You keep your organisation in the business of winning.

In past lives you might have been called a developer, a sysadmin, or a devops engineer.

You're always on the lookout for better tools, better mental models, and better systems.

I'm going to show you how I do some day-to-day activities using BOSH. You get to decide if you'd like to level up your superhero status and learn how to do this too. Learning is involved. Effort. New tools. New ecosystem. I definitely think it's worth it. Let me know what you decide!

Deploy a 5-node cluster of Zookeeper to Amazon AWS:

```
git clone https://github.com/cppforlife/zookeeper-release
export BOSH_ENVIRONMENT=aws
export BOSH_DEPLOYMENT=zookeeper
bosh deploy zookeeper-release/manifests/zookeeper.yml
```


Sanity check that the ZooKeeper cluster is working:

```
bosh run-errand smoke-tests
```

Check the status of each node in the ZooKeeper cluster:

```
bosh run-errand status
```

Upgrade to new version of ZooKeeper:

```
git pull
bosh deploy zookeeper-release/manifests/zookeeper.yml
```

Upgrade the base operating system to push out critical security patches:

```
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent
bosh deploy zookeeper-release/manifests/zookeeper.yml
```

If AWS deletes one of your VMs, heal your cluster by recreating a new VM, reattach the persistent disk, remount it, and restart all the processes to join the ZooKeeper node into the cluster:

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

Tear down your ZooKeeper cluster, but retain the persistent disks for five days and then delete them:

```
bosh delete-deployment
```

## What is BOSH?

BOSH is a project of the Cloud Foundry Foundation. It was originally created to help the developers of Cloud Foundry to absolutely describe and test each commit and each release; and to help the site reliability engineers (SREs) tasked with running Cloud Foundry as a service.

Cloud Foundry has a micro-services architecture - bespoke applications written in Ruby, Java, C, and Golang - combined with stateful data services such as PostgreSQL, Redis, and local disks for storing user-uploaded application code. The developers wanted to work with the SREs to reduce the time of upgrades to new releases, to reduce the time between new releases, to reduce the time to deploy security fixes, and to help SREs and developers communicate about issues in production.

TODO twitter joke about laptop going into production https://twitter.com/oising/status/298464920717099009

The solution required addressing the following challenges:

* to have an absolute declaration of what specific versions of all bespoke and upstream projects combined together to form a "release";
* to own responsibility for the lifecycle of the underlying infrastructure upon which Cloud Foundry would run, including healing activities after infrastructure failures;
* to own responsibility for pushing out security patches to the base operating systems, the bespoke code, and the upstream dependencies;
* to give developers and SREs the same tool to prevent "it works on my machine" scenarios.

The "tool" that implemented this solution is a running server - a BOSH environment - which:

* receives requests from operators, who primarily use the `bosh` CLI;
* interacts with cloud infrastructures to provision and de-provision cloud servers and disks;
* interacts with running servers to configure and monitor long-running processes;
* monitors the health of cloud servers and performs remedial actions to recreate or fix any missing infrastructure

Today, small teams and large businesses are using BOSH to run a wide variety of systems including, but not limited to, platforms such as Cloud Foundry, Kubernetes, DC/OS, Docker, Habitat, and Nomad. It is used to run database clusters. It can run source control systems. It can run web applications.

Some teams use it only for its provisioning/infrastructure lifecycles features, and use their own packaging, container, and configuration management tools.

Some teams put their BOSH environment behind an API, such as the Open Service Broker API, and dynamically provision and de-provision entire systems on demand. For example, [Pivotal Container Services](https://pivotal.io/platform/pivotal-container-service) is an API driven system to deploy entire Kubernetes clusters, all using BOSH.

## Help Users Run Long-Term Systems

The publisher of software ultimately cares about their users **running** their software. At first their users will trial the software: to feel it work, to believe that the software might solve the user's problems, and to get a feel for how to run the software in production. That's Day 1. Day 2 is every day onwards.

Any system that runs for a few years will go through the following events:

* **Upgrades to primary software.** For example, upgrading to new Apache ZooKeeper versions within a running cluster of ZooKeeper.
* **Upgrades to secondary software dependencies.** For example, ZooKeeper runs upon Java so a ZooKeeper cluster will need to upgrade to new Java versions. We will need to perform dependency upgrades to, at the very least, push out critical security fixes. We might always want to use newer dependencies if they improve performance, reduce CPU or RAM usage, or have additional features. We might also need to upgrade dependencies if our primary software ceases to support ageing dependencies.
* **Upgrades to operating system kernels and core software.** If your system is running upon an Ubuntu host machine and/or an Ubuntu container image, then you will need to upgrade or replace the base host machine and/or the containers as soon as possible after the security notices are published. For example, see [Ubuntu Security Notices](https://usn.ubuntu.com/usn/). Security vulnerabilities are continually discovered within all layers of each operating system distribution and pushed out throughout the year.
* **Resizing of the infrastructure.** As a system becomes increasingly used by end users, or as it acquires more data, it will need to grow. Persistent disks will need to be enlarged. Host machines will need to be replaced by larger machines. Container constraints may need to be increased.
* **Healing of the infrastructure.** Host machines disappear. Physical disks can corrupt. Your users' systems will need to heal.
* **Debugging of the software and its dependencies.** The brilliance of self-hosting your own software as a service is that only your team needs to debug your software. Conversely, when you distribute your software to other operations teams, it will be those teams that need to debug the entire system. They will want to be able to perform remediation if they can. If not, next they will want to help the software publisher with debugging and resolution. They want their system to be running and healthy. They want their own end users to be happy.
* **Monitoring of the software and its dependencies.** End users of running systems don't enjoy being the first and only line of notification of downtime. They want to know how they can monitor the running software and its dependencies, they want to know what to look for to actively detect imminent misbehaviour, and they definitely want to know if the system is currently failing.
* **Backup/recovery of data.** If the operators of your software discover that they need to restore data from archives, how will they do that? How much data will be lost? How bad is this situation for the end users of the system? How bad is the situation for all stakeholders? To put it another way, if the operators of your software lose data, what apology email will they be sending out to their own users? What letter will their chairman be sending out to shareholders?

Your software will need to support each of these lifecycle events. It is even better if you help your end users to make great choices about how they run your software in production.

{==

If you author your software and its BOSH release in parallel, you will be giving your end users the ability to run a system, rather than just some software to install.

==}

Publishing a BOSH release is like taking your users on a packaged, guided tour of ancient Roman ruins.

Shipping them installable software is like leaving a Post-It Note on their fridge, "We went to Rome. You should go some day."

Publishing a BOSH release shows that you care about your users' long-term success in running the system.

Shipping installable software is not much better than saying "Works on my machine."

## About the Author

I did not invent BOSH. A group of engineers from the Cloud Foundry project at VMWare conceived of BOSH and built the first iteration of BOSH. BOSH is architecturally still very similar to its original public incarnation in 2012.

Nor have I ever worked on BOSH as a full-time job. The BOSH project is sponsored heavily by full-time employees from Pivotal, as well as IBM, and other Cloud Foundry Foundation members. Your joy and happiness from BOSH is in huge thanks to VMWare, Pivotal, IBM, Stark & Wayne and the vast community of contributors who have contributed in large and small ways to make it fantastic.

My initial role in the history of BOSH was small - I was one of the first people outside VMWare to be publicly excited about BOSH. My relationship with BOSH and its community has snowballed ever since.

In the years before BOSH, I was the VP of Technology at Engine Yard. Engine Yard was a "devops as a service" or "platform as a service" company. Or more crassly, it was a web hosting company.

Several years earlier, the Engine Yard platform had been hastily prototyped on the new [Amazon Web Services](https://en.m.wikipedia.org/wiki/Amazon_Web_Services) (AWS) platform using an also-new configuration management tool called Chef. The Engine Yard platform worked and many customers' entire business run successfully upon it to this day. The internal cost of allowing customers to continually run their applications - even if they didn't want to upgrade and maintain their code bases - was tremendous. Upgrading the Engine Yard platform was difficult. The surface area of the implicit contract we made with customers - the base operating system, available packages, the version of Chef - was vast. Every change we wanted to make needed to be considered from the perspective of 2000 different web applications, run by 2000 different development teams, running 2000 different businesses.

Some of our internal systems were not as automated as others. We did not have a nice way to publish new AMIs to AWS. Since we didn't publish new AMIs, we also didn't initially have a way to share them with customers. We had spent a year preparing, curating, and releasing the second-ever edition of our base AMI, when I saw BOSH for the first time. The demonstration I was shown was incredible - a small change was made to a YAML file, they run `bosh deploy`, and in the vCenter window it could be seen that all the VMs were being progressively destroyed and replaced by new ones built upon a new base machine image, with newly compiled packages for that machine image. It was also replacing VMs with bigger ones. And it was resizing the persistent disks for the databases. BOSH was incredible.

## Brief History of BOSH

I was fortunate to see that demo. I had been invited to the VMWare campus in Palo Alto, CA, on April 11, 2012, for the unveiling of "The Outer Shell" that VMWare internally had been using to deploy Cloud Foundry. The Outer Shell was called BOSH. This is an acronym for "BOSH Outer Shell." All engineers know that recursion is funny.

I invited myself to the VMware campus for two days to meet the developers of BOSH and Cloud Foundry and came away fascinated by the vast scope of problems that the BOSH team was trying to solve. The BOSH team were kind enough to either answer a question or fix BOSH so the question was void.

In 2012, I was very publicly excited about BOSH on Twitter ([@drnic](https://twitter.com/drnic)) and I gave presentations at meetups and conferences about BOSH. I also began creating new open source projects to help myself and others to use BOSH and to create new BOSH releases. I also created the first [BOSH Getting Started](https://github.com/cloudfoundry-community-attic/LEGACY-bosh-getting-started) guide.

Many of those early presentations and guides were helpful to the many people who've discovered BOSH and are using it to run production systems at huge scales.

In 2013, the BOSH project was taken over by Pivotal engineering and has been gifted to the Cloud Foundry Foundation to secure its long term success as an open source, open community project. Thanks to Pivotal, IBM, and other members of the Cloud Foundry Foundation, the BOSH project has been receiving huge consistent investment to this day.

There are many people in the history of BOSH who have directly made BOSH what it is, actively sponsored its investment, or evangelised it.

* James Watters, SVP Product at Pivotal, has been the loudest cheerleader of BOSH on the planet.
* Ferran Rodenas, has been my BOSH friend since 2012 and who created some of the greatest BOSH community contributions, including [docker-boshrelease](https://github.com/cloudfoundry-community/docker-boshrelease).
* James Bayer, VP Project Management at Pivotal, was the creator of the Clam logo for BOSH. I love the clam.
* Dmitriy Kalinin, Product Manager for BOSH, has been driving his incredible vision for BOSH.
* Original BOSH team at VMware - Mark Lucovsky, Vadim Spivak, Oleg Shaldybin, Martin Englund - who had the original vision and execution to create the ultimate tool for release engineering, deployment, lifecycle management, and monitoring of distributed systems.
* Stark & Wayne staff - I've been ever so lucky to have started an organisation that has attracted so many wonderful team members around the world, who've gone on to continuously expand the BOSH ecosystem with releases, tools, and 200+ blog posts.

## BOSH in Production

BOSH is the core technology to Pivotal Ops Manager and its Pivotal Network delivery system for complex on-premise software systems. BOSH is the deployment technology used behind the scenes for [Pivotal Web Services](https://run.pivotal.io) which runs upon AWS.

BOSH is the deployment technology used behind the scenes by [IBM Cloud](https://www.ibm.com/cloud/).

BOSH is the deployment technology used by GE [Predix](https://www.predix.io/) which runs on various clouds and data centres.

BOSH is the deployment technology used by Swisscom [appCloud](https://developer.swisscom.com/) which runs inside Swisscom data centres.

These are huge companies who have small teams running huge production systems using BOSH.

On the smaller end - our consultancy [Stark & Wayne](https://www.starkandwayne.com)  - uses BOSH to run a variety of our internal systems across vSphere, AWS and Google Compute. (The rest of our systems run upon Cloud Foundry itself, such as https://www.starkandwayne.com and https://www.starkandwayne.com/blog).

## Why Write the Ultimate Guide to BOSH?

I have been using BOSH since 2012. I have yet to find any system like BOSH, despite the fast paced and highly competitive fields of Platforms, Containers, DevOps, and Cloud Infrastructure.

I didn't want to write a book. I had written a PhD thesis in 2001 and six people read it (two supervisors, three judges, and myself). Fortunately, this is enough people to be awarded a doctorate. Unfortunately, it doesn't really qualify as "sharing knowledge."

I wrote my first blog post in 2006, and, once I quickly reached seven subscribers, I knew I'd found my preferred medium for sharing.

I enjoy the continuous publication and feedback loop of blogging, and of sharing open source projects. I enjoy a relaxed writing style.

I didn't want to write a "book" book. Not a boring book. Not a reference book.

I want to share all the wonders of BOSH with you. I want you to use BOSH. I want you to feel great using BOSH. I want you to feel like a superhero. I want you to convince your friends and colleagues to use BOSH. I want you to help me evangelise BOSH.

I want this to be a book that you enjoy reading. I hope you enjoy my manner of introducing each topic, and how the topics segue as we gradually introduce more complex topics. I hope you enjoy my sense of humour.

I also want you to switch to Queen's English, to visit me in Australia, and to use the Oxford comma.

## Additional Sources of Information

In addition to this Ultimate Guide to BOSH, there are some other sources of factual knowledge and tutorials.

[BOSH documentation website](https://bosh.io/docs/) is very thorough and new features of BOSH now regularly appear simultaneously in this documentation site.

Maria Shaldibina's [A Guide to Using BOSH](https://mariash.github.io/learn-bosh/) is a short step-by-step tutorial that includes provisioning BOSH within a local VirtualBox machine and the common commands for deploying systems.

Duncan Winn's [Cloud Foundry: The Definitive Guide](http://shop.oreilly.com/product/0636920042501.do) includes four chapters introducing BOSH.

Stark & Wayne's own blog (["bosh" tag](https://www.starkandwayne.com/blog/tag/bosh/)) has over fifty articles, tutorials, and tiny tips on BOSH.

BOSH is an open source project. You can read the source code and learn how it works. A selection of repositories include:

* https://github.com/cloudfoundry/bosh-cli - the `bosh` CLI
* https://github.com/cloudfoundry/bosh - the BOSH director
* https://github.com/cloudfoundry/bosh-agent - the BOSH agent
* https://github.com/cloudfoundry/bosh-deployment - manifests for various permutations of deploying your own BOSH director
