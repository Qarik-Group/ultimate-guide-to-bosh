# Service Discovery

Service discovery is a mechanism for applications to discover and connect to other sub-systems. For example, a ZooKeeper node needs to be able to discover and connect to the other members of its cluster; an application to discover and connect to its database; and a mobile or internet-of-things application to discover and connect to its backend services.

Systems are deployed by BOSH may either want to discover and connect to other systems; or conversely they may want to advertise and be available for connection by clients.

The primary roles of a BOSH environment is to manage the lifecycle of the cloud infrastructure, and the lifecycle of the software running upon it. However, BOSH job templates do have some facilities for service discovery with other job templates, and there are some popular options outside of BOSH for service discovery as well.

BOSH offers three facilities to aide service discovery:

* Static or elastic IP allocation
* BOSH DNS
* BOSH Links

There are also many other additional systems that you could use, or even run them with BOSH, to facilitate service discovery between applications. For example:

* Apache ZooKeeper, Hashicorp Consul, CoreOS etcd
* Cloud Foundry Routing
* Chef Habitat
* Open Service Broker API
* Apcera NATS, RabbitMQ

## Static or elastic IP allocation

The least complex method for a client application to discover and connect with a server backend is to explicitly configure the client application in advance. The least complex method of this least complex method is to provide client applications with IP addresses that are promised to always point to the backend service.

A BOSH deployment can do this with either [static IPs](/networking/#manual-static-addresses) or with [virtual/elastic IPs](/networking/#virtual-ip-addresses).

An example of the former is to add a list of `static_ips` to an instance group's `networks`:

```yaml hl_lines="6 7 8 9 10 11"
instance_groups:
- name: zookeeper
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

In this scenario, you would commit in advance to using the five IP addresses. You would write them down on two identical Post-It Notes. You'd give one Post-It Note to the team deploying ZooKeeper as above, and you'd give the other Post-It Note to the client team that wants to connect to the future ZooKeeper cluster. Instead of Post-It Notes you might send an email. But my guess is you've already ordered Post-It Notes from Amazon.com earlier in the paragraph when I first mentioned them.

## BOSH DNS

Static IP addresses might be the least complex method - you buy some Post-It Notes, you pick some IP addresses - but what if you're a normal human being and have no ambitions to keep track of what systems have been allocated what IP addresses. Or you don't have Post-It Notes.

Your BOSH environment can have the ability to automatically advertise all deployments' instances as DNS hostnames within all other BOSH instances. This means that any deployment can know in advance what its hostnames will be, without knowing in advance the IP addresses allocated or using static IPs.

First, confirm that your BOSH environment has this BOSH DNS feature enabled. Run `bosh env` to view your environment's attributes:

``` hl_lines="10"
> bosh env
Using environment '192.168.50.6' as user 'admin' (openid, bosh.admin)

Name      Bosh Lite Director
UUID      d855fe91-a9cb-43be-b977-f44eea870775
Version   263.2.0 (00000000)
CPI       warden_cpi
Features  compiled_package_cache: disabled
          config_server: disabled
          dns: enabled
          snapshots: disabled
User      admin
```

TODO: why is `dns: disabled` appearing on my `-o local_dns.yml` bosh-lite?

TODO: what about GCP?

TODO: finish section

## BOSH Links

The final service discovery option provided by your BOSH environment is called "links". At the time a job template is being installed into an instance, it is provided with information about other job templates within the same deployment, and possibly other deployments.

BOSH links is the method by which the five instances in our `zookeeper` deployment can find each other and form a cluster.
