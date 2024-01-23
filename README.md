# n8n with PostgreSQL backend

## Deployment to `docker-compose`
> This taken from [here](https://github.com/n8n-io/n8n/tree/master/docker/compose/withPostgres) and patched for PostgreSQL 15+.

In order this to work you have to create a **`.env`** file to be in the same folder as the docker compose descriptor file. **Be aware this file will be available to docker containers when `docker-compose up` is run**. ([More on this](https://docs.docker.com/compose/environment-variables/set-environment-variables/)). In this file there has to be at least assigned variables:
```
POSTGRES_NON_ROOT_USER=<n8n-nonroot-user>
POSTGRES_NON_ROOT_PASSWORD=<n8n-nonroot-password>
POSTGRES_DB=<n8n-database-name>
POSTGRES_USER=<root-pg-user> # usually named postgres
```

Also, by mapping the volume `./init-n8n-pg.sh:/docker-entrypoint-initdb.d/init-n8n-pg.sh` we get the script run on container startup (just by virtue of being placed at `/docker-entrypoint-initdb.d` folder) and our user to use with _n8n_ is created.

### Remote backend using a PgAAS

Actually, we don't need a `compose` setup here, but we will follow the previous guidelines for the sake of _consistency_. 

So, instead using the backend database as a docker compose service, we can decouple it by using a remote database. In this case, the configuration is the same but an example ([docker-compose-neon.yml](./docker-compose-neon.yml)) is provided to use [neon.tech](https://neon.tech). There can be issues on how to connect, as you might need a certificate instead to use the user/password combination. Refer to each backend documentation and **n8n** reference in any case.

Also mind the named of the environment variables in ([docker-compose-neon.yml](./docker-compose-neon.yml)) are prefixed as `NEON_` but their meaning is the same as pointed above. It is recommendable to add a new environment variable `[NEON_DB_]POSTGRESDB_HOST` pointing to the URL host (also used in the compose file linked previously).

## Deployment to kubernetes

### In-cluster PostgreSQL backend

To do this from the scratch we need a Postgres database in the cluster. We'll use a halm chart as there are plenty of options for postgres. We'll choose a simple one for this POC by installing the chart build by the cetic org, which can be found in [artifacthub](https://artifacthub.io/packages/helm/cetic/postgresql). Just follow the instructions to install.

Default values in `values.yml` are moslty fine for our purposes here. Database root user and password can be customized.

From the project root:
`helm install postgresn8n cetic/postgresql -f chart/pg-values.yaml`

No point in define the name to use a exported port as the _service_ template is hardcoded to use a service type `ClusterIP`. Read the documentation just after the installation to forward the port from the host to the container pod and connect to the database server with the postgres client of choice.

### n8n single instance

We chosen [this excellent char](https://artifacthub.io/packages/helm/open-8gears/n8n) in order to deploy the application in a minikube _cluster_ with default values but for the postgres parameters. They have to be those parameters we chosen for our backend. In the case of using the previously deployed Postgres backend, the IP of the postgres service has to be set.

From the project root:
`helm install n8n-poc oci://8gears.container-registry.com/library/n8n --version 0.20.1 -f chart/n8n-values.yaml`

As in the case of the postgresql deployment, we need to **forward the port** from the host to the container port in order to get host connectivity to the n8n instance from outside the cluster. That is documented similarly to the postgres deployment just after the installation.


