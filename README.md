# n8n custom deploymments

## Deployment to Docker Compose
> This taken from [https://github.com/n8n-io/n8n/tree/master/docker/compose/withPostgres](https://github.com/n8n-io/n8n/tree/master/docker/compose/withPostgres) and patched for PostgreSQL 15+.

In order for this to work you have to create a **`.env`** file to be in the same folder as the docker compose descriptor file. **Be aware this file will be available to docker containers when `docker-compose up` is run** ([more on this](https://docs.docker.com/compose/environment-variables/set-environment-variables/)). In this file there has to be at least assigned variables:
```
POSTGRES_NON_ROOT_USER=<n8n-nonroot-user>
POSTGRES_NON_ROOT_PASSWORD=<n8n-nonroot-password>
POSTGRES_DB=<n8n-database-name>
POSTGRES_USER=<root-pg-user> # usually named postgres
```

Also, by mapping the volume `./init-n8n-pg.sh:/docker-entrypoint-initdb.d/init-n8n-pg.sh` we get the script run on container startup (just by virtue of being placed at `/docker-entrypoint-initdb.d` folder) and our user to use with _n8n_ is created. This script [init-n8n-pg.sh](init-n8n-pg.sh) just makes things easier for us, by creating the database environment for n8n (database, user, authentication, privileges).

### Remote backend using a PgAAS

Actually, we don't need a `compose` setup here. The aim of this POC is deploy and prove viability of n8n _alone_. As per [the documentation](https://docs.n8n.io/hosting/supported-databases-settings/), n8n needs a database to work, but for simplicity we can get an already running database server from elsewhere, without us having to provide one.

So, instead using the backend database as a docker compose service, we can decouple it by using a remote database. In this case, the configuration is the same but an example ([docker-compose-neon.yml](./docker-compose-neon.yml)) is provided to use [neon.tech](https://neon.tech). There can be issues on how to connect, as you might need a certificate instead to use the user/password combination. Refer to each DbAAS documentation and [this **n8n** reference](https://docs.n8n.io/hosting/supported-databases-settings/#tls) in any case.

Also mind the named of the environment variables in ([docker-compose-neon.yml](./docker-compose-neon.yml)) are prefixed as `NEON_` but their meaning is the same as pointed above. It is advisable to add a new environment variable `[NEON_DB_]POSTGRESDB_HOST` pointing to the URL host (also used in the compose file linked previously).

In addition, we need to create _manually_ the database and user (and its password) and grant privileges _before_ starting our n8n instance (otherwise we'll get a `DatabaseError` error).


## Deployment to Kubernetes

### TL;DR

After updating the `yaml` files with the right values
```sh
$ docker-compose -f docker-compose-pg-minikube.yml up -d # to start sftp and kafke if necessary in the same minikube network

$ helm install postgresn8n cetic/postgresql -f chart/pg-values.yaml # postgres backend for n8n in minikube

$ kubectl port-forward --namespace default svc/postgresn8n-postgresql 15432:5432 &

psql --host 127.0.0.1 -p 15432 -U postgres -d postgres
postgres@127:postgres> create user n8n with password 'n8np0stgr3s';
CREATE ROLE
postgres@127:postgres> create database n8ndb;
CREATE DATABASE
postgres@127:postgres> \c n8ndb postgres
You are now connected to database "n8ndb" as user "postgres"
postgres@127:n8ndb> grant all privileges on database n8ndb to n8n;
GRANT
postgres@127:n8ndb> \q
Goodbye!

$ helm install n8n-poc oci://8gears.container-registry.com/library/n8n --version 0.20.1 -f chart/n8n-values.yaml

$ kubectl port-forward --namespace default svc/n8n-poc 8888:80 &

# connect to n8n as http://localhost:8888
```

### In-cluster PostgreSQL backend

To do this from scratch we need a Postgres database in the cluster. We'll use a helm chart as there are plenty of options for postgres. We'll choose a simple one for this POC by installing the chart build by the cetic org, which can be found in [artifacthub](https://artifacthub.io/packages/helm/cetic/postgresql). Just follow the instructions to install.

Default values in `values.yml` are mostly fine for our purposes here. Database root user and password can be customized.

From the project root:
`helm install postgresn8n cetic/postgresql -f chart/pg-values.yaml`

No need to define the name to use a exported port as the _service_ template is hardcoded to use a service type `ClusterIP`. Read the documentation just after the installation to forward the port from the host to the container pod and connect to the database server with the postgres client of choice, which can be done by running:

`kubectl port-forward --namespace default services/<postgresql-service-name> 15432:5432 &`

This command is better run in background (otherwise it will block the terminal) and makes the requests _on the host_ to port `15432` get redirected to the port `5432` in the cluster.


### n8n single instance

We chosen [this excellent chart](https://artifacthub.io/packages/helm/open-8gears/n8n) for its support and simplicity in order to deploy the application in a minikube _cluster_ with default values but for the Postgres parameters. They have to be those parameters we chosen for our backend. In the case of using the previously deployed Postgres backend, the IP of the Postgres service has to be set.

From the project root, run:
`helm install n8n-poc oci://8gears.container-registry.com/library/n8n --version 0.20.1 -f chart/n8n-values.yaml`

As in the case of the postgresql deployment, we need to **forward the port** from the host to the container port in order to get host connectivity to the n8n instance from outside the cluster. That is documented similarly to the postgres deployment just after the installation.

### n8n limited high availability

Install a redis instance using a helm chart

```sh
$ helm install n8nredis oci://registry-1.docker.io/bitnamicharts/redis
```

Then, follow the instructions printed after emitting the previous command. In short

```sh
# get the password for redis
$ kubectl get secret --namespace default n8nredis -o jsonpath="{.data.redis-password}" | base64 -d
$ export REDIS_PASSWORD=$(kubectl get secret --namespace default n8nredis -o jsonpath="{.data.redis-password}" | base64 -d)

# deploy a redis client in the cluster as a pod
$ kubectl run --namespace default redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:7.2.4-debian-11-r2 --command -- sleep infinity

# access to the pod and run the redis-cli program as usual to check if n8nredis-master and n8nredis-replicas are up and running
$ kubectl exec -i -t redis-client -n default -- bash

# forward ports to be able to access redis from this world (here we use port 16379 to avoid confusions)
$ kubectl port-forward --namespace default svc/n8nredis-master 16379:6379 &

# deploy n8n instance to kubernetes
helm install n8n-poc oci://8gears.container-registry.com/library/n8n --version 0.20.1 -f <custom-values.yaml>

# forward n8n port to be able to access from terminal (in the case of minikube at least)
kubectl port-forward --namespace default svc/n8n-poc 8888:80 &

# then, form a browser, http://localhost:8888/ will bring n8n

```

> **Note** To create a **n8n credential** in the deployed n8n to access to n8n API from the workflows, the address of the _Base URL_ has to be **the IP of the main service in the cluster** and not the address n8n is accessed from the outside. In the case of our tests with _minikube_ we access to n8n from the browser through `http://localhost:8888` (arbitrary post), but to set the credential the _Base URL_ is `http://10.109.132.115:80/api/v1`, being the IP the service address.

#### Main instance autoscaling
After deploying redis in the cluster, just update the `values.yaml` file to allow n8n work in [queue mode](https://docs.n8n.io/hosting/scaling/queue-mode/):
- set `.Values.autoscaling.enabled` to true
- choose the number of workers to be spawned in the cluster
- configure redis host parameters to be n8n able to communicate with it

> This configuration will _autoscale_ the main instance but **not** the workers which is a bit counterintuitive as with high load you will see many main instances spawning but the workers keeps in a stable number. This might not be the desired behaviour.

#### Worker instances autoscaling
As it is the workers [who run the workflows](https://docs.n8n.io/hosting/scaling/queue-mode/#how-it-works) (but those triggered [manually](https://docs.n8n.io/hosting/scaling/queue-mode/#running-n8n-with-queues) are still run by the main instance), it can be considered to be the workers which _should really be autoscaled_. In order to do this one:
- to keep just one main instance, let's set `.Values.autoscaling.enabled` to `false`
- add a new configuration to your _values_ file just like
```yaml
workerResources:
  limits:
    cpu: 1000m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 256Mi
```
- deploy n8n as described [above](#n8n-limited-high-availability)
- then, _apply_ `hpa-workder.yaml` (`kubectl apply -f hpa-worker.yaml`). You might need to update the `.spec.metrics` properties in the `hpa-worker.yaml` file to adapt to your cluster and your needs

With this deployment, there will be _always_ just one main n8n instance and, depending on the workload, multiple workers will be spawned and removed automatically to adapt resources to the needs.

We've observed that is necessary to explicitly define resource limits for both main and worker instances to be able to send metrics to the cluster's `metrics-server`. Otherwise, the metrics server won't get metrics and _horizontal pod autoscaler (HPA)_ won't be able to autoscale resources depending on workloads.

The values for the `.Values.resources` and `.Values.workerResources` might need some tweak as well to adapt to the particular cluster.

## Workload testing

Using the above configuration with workers autoscaling (max to 8 pods) and verbosity of n8n main and workers instances to _info_ (it might slow the workflow processing), a performance/workload test was carried out:
- macos sonoma 14.3, 2 GHz Quad-Core Intel Core i5 and 16Gb RAM
- minikube cluster with postgresql, redis and n8n in queue mode worker autoscaling deployed
- two **simultaneous** message producers written in python, publishing 11K and 10K messages with maximum delay of 0.2 and 0.3 seconds each (from terminal)
- message broker was the same redis instance used by n8n to support queue mode
- the [processing workflow](https://github.com/telekosmos/n8n-poc/blob/main/workflows/redis2s3%20-%20user%20behaviour%20events.json) is a simple workflow to pick the message, do some simple transformations on messages and send them to S3.
- the messages published were _fake_ user behaviours. In order to do it, you can check [this PR](https://github.com/telekosmos/fake-data-producer-for-kafka/pull/1) and also the files involved in the script related ([user producer](https://github.com/telekosmos/fake-data-producer-for-kafka/blob/main/userproducer.py) and [user behaviours producer](https://github.com/telekosmos/fake-data-producer-for-kafka/blob/main/userbehaviorproducer.py))

Overall, performance was sufficient to good. All messages where picked up by the n8n autoscaling instance and sent in real time to S3, with no substantial delay observed.

While processing the messages the n8n designer got a bit unresponsive, so in a production environment the deployment should be devoted only to run workflows and leaving the development (and ideally staging) instances for workflow development.

All of the workers were over the top on max memory resource used (they went red in `k9s`). As mentioned, a more custom tweak based on trials might be necessary.
