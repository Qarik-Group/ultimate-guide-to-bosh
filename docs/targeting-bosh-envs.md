# Targeting BOSH environments

Throughout the Ultimate Guide to BOSH I have never explicitly referenced which BOSH director or which BOSH deployment name I am referring. I do this because adding `-d zookeeper` is repetitive, ugly, and repetitive.

## Expected non-empty deployment name

```
$ bosh deploy manifests/zookeeper.yml
Using environment '10.0.0.4' as client 'admin'

Expected non-empty deployment name

Exit code 1
```
