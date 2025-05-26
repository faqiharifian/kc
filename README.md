# KC

A helper script to connect to various databases and manage utility pods in Kubernetes clusters.

## Features
- Connect to PostgreSQL, MongoDB, and Redis databases through kubernetes pod easily.
- **No need to store DB credentials in your local**
- Clean up utility pods you own across namespaces from your config.
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
- `-v`, `--verbose`  Enable verbose mode to print commands being executed.

### Subcommands
- `pg <app-name>`      Connect to PostgreSQL database of the specified app.
- `mongo <app-name>`   Connect to MongoDB database of the specified app.
- `redis <app-name>`   Connect to Redis database of the specified app.
- `util <namespace>`   Create a utility pod in the specified namespace.
- `cleanup`            Delete all utility pods you own across all namespaces defined in your config.

### Examples

Connect to PostgreSQL:
```sh
kc pg oms
```

Connect to MongoDB:
```sh
kc mongo content
```

Connect to Redis:
```sh
kc redis bff
```

Create a utility pod in a namespace:
```sh
kc util kube-system
```

Clean up your utility pods:
```sh
kc cleanup
```

Using context option to switch context before executing subcommand:
```sh
kc -c dev pg oms             # switch context with alias configured in .kc/config.yaml
kc -c gke_project_dev pg oms # switch context using full context

# Of course you can rename your kube context to a shorter name centraly using the following command
# kubectl config rename-context <current-context-name> <new-context-name>
# For example
kubectl config rename-context gke_project_dev dev
```

## Configuration

Create a config file at `<path-to-kc>/.kc/config.yaml` with the following structure:

```yaml
# Map of context aliases to full context names.
ctxs:
  dev: gke_project_dev
  prod: gke_project_prod
# Map of app names to their deployment, namespace, and database config.
apps:
  oms:
    # deployment is to lookup deployment and container from the cluster.
    # if multiple deployment matched, only use the first one
    deployment: oms-server
    namespace: booking-experience
    # Postgresql config, map of key to ENV variable key in kubernetes deployment
    # host, user, password and dbname is required. 
    # port is optional with 5432 as default
    pg:
      host: DB_HOST
      port: DB_PORT
      user: DB_USER
      password: DB_PASSWORD
      dbname: DB_NAME
  content:
    deployment: content-management
    namespace: content-management
    # MongoDB config, map of key to ENV variable key in kubernetes deployment.
    # host, user, password, dbname is required
    # port is optional with 27017 as default
    mongo:
      host: MONGO_DB_HOSTS
      user: MONGO_DB_USERNAME
      password: MONGO_DB_PASSWORD
      dbname: MONGO_DB_NAME
  bff:
    deployment: bff-server
    namespace: user-facing
    # Redis config, map of key to ENV variable key in kubernetes deployment.
    # host is required
    # password is optional
    # port is optional with 6379 as default
    redis:
      host: CACHE_MASTER_REDIS_HOST
      password: CACHE_REDIS_PASSWORD
  gw:
    deployment: gateway-kong-kong
    namespace: api-gateway
    redis:
      # You can also use a static value, just put random valid deployment and namespace.
      host: gateway-redis-master
```

## Behind the scene
### 1. Lookup Deployment
kc uses `deployment` to lookup the deployment and cluster from the cluster. Let's say we have the following config
```yaml
apps:
  bff:
    deployment: bff-server
    namespace: user-facing
    redis:
      host: CACHE_MASTER_REDIS_HOST
      password: CACHE_REDIS_PASSWORD
```
Based on above config, kc will look for deployment that has name CONTAINS `bff-server` and pick the first one.

And then, it will get the environment variable from the container from that deployment which name CONTAINS `bff-server`.

The previous config will match with the following deployment
```yaml
kind: Deployment
metadata:
  name: bff-server-92427a6d-10e8
  namespace: user-facing
spec:
  template:
    spec:
      containers:
        - name: bff-server
          env:
            - name: CACHE_MASTER_REDIS_HOST
              value: redis-bff-server.consul
            - name: CACHE_REDIS_PASSWORD
              value: password123 # sample value
```
### 2. Resolve DB config
kc will resolve to use config from env variable in the deployment.
```yaml
redis:
  host: redis-bff-server.consul # coming from CACHE_MASTER_REDIS_HOST env variable
  password: password123 # coming from CACHE_REDIS_PASSWORD env variable
```
### 3. Spawn utility pods will all the details
kc will run the following command for redis
```bash
kubectl run util-redis-<gitusername> --rm -it --image redis:latest --restart Never -n $ns --env REDISCLI_AUTH=password123 -- redis-cli -c -h redis-bff-server.consul -p 6379
```