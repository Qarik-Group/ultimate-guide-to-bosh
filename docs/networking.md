# Networking

If your professional relationship with computers to date has been running processes, maybe using containers, but you've never had to setup servers and their networking before, then allow me to be the first to congratulate you on a career path well trodden, and to apologise that your joyride is over.

Networking is more complicated and anti-human than it needs to be. Consider a simple example. You will have seen an IP address before. Four small numbers with three dots. For example, `10.11.12.13`. This address is actually called IPv4. The total number of public Internet IPv4 addresses is relatively small compared to our growing use for them, so networking people invented IPv6. Your innocent soul would be wrong to think that IPv6 addresses look like: `10.11.12.13.14.15`.

Instead, the sociopaths involved in computer networking made them look like:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

I will try hard to never discuss [IPv6](https://en.wikipedia.org/wiki/IPv6_address) again.

As a consumer of computers, networking and addressing of computers talking to each other is hidden. `google.com` in a browser just works. From there you go to other URLs and their pages appear in your browser. It all just works. But you're no longer a consumer of computers. You are a purveyor of fine software systems.

## Why Learn Networking?

One of the reasons that I want to share information on networking, is that a common cause of most, "Why doesn't this work for me" requests on the `#bosh` community [Slack channel](https://cloudfoundry.slack.com) is networking.

Networking is difficult in part due to two opposing requirements:

**Distributed Systems** - where different processes want to talk to each other on different servers - requires that the processes can discover each other and can connect to each other.

**Security** - ensuring a system is only doing what it is supposed to do and is not corrupted or misused by bad actors - requires that we restrict as much access to servers and processes as possible. But not too much so as to stop a distributed system from working.

When processes within a distributed system cannot discover or communicate with its peers, it probably will not work. But the way in each distributed system "probably doesn't work" will be different.

### Discovering That a Distributed System is Failing is Nontrivial

A process might refuse to start successfully because it cannot connect to a dependent subsystem. Monit will then restart that process over and over infinitely. Another process might start running even if it cannot access its dependencies, but when users interact with that process it might then return errors. Or it might provide a subset of normal behaviour, rather than explicitly error. Some erroneous behaviour might be intermittent.

### Debugging a Distributed System is Nontrivial

It is hard enough for software developers to write software that works at all and provides the features that end users want. The ways in which networking errors can propogate into each application process are more numerous than the "happy path" for the application when there are no networking errors. It will be likely that many distributed systems you work with do not provide a lot of assistance in debugging what networking errors have appeared.

More commonly, inside the error logs of one or more processes will be a variation of a "connection timeout" error or more subtle/invisible indications that a networking issue has appeared.

So hopefully, by learning networking and "owning responsibility" for the networking of your distributed systems, you will limit the scope for accidental networking issues and improve your ability to debug and resolve networking issues.

## BOSH Does Not Configure Networks

Whilst, BOSH can provision cloud servers and persistent disks, it cannot provision/change networking. Either your networking will have been setup at the time your cloud infrastructure was setup, such as a vSphere environment; or you will use additional tools to provision and manage networking in cloud infrastructures such as AWS, GCP, or Microsoft Azure.

When we discuss using BOSH in different infrastructures, we will look at some specific aspects of setting up networking before using BOSH.

## Example Networking in Cloud-Config

To help frame this mini-guide to networking, I'll first introduce where networking configuration appears within `bosh cloud-config` and deployment manifests.

### Networking Configuration in Cloud-Config

Remember that `bosh cloud-config` is where the bulk of cloud infrastructure specific configuration resides. Earlier we reviewed `vm_types` and mapped each one to a CPI specific instance type/machine type/VM configuration depending on the CPI.

Networking configuration is also specific to the target infrastructure, but fortunately there is a common set of attributes across all CPIs.

Look for the subtle differences in each example.

Here is an example of describing networking for a vSphere environment:

```yaml
networks:
- name: default
  type: manual
  subnets:
  - range: 10.0.0.0/24
    gateway: 10.0.0.1
    static: [10.0.0.5-10.0.0.20]
    azs: [z1,z2,z3]
    dns: [8.8.8.8]
    cloud_properties:
      name: net-10-0-0-0

azs:
- name: z1
  cloud_properties:
    datacenters:
    - clusters: [mycluster: {}]
- name: z2
  cloud_properties:
    datacenters:
    - clusters: [mycluster: {}]
- name: z3
  cloud_properties:
    datacenters:
    - clusters: [mycluster: {}]
```

Here is an example of describing networking for a GCP environment:

```yaml
networks:
- name: default
  type: manual
  subnets:
  - range: 10.0.0.0/24
    gateway: 10.0.0.1
    static: [10.0.0.220-10.0.0.254]
    dns: [8.8.8.8]
    cloud_properties:
      network_name: mynetwork
      subnetwork_name: mysubnetwork
      ephemeral_external_ip: true
      tags: [tag1, tag2]

azs:
- name: z1
  cloud_properties:
    zone: europe-west1-b
- name: z2
  cloud_properties:
    zone: europe-west1-c
- name: z3
  cloud_properties:
    zone: europe-west1-d
```

Here is an example of describing networking for an AWS VPC environment:

```yaml
networks:
- name: default
  type: manual
  subnets:
  - range: 10.0.0.0/24
    gateway: 10.0.0.1
    static: [10.0.0.220-10.0.0.254]
    reserved: [10.0.0.0/30]
    dns: [8.8.8.8]
    cloud_properties:
      subnet: subnet-123456
      security_groups: [bosh]
- name: elastic
  type: vip

azs:
- name: z1
  cloud_properties:
    availability_zone: eu-west-2a
- name: z2
  cloud_properties:
    availability_zone: eu-west-2b
- name: z3
  cloud_properties:
    availability_zone: eu-west-2c
```

There is a lot going on at first glance. Look at the three examples a few times and you'll start to see some patterns.

The common structure looks like:

```yaml
networks:
- name: some-name-used-by-deployment-manifests
  type: manual
  subnets:
  - range: 10.0.0.0/24
    gateway: 10.0.0.1
    static: [10.0.0.220-10.0.0.254]
    dns: [8.8.8.8]
    cloud_properties: {...}

azs:
- name: z1
  cloud_properties: {...}
- name: z2
  cloud_properties: {...}
- name: z3
  cloud_properties: {...}
```

I promise that everything you've seen in these example `cloud-config` will make sense. This is all learnable and understandable.

### Networking Configuration in a Deployment Manifest

Consider this abbreviated `zookeeper.yml` deployment manifest:

```yaml
instance_groups:
- name: zookeeper
  azs: [z1, z2, z3]
  instances: 5
  vm_type: default
  networks:
  - name: default
```

When the abbreviated `zookeeper-release/manifests/zookeeper.yml` was first introduced in [Deployment manifests, part 1](/deployment-manifests-part-1) above, the `azs` and `networks` attributes were omitted.

Each `instance_groups` item must include an `azs` and `networks` attribute. At a glance you can see that the `azs` values correspond to the `azs` from the sample `cloud-config` above, and the `networks` name `default` corresponds to one of the `cloud-config` `networks` items.

This deployment manifest is inferring that each of the 5 instances will be allocated a different IP address from the `default` network. It isn't important what exact IP address will be assigned, as the job templates running on each instance will be provided the IP addresses for all the instances in the instance group.

It is useful to understand that from the sample `cloud-config` we can see that these IP addresses might be in the range of `10.0.0.2` to `10.10.0.219`. Let's investigate IP ranges and how BOSH allocates IP address(es).

### Mapping to External Network

As stated earlier, BOSH does not provision or manipulate the cloud infrastructure networking. Instead, the `networking` section in `cloud-config` describes the available networking space that BOSH can use for its cloud servers.

The examples above are each stating that there already exists a network `10.0.0.0/24` that the BOSH director and CPI has access to on their respective infrastructure. Your `cloud-config` will need to specifically describe your own network availability.

## BOSH IP Allocation vs DHCP

If you've played with networking before - such as trying to get your computer and devices onto your home router - you'll have blissfully ignored how an IP address is allocated to your computer or device. This facility is thanks to Dynamic Host Configuration Protocol (DHCP).

You might have seen that mysterious IP `169.254.X.Y` that indicates that DHCP has failed and your device has [allocated itself](http://packetlife.net/blog/2008/sep/24/169-254-0-0-addresses-explained/) an IP address.

The allocation of IP addresses to BOSH instances is not performed with DHCP. Instead, the BOSH director/CPI statically assign IP addresses to each instance. From our perspective, the net result is the same: IP address allocation is not manually managed by you (by default).

Instead, you will give guidance to the BOSH director and CPI as to where each instance will be placed within your networking range, and an IP will be chosen.

Consider an example network in a `cloud-config`:

```yaml
networks:
- name: default
  type: manual
  subnets:
  - range: 10.0.0.0/24
    gateway: 10.0.0.1
    static: [10.0.0.220-10.0.0.254]
    reserved: [10.0.0.0/30]
    dns: [8.8.8.8]
    cloud_properties:
      subnet: subnet-123456
      security_groups: [bosh]
```

The name `default` will be used in each deployment manifest to describe "where" its instances will be placed. In this AWS example, the AWS CPI will use `subnet: subnet-123456` to place the instance within that specific subnet ID.

The IP address will be selected from the available IPs within the `range` range, but not including the `static` ranges, nor `reserved` ranges.

BOSH allows three formats for describing a range of IPs:

* single IP address - `10.0.0.220`
* range of IP addresses - `10.0.0.220-10.0.0.254`
* CIDR notation - `10.0.0.0/30`

## CIDR Notation

BOSH will select an available IP address from the range `10.0.0.0/24`. This notation for a range of IP addresses is called, [Classless Inter-Domain Routing](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) (CIDR). It is short hand for "all IPs between 10.0.0.0 and 10.0.0.255".

[CIDR calculators](http://www.ipaddressguide.com/cidr) can be your friend to learn and experiment with CIDR notations.

Some examples:

* `10.0.0.0/30` - 4 IPs, from `10.0.0.0` to `10.0.0.3`
* `10.0.0.0/29` - 8 IPs, from `10.0.0.0` to `10.0.0.7`
* `10.0.0.0/28` - 16 IPs, from `10.0.0.0` to `10.0.0.15`
* `10.0.0.0/24` - 255 IPs, from `10.0.0.0` to `10.0.0.255`
* `10.0.0.0/16` - 65536 IPs, from `10.0.0.0` to `10.0.255.255`

A useful nuance of CIDR notation is that we round down the base IP address, so the following examples are equivalent:

* `10.0.0.0/30` - 4 IPs, from `10.0.0.0` to `10.0.0.3`
* `10.0.0.1/30` - 4 IPs, from `10.0.0.0` to `10.0.0.3`
* `10.0.0.2/30` - 4 IPs, from `10.0.0.0` to `10.0.0.3`
* `10.0.0.3/30` - 4 IPs, from `10.0.0.0` to `10.0.0.3`

In some BOSH files, you might see this nuance used within operator files, such as this sample [AWS cloud-config template](https://github.com/cloudfoundry/bosh-deployment/blob/096b55060b77d438af385c010841cf8608c63dfb/aws/cloud-config.yml#L31-L36):

```yaml
networks:
- name: default
  type: manual
  subnets:
  - range: ((internal_cidr))
    gateway: ((internal_gw))
    reserved: [((internal_gw))/30]
```

The `((internal_gw))` variable is used to describe the gateway to the subnet range. If `((internal_gw))` is `10.0.0.1`, then the `reserved: [((internal_gw))/30]` effectively evaluates to `reserved: [10.0.0.0-10.0.0.3]`. We will properly introduce BOSH [Operator files](/deployment-updates/#operator-files) and [Variables](/deployment-updates/#deployment-manifest-variables) soon.

## Gateway

The gateway is the IP address that your BOSH instances will use to communicate with the Internet or beyond its network it will send it to the gateway IP address. Data arriving from outside the network will arrive via this gateway IP address.

Typically the gateway IP is the base IP address of the `range` plus 1. A network range of `10.11.0.0/16` will typically have a gateway of `10.11.0.1`.

## Reserved Ranges

When BOSH is dynamically allocating IPs for instances it needs to know any IP addresses that it cannot use. In the `cloud-config`, this is an attribute called `reserved`. It is an optional array of IP ranges.

Some cloud infrastructures reserve IP addresses for every network. AWS reserves the first four IP addresses for itself. Therefore, an AWS `cloud-config` will always have at least one `reserved` range.

Sometimes it might be that there is one common infrastructure network that must be split into multiple virtual BOSH networks, perhaps within a single `cloud-config` or perhaps across multiple BOSH directors (you might have a BOSH director for staging deployments and another for production deployments but yet only have one common network for all deployments to share). You will use `reserved` ranges to manually split the underlying network into virtual networks.

## Explicit Static IP Address

For the most part, you should not feel you need to declare IP addresses in advance. Job templates have a mechanism for discovering the IP addresses for each other, called Links, which will be introduced soon.

There are occasions where you may want to share the IP addresses of instances in an instance group with external clients. One method will be to statically declare specific IP addresses in the deployment manifest that BOSH will promise to assign to the instances.

There are two systems of pre-defined IP addresses that you can consider.

### Manual Static Addresses

Rather than delegate the selection of IP addresses to BOSH, your deployment manifest can explicitly request specific IP addresses for each instance in an instance group. We add the attribute `static_ips` to our instance group's `networks` section.

The abridged manifest for an instance group with static IPs might be:

```yaml
instance_groups:
- name: zookeeper-instances
  instances: 5
  networks:
  - name: default
    static_ips:
    - 10.0.0.220
    - 10.0.0.221
    - 10.0.0.222
    - 10.0.0.223
    - 10.0.0.224
```

Since `instances` is `5`, therefore the `static_ips` list must include five IP addresses.

The `static_ips` IP addresses must also be declared within the `cloud-config`. In the examples included earlier and below, they each had a `static: [10.0.0.220-10.0.0.254]` attribute. These IP addresses - between `10.0.0.220` and `10.0.0.254` - will not be dynamically assigned to other instances; they can only be used by deployment manifests declaring `static_ips`.

When we introduce Links, you will see that there are relatively few reasons for requiring static IPs; and your life is a lot simpler without having to manually select IP addresses for the benefit of external systems.

### Virtual IP Addresses

Most cloud infrastructures have a concept of virtual IP addresses. In OpenStack, they are called floating IPs, in AWS they are called elastic IPs, and in GCP they are called static external IPs. Virtual IPs are public IP addresses that you lease or borrow from your cloud provider, for example elastic IPs are leased from AWS. Amongst other things, virtual IPs are useful for providing some predictability in the network reachability of cloud instances - because for one thing you don't lose them as instances - stop, restart or outright fail. In other words virtual addresses are not ephemeral in nature and can be moved between virtual machines. Since they belong to an organization after allocation (until released) and are not ephemeral, things like firewall rules, network ACLs, and DNS records can be created for them and will not need to be changed even if the VM they point to fails.

Like networks in general, BOSH cannot provision or destroy virtual IP addresses. Instead, it can help you manage the assignment of them to instances.

For example, you might provision 5 elastic IPs on AWS and be told that their values are `54.1.2.3`, `56.2.3.4`, `58.4.5.6`, `123.1.2.3`, and `124.2.3.4`. We can then ask BOSH to assign them to each of our 5 `zookeeper` instances:

```yaml
instance_groups:
- name: zookeeper-instances
  instances: 5
  networks:
  - name: default
    default: [dns, gateway]
  - name: elastic
    static_ips:
    - 54.1.2.3
    - 56.2.3.4
    - 58.4.5.6
    - 123.1.2.3
    - 124.2.3.4
```

This deployment manifest example for AWS uses two `networks`, which map to the two `networks` from the AWS `cloud-config` example earlier by their names `default` and `elastic`:

```yaml
networks:
- name: default
  type: manual
  subnets:
  - range: 10.0.0.0/24
    ...
- name: elastic
  type: vip
```

BOSH supports three different network types, discussed in the next section.

## Network Types

You've now seen enough examples of deployment manifests and `cloud-config` to be introduced to BOSH networks from the [official documentation](https://bosh.io/docs/networks.html).

> A BOSH network is an IaaS-agnostic representation of the networking layer. The Director is responsible for configuring each deployment job’s networks with the help of the BOSH Agent and the IaaS. Networking configuration is usually assigned at the boot of the VM and/or when network configuration changes in the deployment manifest for already-running deployment jobs.
>
> There are three types of networks that BOSH supports:
>
> * **manual**: The Director decides how to assign IPs to each job instance based on the specified network subnets in the deployment manifest
> * **vip**: The Director allows one-off IP assignments to specific jobs to enable flexible IP routing (e.g. elastic IP)
> * **dynamic**: The Director defers IP selection to the IaaS

We've seen `type: manual` and `type: vip` in preceding sections.

The third `type: dynamic` is used with AWS original networking, and legacy OpenStack Nova networking. I personally liked these simpler "flat" networking systems. You requested a BOSH instance, and the underlying networking allocated you the IP. You didn't need to create networks and other networking infrastructure, nor did your BOSH manifests/`cloud-config` need to specify CIDR network ranges, reserved + static ranges. Simpler times.

In production environments, you will wish for the security features and control from more complex networking. Or someone in your organisation will do this wishing for you.

If you are using AWS today, you will instead use VPC networking, and with OpenStack you will use Neutron networking. These are both represented in BOSH `cloud-config` as `type: manual` networking.

### Manual Networks with AWS VPC

Consider an AWS VPC with a single subnet:

![aws-subnet](/images/aws/aws-subnet.png)

The basic `cloud-config` configuration would be:

```yaml
azs:
- name: az1
  cloud_properties: {zone: us-west-2a}
- name: az2
  cloud_properties: {zone: us-west-2a}
- name: az3
  cloud_properties: {zone: us-west-2a}

networks:
- name: default
  type: manual
  subnets:
  - range: 10.0.0.0/24
    gateway: 10.0.0.1
    reserved: [10.0.0.1/30]
    static: [10.0.0.10-10.0.0.19]
    azs: [az1, az2, az3]
    cloud_properties:
      subnet: subnet-4cff3507
- name: external
  type: vip
```

Note: In AWS, 1 subnet = 1 AWS availability zone. Above, all three of the bosh AZs that we defined are within the same subnet, and therefore also within the same AWS availability zone. For the most resiliant deployment, all three cloud-config defined AZs should be in different AWS availability zones.


### Manual Networks with GCP VPC

Consider a GCP VPC with a single subnet:

![gcp-vpc-networks-bosh](/images/gcp/gcp-vpc-networks-bosh.png)

The basic `cloud-config` configuration would be:

```yaml
azs:
- name: z1
  cloud_properties: {zone: us-east1-d}
- name: z2
  cloud_properties: {zone: us-east1-d}
- name: z3
  cloud_properties: {zone: us-east1-d}

networks:
- name: default
  type: manual
  subnets:
  - range: 10.0.0.0/24
    gateway: 10.0.0.1
    reserved: [10.0.0.1/30]
    static: [10.0.0.10-10.0.0.19]
    dns: [8.8.8.8]
    azs: [z1, z2, z3]
    cloud_properties:
      ephemeral_external_ip: true
      network_name: bosh
      subnetwork_name: bosh-us-east1
      tags: [internal, no-ip, windows-rdp]
- name: external
  type: vip
```

The `dns: [8.8.8.8]` configures all VMs to use [Google's public DNS servers](https://developers.google.com/speed/public-dns/docs/intro). These public DNS servers are popular outside of GCP as well as for your BOSH deployments within GCP.

The `tags` are a selection of the available firewall rules and network routes. If the tag matches a firewall rule, then that firewall rule is applied to each instance in that network. For example, the `windows-rdp` tag above matches the firewall rule with the same name below that was preconfigured:

![gcp-vpc-networks-firewall-rules](/images/gcp/gcp-vpc-networks-firewall-rules.png)

Other tags are used to identify which networking routes to apply to each instance in the network. The `no-ip` tag in the example above corresponds to the following predefined route in my Google Compute environment. The "next hop" of the route is to the "nat-instance-primary" server which will provide my instances with outbound, public Internet access:

![gcp-vpc-networks-route-details](/images/gcp/gcp-vpc-networks-route-details.png)

In GCP, the `type: vip` IP addresses are called, "External IP addresses". For this reason, I've named the network `external`. It is also common for the `type: vip` network to be generically named `vip`.

### Manual Networks with OpenStack Neutron

TODO

### Manual Networks with vSphere

TODO


## Further Reading on BOSH Networks

The BOSH documentation has some additional information about [networking and configuration options](https://bosh.io/docs/networks.html) that is well worth reading now and again later for reference.
