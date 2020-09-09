# Instances

A deployment is made up of instances. Normally, instances represent long-running servers on your cloud infrastructure. They can also represent "errands" - one-off tasks that can  be run inside of temporary servers.

The BOSH CLI makes it easy to see the list of instances for a deployment and their basic health status with the `bosh instances` command:

``` hl_lines="1"
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

This is my 5-node cluster of ZooKeeper, running on one cloud infrastructure or another.

We can see that all the `zookeeper` instances are `running`. There is a sixth instance `smoke-tests` which does not have a Process State. It is an errand instance which has no VM running for it at the time the instances were listed.

For now, we will focus on the long-running instances, and return to errands later.

Each instance has at least one assigned IP address. The `zookeeper.yml` manifest did not need to allocate these IP addresses, rather it left this assignment to the BOSH director, the CPI, and the cloud infrastructure. It is possible to statically assign IP addresses in a deployment manifest, but ideally there are few reasons to do so. It is not a fun part of a human's day to keep track of IP allocations.

With the BOSH CLI we can also start to introspect what is running on each instance with `bosh instances --ps`:

``` hl_lines="3 4"
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

```yaml hl_lines="15 16 17 18"
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

In this deployment we have four different instance groups: `db`, `haproxy`, `web` (there are two instances), and `worker` (I've shown one of them above but our deployment of [Concourse](https://concourse.ci/) has many `worker` instances).

Each highlighted `worker` instance is running three processes: `baggageclaim`, `beacon`, and `garden`.

The `haproxy` instance has two IP addresses. The latter `184.98.185.163` is a public IP on the Internet. All the other IPs are private to the vSphere data centre. This `haproxy` instance is an inbound HTTP load balancer and has a statically assigned public IP for the benefit of configuring the external CloudFlare service which sits in front receiving https://ci.starkandwayne.com traffic.

The labels for processes above come directly from inside the running instances. We will now look inside the `worker` instance and match up where this information comes from.

## Instances and Cloud Servers

Throughout this Ultimate Guide to BOSH we will refer to Instances and Cloud Servers. They are related but different. BOSH instances are a permanent component of a BOSH deployment. The terminology "Cloud Server" will be a generic reference to the compute infrastructure where processes are running.

In your environments, Cloud Servers are the actual virtual machines, physical machines (some CPIs work with bare machines), or containers (some CPIs provision Linux containers). Throughout the Ultimate Guide to BOSH I will try to use the generic terminology "cloud server" to represent the unit of running infrastructure. In your day-to-day operations you will normally call them VMs, machines, or containers because you will know what infrastructure you're using.

For example, in most production environments you will be using a CPI that manages virtual machines (AWS, GCP, Microsoft Azure, VMWare vSphere, OpenStack).

There is popular version of a BOSH director where its CPI provisions Linux containers.

During the life of a deployment, one Instance will map to one Cloud Server most of the time. Sometimes the Cloud Server might be accidentally missing or being deliberately replaced.


A BOSH deployment with five `zookeeper` instances will always display five results with the `bosh instances` command.

## SSH

To access a shell session on any instance we can use `bosh ssh`:

```
> bosh ssh worker/194ac3c7-0a07-4681-ade9-afbf0e47a1a9
```

If you don't know the long UUID for an instance, and you just want to SSH into any instance in the instance group, then use a numbered index such as `/0` or `/1`.

```
> bosh ssh worker/0
```

If your deployment only has a single instance then you can omit the label altogether:

```
> bosh ssh
```

If you attempt this latter command but your deployment has more than one instance, you will get a red error message similar to:

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

## Shell User Prompts in Examples

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

If an example does not include a shell prompt (`>`, `$`, or `#`) then it is the contents of a shell script or the output from a command.


**If you `bosh ssh` into a production system and have changed to `root` user then please place a large cowboy hat on your head. A big one. Everyone needs to know you're a cowboy.**

Instead, try changing to the user for the processes you are working on. On most BOSH deployments, the convention is to use a user called `vcap`. To change from your random `bosh_xxx` user to `vcap`:

```
$ sudo su - vcap
$ whoami
vcap
```

Right now we need to be the `root` user to inspect Monit. Generally, you should not need to be `root` user.

## Monit Process Monitoring

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

Monit plays a small but vital role within every BOSH deployment running on Linux servers. Monit has been the process monitoring heart of BOSH instances since before BOSH was publicly open sourced in 2012. And ever since 2012, every single Product Manager of BOSH has said, "We will replace Monit with something else."

If you learn about using BOSH on Windows, you will discover that instead of Monit we use native Windows Services to start, stop, and repair processes. But for Linux, it's Monit.

The good news is that Monit has an [extensive set of configuration](https://mmonit.com/monit/documentation/monit.html#THE-MONIT-CONTROL-FILE) that you might wish to use in the future to describe good process behavior.

Let's regroup and reestablish what we know:

`bosh instances --ps` displays a list of processes and their `running` or otherwise state. This information comes directly from Monit running on each instance.

In daily life, you will run `monit summary` or `bosh instances --ps` as part of debugging.

## Job Templates Describe Processes

Earlier in [New Deployments](/deployments/#new-deployments) I introduced the terminology of a "job template":

> BOSH will construct configuration files for the packages and commence running the software (called "job templates")

Job templates are where we configure processes and how monit should interact with them. As we know, monit runs processes which represent specific software such as ZooKeeper, Concourse, or whatever your deployment is designed for.

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

## Job Templates

The `groundcrew` job template will always be located at `/var/vcap/jobs/groundcrew`. The `garden` job template will always be located at `/var/vcap/jobs/garden`.

As it happens, within our `zookeeper` example deployment each `zookeeper` instance includes a job template called `zookeeper`. This means there will be a `/var/vcap/jobs/zookeeper` job template folder. This folder contains a `monit` control script with `create program zookeeper`. Consistency of names - using `zookeeper` as the name of the instance group, job template, and Monit process - is convenient once you understand what is going on and is a common pattern that you will see.

In summary, all job templates - the configuration of how software is configured and executed - are located on every BOSH instance around the world in the same location: `/var/vcap/jobs/`

**Conventions like this radically lower the mental challenges of support/debugging on running production systems.** You will discover BOSH has many pleasant conventions across all deployments.

Job templates must contain a `monit` file, but that `monit` file can be empty if the job template does not require any processes to be run.

Job templates will also be able to provide any configuration files used by the running processes. Some software requires configuration files. Or software might be configured with environment variables. Job templates will be written to suit the software it is configuring to run.

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
    ├── java
    └── zookeeper
```

As discussed above, the `monit` file is the entry point for a job template being used to run Linux processes. The contents of this file tell Monit what command to run to start or stop any Linux processes that are required:

```
check process zookeeper
  with pidfile /var/vcap/sys/run/zookeeper/pid
  start program "/var/vcap/jobs/zookeeper/bin/ctl start"
  stop program "/var/vcap/jobs/zookeeper/bin/ctl stop"
  group vcap
```

Within our `/var/vcap/jobs/zookeeper` job template directory the `bin/ctl` script has a `start` command, and Monit expects a PID file to be placed at `/var/vcap/sys/run/zookeeper/pid` (a file location outside of the job template directory that we will revisit soon).

The abbreviated `bin/ctl` shell script showing the `start` and `stop` subcommands is below (the full original source code is [online](https://github.com/cppforlife/zookeeper-release/blob/master/jobs/zookeeper/templates/ctl.erb))

```bash
case $1 in
  start)
    # Hidden: setup of other env vars
    export ZOOCFGDIR=/var/vcap/jobs/zookeeper/config

    # Hidden: create log/pid folders
    echo $$ > /var/vcap/sys/run/zookeeper/pid

    exec chpst -u vcap:vcap \
      /var/vcap/packages/zookeeper/bin/zkServer.sh start-foreground \
      >>/var/vcap/sys/log/zookeeper/stdout.log \
      2>>/var/vcap/sys/log/zookeeper/stderr.log
    ;;

  stop)
    if [ -f $PIDFILE ]; then
      kill -9 `cat $PIDFILE` || true
      rm -f $PIDFILE
    fi
    ;;
esac
exit 0
```

This `bin/ctl` is a very common implementation of a Monit start/stop wrapper script that you will see in most BOSH job templates. A `case` statement that takes `start` and `stop` from the first argument passed to `bin/ctl` and runs one of two different code paths:

* The `start` subcommand will run a single Linux process.

* The `stop` subcommand will terminate the Linux process, if one is running.

Some software is capable of running as a background/daemon process and managing its own PID file. Others do not manage their own PID. Some software can do either and the BOSH job template author will have to decide in which mode to run the software.

In the `zookeeper` example above the `bin/ctl start` command is creating the PID file:

```
echo $$ > /var/vcap/sys/run/zookeeper/pid
```

The expression `$$` is PID of the current shell (the running `bin/ctl` script). So the command above is inserting the current running script's PID into a file. Importantly, this file is the same as the `check process zookeeper with pidfile /var/vcap/sys/run/zookeeper/pid` from the `monit` file above.

The `bin/ctl` running script then uses `exec` to replace the current running shell (the wrapper `bin/ctl` script) with a new command `/var/vcap/packages/zookeeper/bin/zkServer.sh`

If the script only used `echo $$ > pid` and `exec run-software` then the software would be run with escalated root user privileges. Instead, the ZooKeeper application will be run as a restricted `vcap` user and `vcap` group using the `chpst` command.

Combined together we have a common pattern in many BOSH job templates:

```
echo $$ > path/to/pidfile

exec chpst -u user:group run-something-in-foreground
```

For the `zookeeper` example, the ZooKeeper software was run using a shell script provided by the Apache ZooKeeper package (`zkServer.sh`):

```
echo $$ > /var/vcap/sys/run/zookeeper/pid

exec chpst -u vcap:vcap \
  /var/vcap/packages/zookeeper/bin/zkServer.sh start-foreground
```

An alternate pattern to using `exec chpst -u vcap:vcap run-something` that you might find is the use of `su - vcap -c "run-something"`.

## Running Processes, Summary-in-Progress

We have now covered the most important primary concepts of BOSH: it will provision virtual machines on your target cloud infrastructure, it will install job templates that describe how software is to be started and stopped, and it uses Monit to continuously monitor the health of these processes.

We can see the overall health of a BOSH deployment using:

```
> bosh instances --ps
```

From inside a BOSH instance you can see similar information:

```
# monit summary
```

If you want to see all the Linux processes currently running on a BOSH instance, try the `ps` command:

```
$ ps axwwf
```

This command will show all running processes in wide screen mode. You will see the full explicit commands that were run to start Linux processes. From a `zookeeper` instance this will include this Java monster command:

```
7913 ?        S<l    0:08 /var/vcap/packages/java/jdk/bin/java -Dzookeeper.log.dir=/var/vcap/sys/log/zookeeper -Dzookeeper.root.logger=INFO,CONSOLE,ROLLINGFILE -cp /var/vcap/packages/zookeeper/bin/../build/classes:/var/vcap/packages/zookeeper/bin/../build/lib/*.jar:/var/vcap/packages/zookeeper/bin/../lib/slf4j-log4j12-1.6.1.jar:/var/vcap/packages/zookeeper/bin/../lib/slf4j-api-1.6.1.jar:/var/vcap/packages/zookeeper/bin/../lib/netty-3.10.5.Final.jar:/var/vcap/packages/zookeeper/bin/../lib/log4j-1.2.16.jar:/var/vcap/packages/zookeeper/bin/../lib/jline-0.9.94.jar:/var/vcap/packages/zookeeper/bin/../zookeeper-3.4.10.jar:/var/vcap/packages/zookeeper/bin/../src/java/lib/*.jar:/var/vcap/jobs/zookeeper/config:/var/vcap/packages/zookeeper/lib/slf4j-log4j12-1.6.1.jar:/var/vcap/packages/zookeeper/lib/slf4j-api-1.6.1.jar:/var/vcap/packages/zookeeper/lib/netty-3.10.5.Final.jar:/var/vcap/packages/zookeeper/lib/log4j-1.2.16.jar:/var/vcap/packages/zookeeper/lib/jline-0.9.94.jar:/var/vcap/packages/zookeeper/dist-maven/zookeeper-3.4.10.jar:/var/vcap/packages/zookeeper/dist-maven/zookeeper-3.4.10-tests.jar:/var/vcap/packages/zookeeper/dist-maven/zookeeper-3.4.10-sources.jar:/var/vcap/packages/zookeeper/dist-maven/zookeeper-3.4.10-javadoc.jar: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false org.apache.zookeeper.server.quorum.QuorumPeerMain /var/vcap/jobs/zookeeper/config/zoo.cfg
```

 Perhaps seeing all that will help you debug ZooKeeper. Perhaps it will prompt you to change professions.

## Logs

The previous sections have shown you what instances and processes are running in a system. In order to learn more about what all the software is doing as it is running you will need to be able to view the logs of all the running processes.

The BOSH CLI is very convenient for getting started with logs. You can stream all the logs from all the job templates from all the instances in a deployment with one beautiful command:

```
bosh logs --follow
```

The `bosh logs --follow` flag also has the short alias `bosh logs -f`.

Some systems only emit logs if interesting things are happening. These can be pleasant logs to view.

Unfortunately, other systems can emit logs with a frequency which might infer they have nothing better to do. Your screen might be continuously populated with new logs such that it is impossible to understand anything.

You have some options to survive log overload.

* Pipe the logs to `grep` to restrict what you see

    For example, if you're looking for log lines that contain `stderr` string:

    ```
    > bosh logs -f | grep stderr
    ```

* Use `bosh logs` (without the `--follow` or `-f` following flag) and all the current logs will be downloaded to your local machine.

* Use `bosh logs --job name` flag to try scoping down the subset of instances for which you want to see logs.

* Use an external log collector where you may have additional features for searching/filtering logs.

* SSH into each instance and inspect the logs directly

The reason that `bosh logs` can find all the process logs is a convention used amongst all BOSH deployments to place their process logs in the same subfolder `/var/vcap/sys/log`.

Note, this log location is different from many other Linux conventions, such as `/var/log`. We will discuss the alternate filesystem layout of BOSH instances soon.

Within a BOSH instance you can try watching all the logs using `tail`:

```
# tail -f /var/vcap/sys/log/{*.log,*/*.log}
```

If we revisit the `bin/ctl start` for `zookeeper` above, we can see how it is storing some logs:

```
exec chpst -u vcap:vcap \
  /var/vcap/packages/zookeeper/bin/zkServer.sh start-foreground \
  >>/var/vcap/sys/log/zookeeper/stdout.log \
  2>>/var/vcap/sys/log/zookeeper/stderr.log
```

Any regular output will be appended to a file `/var/vcap/sys/log/zookeeper/stdout.log`, and any error or warning messages will be appended to a file `/var/vcap/sys/log/zookeeper/stderr.log`.

## Linux Pipe Operators

The `>>` and `2>>` symbols in the example above are Linux pipe operators. They are similar to the pipe operators `>` and `2>`.

Every Linux process can emit two pipes called "standard out" (stdout) and "standard error" (stderr). Typically applications will print any regular output for the application to stdout and emit errors and warnings to stderr. This means that any text output to either stdout or stderr has some additional metadata - was it regular output or was it an error/warning message.

When we run applications in a terminal or SSH session, anything appearing in these two pipes will be displayed on the screen.

When we use Monit to run applications, there is no terminal screen for a human to see. The contents of these two pipes might be ignored. That is, we would not know if applications were emitting good output or error messages.

One option is to explicitly redirect these two pipes to local files. That is what is happening in the `bin/ctl start` example above.

* `>>` will append any stdout pipe contents to the file `/var/vcap/sys/log/zookeeper/stdout.log`
* `2>>` will append any stderr pipe contents to the file `/var/vcap/sys/log/zookeeper/stderr.log`

You will now be able to look inside `/var/vcap/sys/log/zookeeper/stderr.log` and see only the error/warning messages.

If the pipe operators `>` and `2>` are used, then new log files will be created when the application is executed. This will effectively erase any previous logs that might have explained the history of the process before it was restarted. Therefore you will typically see the append operators `>>` and `2>>`.

## Short-Lived Infrastructure

We cannot assume that files or data written to the filesystem will be available forever. Cloud infrastructure providers will not make any commitments to the longevity of their virtual machines and the associated root file systems.

The ephemeral lifespan of virtual machines needs to be considered and fortunately, BOSH is here to help.

If the BOSH director ever discovers that an instance has disappeared or has become unresponsive, it will fix the problem. The BOSH director will resurrect any missing instances.

**Resurrection of infrastructure is an incredibly powerful feature of BOSH that makes it essential to your organization.** And your personal life.

If the original server is missing (perhaps the cloud provider lost the host machine or perhaps someone with admin permissions accidentally deleted it), the BOSH director creates a new server.

Alternately, if the cloud provider API believes the original server still exists but the BOSH director cannot access it, then the BOSH director will request that the server be destroyed and will then create a replacement.

Cloud servers will also be destroyed and recreated during normal BOSH operations, in addition to the abnormal situations above.

Routinely you will want to upgrade all the base operating systems for all the instances of all your deployments to push out security patches. The base operating system is called a BOSH "stemcell." These are maintained by the BOSH Core team and are regularly released with new security fixes (thanks also go to Canonical who maintain the Ubuntu distribution). You might find one or two new stemcells are released each month containing security fixes in the base operating system alone.

Fortunately, upgrading all your deployments to new BOSH stemcells is a very easy operation and we will definitely be returning to this topic soon. For now, you need to know that this process will result in your application processes being stopped (via Monit), the original servers being deleted (via the CPI), and new servers being created to replace them (via the CPI). Any files written arbitrarily to the filesystem will be lost.

Another routine operation you will perform that causes instances to be stopped, and the servers deleted and replaced, is resizing or scaling up your instances. BOSH CPIs assume that cloud providers do not know how to resize a running server, so they will emulate "resizing" by deleting the original small server and replacing it with a new larger server. Any files written arbitrarily to the filesystem will be lost.

## Persistent Volumes

Fortunately, there is a solution to storing data that survives longer than any ephemeral cloud server: persistent volumes.

Each cloud infrastructure provider has its own implementation of long-lived storage volumes. AWS has [Elastic Block Storage](https://aws.amazon.com/ebs/) (EBS). GCP has [Persistent Disks](https://cloud.google.com/compute/docs/disks/#pdspecs).

BOSH CPIs will map each cloud implementation to a homogenous experience for BOSH instances. Any BOSH instance that has persistent storage enabled will have a folder `/var/vcap/store`. Any files or data written within this folder will survive the turbulent life of ephemeral cloud servers.

Each BOSH instance has its own independent persistent volume. For example, our `zookeeper` deployment with five instances will have five persistent volumes. Each volume is associated to its BOSH instance for the life of the deployment. If an instance's underlying cloud server is destroyed and recreated, the same persistent volume will be reattached to the replacement cloud server. We will discuss this process more in later sections.

There is no concept in BOSH for sharing a volume between instances. If this facility is required, your deployment would include NFS or a similar network filesystem service running atop your BOSH instances and their persistent volumes.

Within a BOSH instance, we can see that `/var/vcap/store` is implemented as a separate Linux volume. The `df -h` Linux command will display disk usage summary for all mounted volumes:

```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            3.7G  4.0K  3.7G   1% /dev
tmpfs           748M  272K  748M   1% /run
/dev/sda1       2.8G  1.2G  1.5G  46% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
none            5.0M     0  5.0M   0% /run/lock
none            3.7G     0  3.7G   0% /run/shm
none            100M     0  100M   0% /run/user
/dev/sda3       9.6G  366M  8.7G   4% /var/vcap/data
tmpfs           1.0M  4.0K 1020K   1% /var/vcap/data/sys/run
/dev/sdb1       9.8G   23M  9.2G   1% /var/vcap/store
```

At the bottom we can see a volume mounted at `/var/vcap/store`. It is almost 10GB in size, has 23MB already used, and 9.2GB remaining. These numbers don't mathematically add up at all. In summary, there is a persistent disk and not much of it has been used yet.

The existence and size of the persistent volume is configured in the deployment manifest. For example, the `zookeeper-release/manifests/zookeeper.yml` file from our ongoing ZooKeeper example. Soon, we will begin looking inside this YAML file.

For now, know that it can be very simple to declare a persistent disk. A 10GB persistent volume will be provisioned, attached, and mounted at `/var/vcap/store` with the simple manifest entry:

```
persistent_disk: 10240
```

You will also learn that you can select different volume types and provide other cloud configuration using Persistent Disk Pools. For example, it is possible to configure all the different [Amazon EBS Volume Types](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html) and specific IOPS requirements.

## Filesystem Layout

The `df -h` command output above provides hints on why BOSH instances have a non-standard filesystem layout (for example, log files do not go into `/var/log`, but rather are stored in `/var/vcap/sys/log`).

The root volume (`/`) of the instance above is very small:

```
/dev/sda1       2.8G  1.2G  1.5G  46% /
```

It only has 1GB of available capacity. We do not want to install software packages or place fast-growing log files on this disk volume. Computer system behavior is unpredictable when disks fill up. The bad sort of unpredictable.

In addition to the small root volume, each BOSH instance is allocated an additional larger volume `/var/vcap/data`:

```
/dev/sda3       9.6G  366M  8.7G   4% /var/vcap/data
```

The size of `/var/vcap/data` is configurable and large, where as the `/` root volume is fixed and small.

**We want to use `/var/vcap/data` for storing everything that is ephemeral, `/var/vcap/store` for everything that is permanent, and avoid the root volume.**

For example, we want to use `/var/vcap/data` to install software packages, job templates, and log files. Except I've previously indicated that the filesystem location for these three was `/var/vcap/packages`, `/var/vcap/jobs`, and `/var/vcap/sys/log`.

BOSH instances have convenience symlinks so that files are actually stored within the large ephemeral `/var/vcap/data` volume. Some examples:

```
$ ls -al /var/vcap/jobs/zookeeper
lrwxrwxrwx ... /var/vcap/jobs/zookeeper -> /var/vcap/data/jobs/zookeeper/a14a07c550a0...

$ ls -al /var/vcap/packages/*
lrwxrwxrwx ... /var/vcap/packages/java -> /var/vcap/data/packages/java/050ac5643452f2...
lrwxrwxrwx ... /var/vcap/packages/zookeeper -> /var/vcap/data/packages/zookeeper/be0f...

$ ls -al /var/vcap/sys
lrwxrwxrwx ... /var/vcap/sys -> /var/vcap/data/sys
```

## What is VCAP?

In the time before VMWare first publicly announced Cloud Foundry in April 2011, and before they announced BOSH in April 2012, these collections of projects were known as, "VMWare Cloud Application Platform," or VCAP for short. Which naturally became `vcap` anywhere the engineers aesthetically preferred all-lowercase labels. Thus, the common root folder for everything installed on a machine became `/var/vcap`.

There have been occasional half-hearted ideas about renaming `/var/vcap` to something else. Anything else. But the convention that every file in a BOSH instance is nested underneath `/var/vcap` is incredibly pervasive today.

Naming is hard. Renaming is harder.

If you ever want to make a permanent mark in the short-lived, short-attention span world of software and operations, just put your name at the root of a folder system.

## Packages

This is the final stop on our tour of BOSH instances. We arrive at the depot of raw materials of what allows a `zookeeper` BOSH deployment with five BOSH instances to actually behave like a cluster of Apache ZooKeeper: the Apache ZooKeeper software itself.

If you have been exposed to a variety of different operating systems, (Windows, OS X, different Linux distributions) then you will have also been exposed to different packaging systems for the distribution and installation of software. This broad exposure will prepare you for BOSH packaging in one special way: you will be lacking the energy and the will to fight against, "OMG, why is there another packaging system?!"

BOSH packaging is similar to [Homebrew](https://brew.sh/) packaging for MacOS/OS X in that an installed package is a singular folder. An installed Homebrew package will be located at `/usr/local/Cellar/pkgname`. An installed BOSH package will be locatable at `/var/vcap/packages/pkgname` (and actually stored within the `/var/vcap/data` ephemeral volume as discussed in a previous section).

This model of "a package is a folder" is not compliant with the [Filesystem Hierarchy Standard](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard). But then again, you didn't know that the FHS existed until I just mentioned it.

There is a comfortable convenience to standard Linux filesystems. Your `PATH` environment variable is already setup for the common locations of executables. For example, on a BOSH instance, it is:

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

That's right, there are two folders included in `PATH` for games. I had to look it up - these are not mentioned in the [FHS wikipedia page](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard).

But I personally have a dissatisfaction with the standard Linux filesystem hierarchy: the knowledge of which files belong to each package is obscure. At the time that you need to know, "How does this process get started and stopped," or, "Where are the log files for this process." It's normally urgent and you're in a bad mood.

This is a big positive for the BOSH filesystem hierarchy: you can find everything in a hurry.

You look inside `/var/vcap/packages/pkgname/` and find every executable, shared library, and includable header file associated with that package.

You look inside `/var/vcap/jobs/jobname` and find the Monit control script, wrapper start/stop script, and every other associated configuration file to run that job template.

You look inside `/var/vcap/sys/log` and find all the logs for the processes that are running now or have run in the past.

One difference between BOSH packaging and every other packaging system (including Homebrew), is that packages are neither first class citizens nor is there any global repositories of shared packages.

Packages do not exist in isolation. Neither do job templates. Instead, they are rolled up into a singular first class entity called the BOSH release.

## Releases, Part 1

A BOSH deployment is a collection of instances (long-running servers and short-lived errands) upon which have been installed job templates and their dependent packages.

In the `zookeeper` deployment, the `zookeeper` job template needs the `zookeeper` and `java` packages installed.

Which `zookeeper` package? You might assume that it is Apache ZooKeeper, but which version? Is it the unmodified source code, or a pre-built package? Do we need some additional security or features patched into it?

Which `java` package? Oracle Java or OpenJDK? The JDK or the JRE? Which version?

There is an important assumption built into the very fiber of BOSH that each of these questions is important and the answer potentially critical. Even if the initial person who deployed ZooKeeper initially didn't specifically care which version of Apache ZooKeeper or which JDK they used; everyone from that day onward will need to care.

Each version of a BOSH release combines a specific combination of packages and job templates that are written and packaged to work together. Instead of worrying about a matrix of ZooKeeper and Java editions and versions that might work together, a BOSH release will explicitly package one of each together as a known good pair.

If a security patch is released for Apache ZooKeeper, then a new version of the entire BOSH release needs to be created, tested, and deployed.

If you need to switch from Oracle Java to OpenJDK, then a new version of the entire BOSH release needs to be created, tested, and deployed.

For this reason I say that packages and job templates are not first class citizens in BOSH. BOSH releases are the primary unit of deployment into BOSH instances.

The method for selecting specific versions of specific BOSH releases to be combined into a BOSH deployment is using the deployment manifest.
