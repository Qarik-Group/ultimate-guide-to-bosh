# Service Discovery

Service discovery is a mechanism for applications to discover and connect to other sub-systems. For example, a ZooKeeper node needs to be able to discover and connect to the other members of its cluster; an application to discover and connect to its database; and a mobile or internet-of-things application to discover and connect to its backend services.

Systems the are deployed by BOSH may either want to discover and connect to other systems; or conversely they may want to advertise and be available for connection by clients.

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


## BOSH Links
