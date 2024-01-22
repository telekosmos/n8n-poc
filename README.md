# n8n with PostgreSQL backend

## Using `docker-compose`
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

