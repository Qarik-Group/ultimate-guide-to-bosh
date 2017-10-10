# Targeting BOSH environments

Throughout the Ultimate Guide to BOSH I have never explicitly referenced which BOSH director or which BOSH deployment name I am referring. I do this because adding `-d zookeeper` is repetitive, ugly, and repetitive.

## Targeting BOSH deployments

The deployment `name` attribute is to confirm that the `bosh deploy` command is updating the correct deployment. The `name` attribute must match the `BOSH_DEPLOYMENT` variable or the `bosh deploy -d name` value.

If our `name` attribute is set:

```yaml
name: zookeeper
```

Then the `bosh deploy` command must be executed in one of two ways.

1. Using `BOSH_DEPLOYMENT` environment variable:

    ```
    export BOSH_DEPLOYMENT=zookeeper
    bosh deploy zookeeper-release/manifests/zookeeper.yml
    ```

    This is the method used by default in all examples within the Ultimate Guide to BOSH. It makes the `bosh` CLI examples simpler.

2. With explicit `-d` flag option placed anywhere in the command:

    ```
    bosh -d zookeeper deploy zookeeper-release/manifests/zookeeper.yml
    ```

## Expected non-empty deployment name

```
$ bosh deploy zookeeper-release/manifests/zookeeper.yml
Using environment '10.0.0.4' as client 'admin'

Expected non-empty deployment name

Exit code 1
```
