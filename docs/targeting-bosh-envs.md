# Targeting BOSH Environments and Deployments

Many `bosh` CLI commands require you to explicitly reference which BOSH deployment name, and the details about the BOSH environment (specifically the BOSH director API).

You can provide the full array of information using `bosh` CLI flags:

```
bosh instances \
    --deployment    zookeeper \
    --environment   https://192.168.50.6:25555 \
    --ca-cert       path/to/root-ca \
    --client        admin \
    --client-secret password
```

You can avoid typing secrets into your shell by dynamically extracting them from files:

```
bosh instances \
    --deployment    zookeeper \
    --environment   https://192.168.50.6:25555 \
    --ca-cert       "$(bosh int path/to/bosh/creds.yml --path /director_ssl/ca)" \
    --client        admin \
    --client-secret "$(bosh int path/to/bosh/creds.yml --path /admin_password)"
```

Alternately, each of the flags above can be declared using an environment variable.

``` hl_lines="7"
export BOSH_DEPLOYMENT=zookeeper
export BOSH_ENVIRONMENT=https://192.168.50.6:25555
export BOSH_CA_CERT="$(bosh int path/to/bosh/creds.yml --path /director_ssl/ca)"
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET="$(bosh int path/to/bosh/creds.yml --path /admin_password)"

bosh instances
```

## Alias a BOSH Environment

It will be quickly tiresome to type all this information. No one does that. Continuous integration scripts might do this, but not you nor I. We like short cuts.

The `bosh` CLI allows us to alias each BOSH environment URL and root CA with a meaningful name. For example, in the tutorial for setting up a local VirtualBox BOSH environment we used the alias `vbox`:

``` hl_lines="1"
bosh alias-env vbox \
    --environment https://192.168.50.6:25555 \
    --ca-cert "$(bosh int path/to/vbox/creds.yml --path /director_ssl/ca)"
```

## Login to a BOSH Environment

Once you have aliased an environment you can also cache the authentication credentials for that environment.

That is the most elaborate way I could describe "log in":

```
export BOSH_ENVIRONMENT=vbox
bosh login
```

Or if you want to use `--environment` flag:

```
bosh --environment vbox login
```

## Simplified Targeting

After aliasing and logging in to an environment, we can now simplify all our commands.

We can now express our `bosh instances` command from above with flags:

```
bosh instances \
    --deployment    zookeeper \
    --environment   vbox
```

Or with shortened flags:

```
bosh instances \
    -d zookeeper \
    -e vbox
```

We can also place the flags before the subcommand:

```
bosh -e vbox -d zookeeper instances
```

Or, we can use environment variables:

```
export BOSH_ENVIRONMENT=vbox
export BOSH_DEPLOYMENT=zookeeper

bosh instances
```

{==

I personally prefer using the `BOSH_ENVIRONMENT` and `BOSH_DEPLOYMENT` environment variables to target BOSH environments and their deployments.

==}

Especially for you reading the Ultimate Guide to BOSH, I felt that adding `-e environment-alias -d deployment-name` to every command in every example was repetitive, verbose, and repetitive.

## Available Aliased Environments

If you've forgotten what aliases you've given your environments you can list them all:

```
bosh envs
```

## Forgetting the Environment Alias

Sometimes you will forget to provide the environment alias. This is not a career-altering mistake.

``` hl_lines="2"
> bosh deploy manifests/zookeeper.yml
Expected non-empty Director URL

Exit code 1
```

Run the command again with the `-e` flag or set the `BOSH_ENVIRONMENT` environment.

## Forgetting the Deployment Name

Sometimes you will forget to provide the deployment name. You probably also forgot the environment alias.

``` hl_lines="4"
> bosh deploy zookeeper-release/manifests/zookeeper.yml
Using environment '10.0.0.4' as client 'admin'

Expected non-empty deployment name

Exit code 1
```

Run the command again with the `-d` flag or set the `BOSH_DEPLOYMENT` environment.
