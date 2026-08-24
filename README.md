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
kc pg api
```

Connect to MongoDB:
```sh
kc mongo orders
```

Connect to Redis:
```sh
kc redis sessions
```

Create a utility job in a namespace:
```sh
kc util sessions
```

Clean up your utility jobs:
```sh
kc cleanup
```

Using context option to switch context before executing subcommand:
```sh
kc -c dev pg api             # switch context with alias configured in .kc/config.yaml
kc -c gke_project_dev pg api # switch context using full context

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
  api:
    # deployment is to lookup deployment and container from the cluster.
    # if multiple deployment matched, only use the first one
    deployment: my-api
    namespace: backend
    # Postgresql config, map of key to ENV variable key in kubernetes deployment
    # host, user, password and dbname is required. 
    # port is optional with 5432 as default
    pg:
      host: DB_HOST
      port: DB_PORT
      user: DB_USER
      password: DB_PASSWORD
      dbname: DB_NAME
  orders:
    deployment: orders-service
    namespace: orders
    # MongoDB config, map of key to ENV variable key in kubernetes deployment.
    # host, user, password, dbname is required
    # port is optional with 27017 as default
    mongo:
      host: MONGO_HOST
      user: MONGO_USER
      password: MONGO_PASSWORD
      dbname: MONGO_DBNAME
  sessions:
    deployment: sessions-service
    namespace: sessions
    # Redis config, map of key to ENV variable key in kubernetes deployment.
    # host is required
    # password is optional
    # port is optional with 6379 as default
    redis:
      host: REDIS_HOST
      password: REDIS_PASSWORD
  gateway:
    deployment: gateway-proxy
    namespace: gateway
    redis:
      host: redis-gateway-master
```

## Behind the scene
### 1. Lookup Deployment
kc uses `deployment` to lookup the deployment and cluster from the cluster. Let's say we have the following config
```yaml
apps:
  sessions:
    deployment: sessions-service
    namespace: sessions
    redis:
      host: REDIS_HOST
      password: REDIS_PASSWORD
```
Based on above config, kc will look for deployment that has name CONTAINS `sessions-service` and pick the first one.

And then, it will get the environment variable from the container from that deployment which name CONTAINS `sessions-service`.

The previous config will match with the following deployment
```yaml
kind: Deployment
metadata:
  name: sessions-service-92427a6d-10e8
  namespace: sessions
spec:
  template:
    spec:
      containers:
        - name: sessions-service
          env:
            - name: REDIS_HOST
              value: redis-sessions.internal
            - name: REDIS_PASSWORD
              value: password123 # sample value
```
### 2. Resolve DB config
kc will resolve to use config from env variable in the deployment.
```yaml
redis:
  host: redis-sessions.internal # coming from REDIS_HOST env variable
  password: password123 # coming from REDIS_PASSWORD env variable
```

A value only goes through this lookup when it names an env variable that the
deployment's container actually has. Anything else is used as written, so you
can hardcode a field the deployment does not expose - like `gateway` above, whose
redis host is the literal `redis-gateway-master`. The two can be mixed freely
within one app:
```yaml
redis:
  host: redis-gateway-master   # used as-is
  password: REDIS_PASSWORD # read from the deployment
```
### 3. Spawn utility jobs will all the details
kc will run the following commands for redis
```bash
kubectl create job kc-util-redis-<gitusername> --image redis:latest -n $ns --dry-run=client -o yaml -- sleep 86400 \
  | kubectl set env --local -f - -o yaml REDISCLI_AUTH=password123 \
  | yq '.spec.activeDeadlineSeconds = 86400 | .spec.ttlSecondsAfterFinished = 0 | .spec.backoffLimit = 0' \
  | kubectl apply -f -
kubectl wait --for=condition=Ready pod -l job-name=kc-util-redis-<gitusername> -n $ns --timeout=10s
kubectl exec -it job/kc-util-redis-<gitusername> -n $ns -- redis-cli -c -h redis-sessions.internal -p 6379
```
The job is deleted when you exit the session, including on Ctrl-C or when the terminal is closed. As a backstop it also deletes itself 24 hours after creation, even if kc never gets the chance to clean up. Set `config.ttl` to change that window; the value replaces `86400` in both places above.

`redis:latest` above is likewise the default, replaced by `config.images.redis` when that is set. Each command has its own key (`pg`, `mongo`, `redis`, `util`, and `pf` for the socat image `-pf` uses), so overriding one leaves the others alone.
