# Cloud Config Updates

The `bosh` CLI includes a `bosh update-cloud-config path/to/new-cloud-config.yml` command.

One approach to curating the `cloud-config` is to:

* download and save it to a file

    ```
    bosh cloud-config > cloud.yml
    ```

* edit it

    ```
    vi cloud.yml
    ```

* upload the changes

    ```
    bosh update-cloud-config cloud.yml
    ```

    This will show you the changes and you press `y` to continue:

    ```
    disk_types:
    - name: default
    -   disk_size: 3000
    +   disk_size: 5000
    ```

Once cloud config is updated, all existing deployments will be considered outdated, as indicated by `bosh deployments` command.

```
bosh deployments
```

The output would show that all deployments now have an outdated `cloud-config`:

```
Name       Release(s)       Stemcell(s)                          Team(s)  Cloud Config
zookeeper  zookeeper/0.0.7  bosh-...-ubuntu-trusty-go_agent/...  -        outdated
```

When you next deploy the `zookeeper` deployment it will show that the deployment's `disk_types` (merged in from the `cloud-config`) will be changing:

```
$ bosh deploy zookeeper-release/manifests/zookeeper.yml

  disk_types:
  - name: default
-   disk_size: 3000
+   disk_size: 5000

Continue? [yN]:
```

After a deployment has been re-deployed, the `bosh deployments` output will show it has the latest `cloud-config`:

```
$ bosh deployments

Name       Release(s)       Stemcell(s)                          Team(s)  Cloud Config
zookeeper  zookeeper/0.0.7  bosh-...-ubuntu-trusty-go_agent/...  -        latest
```

## Redeploy without a manifest

To re-deploy a deployment you will require the deployment manifest in a local file:

```
bosh deploy path/to/manifest.yml
```

But when the only changes to a deployment are in the shared `cloud-config`, we can try using the previously successfully deployed manifest that is stored within the BOSH director:

```
bosh deploy <(bosh manifest)
```

I'm not suggesting this is a "good idea". But its definitely "an idea". Remember to double check the proposed changes to the deployment.
