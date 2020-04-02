# Deployment Updates

One of the fabulous features of BOSH is the ability to change, scale, and evolve your deployments throughout their multi-year lifespans.

## Scaling deployments

Over time, your long-lived BOSH deployments may need to be scaled up to cope with increased traffic or accrued data. Or you might discover that the initial deployment uses more resources than necessary and you want to downsize the deployment.

You can change the scale of your deployments using the same methodology for making any changes to a deployment:

1. Modify the deployment manifest
2. Deploy the manifest

    ```
    bosh deploy path/to/manifest.yml
    ```

For example, we can change our `zookeeper` cluster down to 3 instance, whilst simultaneously changing the instance size's `vm_type` attribute and increasing the `persistent_disk` size. The abridged `zookeeper-release/manifests/zookeeper.yml` deployment manifest showing the changed attributes:

```yaml
---
name: zookeeper

instance_groups:
- name: zookeeper
  instances: 3
  vm_type: large
  jobs:
  - name: zookeeper
    release: zookeeper
  - name: status
    release: zookeeper
  persistent_disk: 20480

- name: smoke-tests
  ...
```

After making these changes and saving the file, you would then run:

```
> bosh deploy zookeeper-release/manifests/zookeeper.yml
```

The `bosh` CLI will display the various proposed changes. Once you confirm by pressing `y`, the new deployment manifest will be uploaded to your BOSH director, which will perform the changes.

## Modifying the Running Deployment Manifest

If you do not have the original deployment manifest available when you want to modify it, you have the option to download the last deployment manifest that was successfully deployed.

```
bosh manifest > manifest.yml
```

After modifying this file, you could then apply the changes with `bosh deploy`:

```
bosh deploy manifest.yml
```

## Operator Files

It can be common that the deployment manifest comes from a source repository that you do not maintain. Our common example `zookeeper-release/manifests/zookeeper.yml` is a file within the https://github.com/cppforlife/zookeeper-release repository. If you directly modify this file you will not be able to commit the changes to this repository.

Instead, the `bosh deploy` command allows you to apply "operators" to the base manifest; that is, changes or amendments. We use the `bosh deploy -o path/changes.yml` flag for each operator file.

An example operator file that will change the number of `zookeeper` instances from 5 to 3 is:

```yaml
- type: replace
  path: /instance_groups/name=zookeeper/instances
  value: 3
```

Another example - either in a new file or appended to the file above - to change the `persistent_disk` attribute for each `zookeeper` instance:

```yaml
- type: replace
  path: /instance_groups/name=zookeeper/persistent_disk
  value: 20480
```

Finally, to change the `vm_type`:

```yaml
- type: replace
  path: /instance_groups/name=zookeeper/vm_type
  value: large
```

If these three YAML snippets were in a file `zookeeper-scale.yml`, then to apply the operators to our deployment manifest:

```
> bosh deploy zookeeper-release/manifests/zookeeper.yml -o zookeeper-scale.yml
```

### Operator paths

There are two `type` options in each Operator: `replace`, and `remove`. The former will update or insert an item in the final YAML file. The latter will remove an item from the final YAML file. The item to be modified is determined by the `path` expression.

The `path` expression describes a walk up a YAML tree, so to speak. Consider the example YAML file ("tree"):

```yaml
name: zookeeper

instance_groups:
- name: zk
  instances: 5
  networks:
  - name: default
```

The following `path` expressions and their corresponding items from the YAML file:

* `path: /name`
    ```
    zookeeper
    ```
* `path: /instance_groups`
    ```
    - name: zk
      instances: 5
      networks:
      - name: default
    ```
* `path: /instance_groups/0` - index 0 in the `/instance_groups` array
    ```
    name: zk
    instances: 5
    networks:
    - name: default
    ```
* `path: /instance_groups/name=zk` - the item of the `/instance_groups` array with `name: zk`
    ```
    name: zk
    instances: 5
    networks:
    - name: default
    ```
* `path: /instance_groups/name=zk/instances` - the value of `instances` from the preceding item
    ```
    5
    ```
* `path: /instance_groups/name=zk/networks` - the value of `networks` is an array of one item:
    ```
    - name: default
    ```
* `path: /instance_groups/name=zk/networks/name=default` - specifically selects the `networks` item with `name: default`:
    ```
    name: default
    ```

### Why not use XPath or CSS selectors?

Walking a structured document such as a YAML/JSON/XML file to select an item has other syntaxes that you might have seen before. Your rounded knowledge of computers will do you well in life. But I'm sorry to say, `bosh deploy` doesn't use those syntaxes.

The Operator `path` syntax comes from the JSONPath spec. TODO link.

### More Examples of Operator Files

Support for Operator files is provided by a library https://github.com/cppforlife/go-patch. See TODO for an extensive list of examples.


## Assign Cloud Config Options to Deployment Manifest

Another use of Operator files is to assign the available options within your `bosh cloud-config` to the deployment manifest. A base deployment manifest will not know in advance the available `vm_types` or `networks` within your Cloud Config.

In the previous section we modified the `vm_type: large` where `large` was known to us in advance as an available option from our `vm_types`.

To see the list of available `vm_types`:

```
> bosh int <(bosh cloud-config) --path /vm_types
```

Similarly, to see the list of available `networks`:

```
> bosh int <(bosh cloud-config) --path /networks
```

You would then create an Operator file to assign your chosen option to each instance group. For our `zookeeper` deployment with its two instance groups `zookeeper` and `smoke-tests`:

```yaml
- type: replace
  path: /instance_groups/name=zookeeper/vm_type
  value: large

- type: replace
  path: /instance_groups/name=zookeeper/networks/name=default/name
  value: private

- type: replace
  path: /instance_groups/name=smoke-tests/vm_type
  value: tiny

- type: replace
  path: /instance_groups/name=smoke-tests/networks/name=default/name
  value: private
```

Note, for completeness of your Operator file education, the two `networks` operator expressions could also be implemented as either of the follow examples:

```yaml
- type: replace
  path: /instance_groups/name=zookeeper/networks
  value:
  - name: private

- type: replace
  path: /instance_groups/name=smoke-tests/networks/name=default
  value:
    name: private
```

The former example will replace the entire `/instance_groups/name=zookeeper/networks` with an explicit declaration of the `networks` for that instance group.

The latter example will isolate and replace the specific `networks` item with the `name: default`.

Since the original example YAML file only had one `networks` array item all three examples result in the same modification.

## Update Job Template Properties

Most job templates have optional or required properties. So far in our `zookeeper` deployment examples we have not explicitly overridden any properties, rather have enjoyed the mystery of the unknown default properties.

We can provide a property within an instance group with the optional `properties` attribute. For example, the `zookeeper` job template has a property `max_client_connections` with a default value `60`. To explicitly declare this property with a value of `120`, we can either modify the deployment manifest directly:

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

Or we can use an Operator file to add the missing `properties.max_client_connections` attribute:

``` hl_lines="2"
- type: replace
  path: /instance_groups/name=zookeeper/jobs/name=zookeeper/properties?/max_client_connections
  value: 120
```

Note the special use of `?` question mark in `/properties?/max_client_connections`. This means, "if `/properties` attribute is missing, then please create it first". Operator files are nothing if not courteous. They will not insert new attributes into a YAML file without permission from the `?` postfix operator.

## Deployment Manifest Variables

In addition to `-o` Operator files, the `bosh deploy` command also allows you to pass in variables with the `-v` flag. Variables are a simpler way to specify values than the relatively complex Operator file syntax. But you can only provide variables if the deployment manifest wants them.

A deployment manifest will request variables with the double parentheses syntax `((var-name))`.

Consider an abridged variation of our `zookeeper-release/manifests/zookeeper.yml` file where the `properties.max_client_connections` attribute is declared, but the value is a variable:

```yaml
name: zookeeper

instance_groups:
- name: zookeeper
  instances: 5
  jobs:
  - name: zookeeper
    release: zookeeper
    properties:
      max_client_connections: ((zk-max-client-connections))
```

Your `bosh deploy` command will now require that you provide the value for this variable:

```
> bosh deploy zookeeper-release/manifests/zookeeper.yml \
  -v zk-max-client-connections=120
```

If you forget to provide a required variable, the BOSH director may error with output similar to:

```
> bosh deploy zookeeper-release/manifests/zookeeper.yml
...
Task 4 | 04:17:27 | Preparing deployment: Preparing deployment (00:00:01)
  L Error: Failed to fetch variable '/.../zookeeper/zk-max-client-connections'
  from config server: Director is not configured with a config server
```

TODO example error on a credhub BOSH

The benefit of variables is they are easier to use than creating new Operator files. The downside of variables occurs if a deployment manifests uses variables, but a default value would have been fine. It can be tedious providing explicit variables for a new deployment at a time when you just don't know or care what good values might be.

Good defaults are better than variables. Operators can always modify or set values later with Operator files. The file are even named after us!

## Updates Are Not Atomic

When you run `bosh deploy`, the changes you've requested will not just atomically and magically be performed. They will take some amount of time and be performed in some specific order. There will be times during `bosh deploy` where some job templates may have been stopped and have not yet been restarted.

## Update Batches

The final top-level section of a deployment manifest is `update` and it specifies the batches of instances to update, and how long should the BOSH director wait before timing out and declaring that a `bosh deploy` has failed.

The `update` section for our `zookeeper-release/manifests/zookeeper.yml` manifest is:

```yaml
update:
  canaries: 2
  max_in_flight: 1
  canary_watch_time: 5000-60000
  update_watch_time: 5000-60000
```

Remember that we are deploying or updating five `zookeeper` instances. The `update.canaries` and `update.max_in_flight` values determine the batch sizes. Do we deploy/update one instance at a time, all of the instances at the same time, or in small batches?

In the `zookeeper` example, we will start any `bosh deploy` by updating the first two instances together. The remaining three instances will continue running as were previously happily doing. The first two instances being updated might fail. They are like a canary in a coal mine. If we update two instances and they fail, we will still have three instances running in our cluster.

If the canaries fail to update, the deployment will cease. As the Operator, you will commence the professional tradecraft of debugging.

If the canaries successfully update, then the instance group will be temporarily comprised of two updated instances and three older instances. It is important that the job templates running on these five instances are capable of running in this temporary configuration. The developers of the `zookeeper` BOSH release and job templates would have tested this, since they were the ones who suggested the `canaries: 2` update attribute.

The next batch of instances to be updated will be of size 1, the `max_in_flight` value. Then another batch of 1, and then the final `max_in_flight` batch of 1 instance. If any batch of instances fails to update the `bosh deploy` activity will cease.

Your deployment manifest can also have an `update` section in any instance group, which will override the top-level global `update`.

The `bosh deploy` command also has options that will override the `update` manifest attribute:

```
> bosh deploy -h
...
[deploy command options]
...
    --canaries=         Override manifest values for canaries
    --max-in-flight=    Override manifest values for max_in_flight
```

## Renaming An Instance Group

Over the lifetime of a deployment you might merge or split out job templates between instance groups. You might then want to rename the instance groups. This can be done using the deployment manifest.

**But**, if you simply change the name of the instance group the BOSH director will not know you wanted to rename an existing instance group. It will destroy the previous instance group, orphan its persistent disks, and then provision a new instance group with new persistent disks.

To **rename** the `zookeeper` instance group in our `zookeeper` deployment manifest, we add the `migrated_from` attribute to our deployment manifest.

```yaml
instance_groups:
- name: zk
  migrated_from:
  - name: zookeeper
  - name: zookeeper-instances
  ...
```

When we next run `bosh deploy`, the BOSH director will harmlessly rename any `zookeeper` or `zookeeper-instances` instances to their new name `zk`. On subsequent `bosh deploy` operations, with the instance group now called `zk`, the `migrated_from` attribute will be ignored.
