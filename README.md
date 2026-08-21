# KC

A helper script to connect to various databases and manage utility jobs in Kubernetes clusters.

## Features
- Connect to PostgreSQL, MongoDB, and Redis databases through kubernetes job easily.
- **No need to store DB credentials in your local**
- Clean up utility jobs you own across namespaces from your config.
- Supports context switching using context aliases from your config.

## Requirements
- macOS or Linux
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [yq](https://github.com/mikefarah/yq)
- sed, awk, tr, git, grep, xargs, sort, uniq

## Installation

Add `kc` to your `PATH` for easy access. For example, if you cloned or downloaded `kc` to `~/work/kc`:

```sh
chmod +x ~/work/kc
export PATH="$PATH:$HOME/work/kc"
```

To make this change permanent, add the `export` line to your `.bashrc`, `.zshrc`, or equivalent shell profile.

## Usage

```sh
kc [options] <subcommand> [args]
```

### Options
- `-c`, `--context`  Specify the Kubernetes context to use. Can be a full context name or an alias set in `~/.kc/config.yaml`.
- `-<context>`       Short hand of `-c <context>. To switch context.
- `-pf`, `--port-forward`
                     To portforward pg, mongo, redis to make it accessible from local port.
- `-v`, `--verbose`  Enable verbose mode to print commands being executed.

### Subcommands
- `pg <app-name>`      Connect to PostgreSQL database of the specified app.
- `mongo <app-name>`   Connect to MongoDB database of the specified app.
- `redis <app-name>`   Connect to Redis database of the specified app.
- `util <namespace>`   Create a utility job in the specified namespace.
- `cleanup`            Delete all utility jobs you own across all namespaces defined in your config.
- `<anycommand>`       Execute whatever command you want. Usually used with -c/--context option.

### Examples

Connect to PostgreSQL:
```sh
kc pg cev2
```

Connect to MongoDB:
```sh
kc mongo dte
```

Connect to Redis:
```sh
kc redis spal
```

Create a utility job in a namespace:
```sh
kc util a000096
```

Clean up your utility jobs:
```sh
kc cleanup
```

Using context option to switch context before executing subcommand:
```sh
kc -c dev pg cev2             # switch context with alias configured in .kc/config.yaml
kc -c gke_project_dev pg cev2 # switch context using full context

# Of course you can rename your kube context to a shorter name centraly using the following command
# kubectl config rename-context <current-context-name> <new-context-name>
# For example
kubectl config rename-context gke_project_dev dev
```

![kcc-demo](docs/kc-demo.gif)

## Configuration

Create a config file at `<path-to-kc>/.kc/config.yaml` with the following structure:

```yaml
# General settings. Optional.
config:
  # How long a utility job lives, in seconds. Defaults to 86400 (24 hours).
  # The job stops itself after this long even if kc never gets to clean up.
  ttl: 86400
  # Image used for each kind of utility job. Every key is optional and
  # independent; any key you omit keeps the default shown here.
  images:
    pg: postgres:latest
    mongo: arunvelsriram/utils
    redis: redis:latest
    util: arunvelsriram/utils
    # -pf/--port-forward runs socat instead of the database client, so it uses
    # this image rather than images.pg/mongo/redis.
    pf: alpine/socat
# Map of context aliases to full context names.
ctxs:
  dev: gke_project_dev
  prod: gke_project_prod
# Map of app names to their deployment, namespace, and database config.
apps:
  cev2:
    # deployment is to lookup deployment and container from the cluster.
    # if multiple deployment matched, only use the first one
    deployment: constraintenricherv2
    namespace: platform-apps
    # Postgresql config, map of key to ENV variable key in kubernetes deployment
    # host, user, password and dbname is required. 
    # port is optional with 5432 as default
    pg:
      host: DB_HOST
      port: DB_PORT
      user: DB_USER
      password: DB_PASSWORD
      dbname: DB_NAME
  dte:
    deployment: driver-tier-evaluator-dte-actor
    namespace: a000151
    # MongoDB config, map of key to ENV variable key in kubernetes deployment.
    # host, user, password, dbname is required
    # port is optional with 27017 as default
    mongo:
      host: MONGO_DB_HOSTS
      user: MONGO_DB_USERNAME
      password: MONGO_DB_PASSWORD
      dbname: MONGO_DB_NAME
  spal:
    deployment: sirspamalot-actor-server
    namespace: a000096
    # Redis config, map of key to ENV variable key in kubernetes deployment.
    # host is required
    # password is optional
    # port is optional with 6379 as default
    redis:
      host: CACHE_MASTER_REDIS_HOST
      password: CACHE_REDIS_PASSWORD
  mpgw:
    deployment: gateway-kong-kong
    namespace: mp-gateway
    redis:
      host: gateway-redis-master
```

## Behind the scene
### 1. Lookup Deployment
kc uses `deployment` to lookup the deployment and cluster from the cluster. Let's say we have the following config
```yaml
apps:
  spal:
    deployment: sirspamalot-actor-server
    namespace: a000096
    redis:
      host: CACHE_MASTER_REDIS_HOST
      password: CACHE_REDIS_PASSWORD
```
Based on above config, kc will look for deployment that has name CONTAINS `sirspamalot-actor-server` and pick the first one.

And then, it will get the environment variable from the container from that deployment which name CONTAINS `sirspamalot-actor-server`.

The previous config will match with the following deployment
```yaml
kind: Deployment
metadata:
  name: sirspamalot-actor-server-92427a6d-10e8
  namespace: a000096
spec:
  template:
    spec:
      containers:
        - name: sirspamalot-actor-server
          env:
            - name: CACHE_MASTER_REDIS_HOST
              value: redis-sirspamalot-write.service.i-cgk.consul
            - name: CACHE_REDIS_PASSWORD
              value: password123 # sample value
```
### 2. Resolve DB config
kc will resolve to use config from env variable in the deployment.
```yaml
redis:
  host: redis-sirspamalot-write.service.i-cgk.consul # coming from CACHE_MASTER_REDIS_HOST env variable
  password: password123 # coming from CACHE_REDIS_PASSWORD env variable
```

A value only goes through this lookup when it names an env variable that the
deployment's container actually has. Anything else is used as written, so you
can hardcode a field the deployment does not expose - like `mpgw` above, whose
redis host is the literal `gateway-redis-master`. The two can be mixed freely
within one app:
```yaml
redis:
  host: gateway-redis-master   # used as-is
  password: CACHE_REDIS_PASSWORD # read from the deployment
```
### 3. Spawn utility jobs will all the details
kc will run the following commands for redis
```bash
kubectl create job kc-util-redis-<gitusername> --image redis:latest -n $ns --dry-run=client -o yaml -- sleep 86400 \
  | kubectl set env --local -f - -o yaml REDISCLI_AUTH=password123 \
  | yq '.spec.activeDeadlineSeconds = 86400 | .spec.ttlSecondsAfterFinished = 0 | .spec.backoffLimit = 0' \
  | kubectl apply -f -
kubectl wait --for=condition=Ready pod -l job-name=kc-util-redis-<gitusername> -n $ns --timeout=10s
kubectl exec -it job/kc-util-redis-<gitusername> -n $ns -- redis-cli -c -h redis-bff-server.consul -p 6379
```
The job is deleted when you exit the session, including on Ctrl-C or when the terminal is closed. As a backstop it also deletes itself 24 hours after creation, even if kc never gets the chance to clean up. Set `config.ttl` to change that window; the value replaces `86400` in both places above.

`redis:latest` above is likewise the default, replaced by `config.images.redis` when that is set. Each command has its own key (`pg`, `mongo`, `redis`, `util`, and `pf` for the socat image `-pf` uses), so overriding one leaves the others alone.
