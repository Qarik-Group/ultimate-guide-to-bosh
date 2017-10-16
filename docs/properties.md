# Properties

Most software allows for configuration by end users/operators. The method of configuration differs between software, such as environment variables, configuration files, command-line arguments, or runtime commands. BOSH job templates are responsible for configuring and describing how to run software. As such, BOSH job templates can offer configuration properties to allow operators to provide their required missing information or to change default configuration.

You were first introduced to overriding a job template's properties in [Update Job Template Properties](/deployment-updates/#update-job-template-properties).

We were able to provide a `max_client_connections` property to the `zookeeper` job template from the `zookeeper` BOSH release:

```yaml hl_lines="7 8"
instance_groups:
- name: zookeeper
  instances: 5
  jobs:
  - name: zookeeper
    release: zookeeper
    properties:
      max_client_connections: 120
```

This raise two excellent questions:

* What other properties are available to the `zookeeper` job template from the `zookeeper` BOSH release?
* What does each property do?

## Available Job Template Properties

The definition of available properties for a job template is in its source repository. Each job template has a `spec` file that documents the `properties` that its templates can use.

For https://github.com/cppforlife/zookeeper-release we look in `jobs/zookeeper/spec`:

```
git clone https://github.com/cppforlife/zookeeper-release
cd zookeeper-release
cat jobs/zookeeper/spec
```

A sample of `properties` are:

```yaml hl_lines="15 16 17"
properties:
  listen_address:
    description: "The address to listen for client connections"
    default: "0.0.0.0"
  client_port:
    description: "The port to listen for client connections"
    default: 2181
  quorum_port:
    description: "Apache Zookeeper Client quorum port"
    default: 2888
  leader_election_port:
    description: "Apache Zookeeper Client leader election port"
    default: 3888
  ...
  max_client_connections:
    description: "Limits the number of concurrent connections that a single client may make to a single member of the ZooKeeper ensemble"
    default: 60
  ...
```

## Default Property Values

The deployment manifest property `max_client_connections` is one of `properties` described in the BOSH release source repository above. It has a default value. This means that a deployment manifest does not need to explicitly provide the `max_client_connections` property.

Default values allow deployment manifests to be smaller, simpler, and easier to read. The original `spec` file above has over twenty properties each with useful default values. A deployment manifest would be twenty lines longer if it were to explicitly provide each property in the deployment manifest:

```yaml hl_lines="8 9 10 11 12"
instance_groups:
- name: zookeeper
  instances: 5
  jobs:
  - name: zookeeper
    release: zookeeper
    properties:
      listen_address: 0.0.0.0
      client_port: 2181
      quorum_port: 2888
      leader_election_port: 3888
      max_client_connections: 60
      ...
```

A potential downside of default property values is that they may change between BOSH release versions. If the previous release version had default `max_client_connections: 120`, but was changed to `max_client_connections: 5` in the latest version, then our system's behaviour and performance might be negatively affected if we do not realise and explicit override the property to a higher value.

Changes to default property values are not displayed when running `bosh deploy`.

Ideally, the BOSH release developers will discuss any changes to property default values in their release notes. And that you will read those release notes.

You can also use `git diff` to inspect any changes to `spec` files between versions. Try either of the following to view changes to one or all `spec` files, respectively:

```
git diff v0.0.3..v0.0.6 -- jobs/zookeeper/spec
git diff v0.0.3..v0.0.6 -- jobs/*/spec
```

We can see that two properties from `zookeeper` have been removed, one new property added, ==and fortunately no properties have had their default values changed==:

```
-  user:
-    description: "User which will own the Apache ZooKeeper services"
-    default: "zookeeper"
-  group:
-    description: "Group which will own the Apache ZooKeeper services"
-    default: "vcap"
...
+  force_sync:
+    description: "Requires updates to be synced to media of the transaction log before finishing processing the update. Setting to 'no' improves performance dramatically at the cost of losing recent commits if all nodes crash at the same time"
+    default: "yes"
```

## Implementation of Properties

TODO
