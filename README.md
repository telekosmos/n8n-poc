# n8n with PostgreSQL in a compose

> This taken from [here](https://github.com/n8n-io/n8n/tree/master/docker/compose/withPostgres) and patched for PostgreSQL 15+.

In order this to work you have to create a `.env` file to be in the same folder as the docker compose descriptor file. **Mind this file will be available to docker containers when we run `docker-compose up`**. ([More on this](https://docs.docker.com/compose/environment-variables/set-environment-variables/)). In this file there has to be at least `POSTGRES_NON_ROOT_PASSWORD`, `POSTGRES_NON_ROOT_USER`, `POSTGRES_DB` and `POSTGRES_USER`.

Also, by mapping the volume `./init-n8n-pg.sh:/docker-entrypoint-initdb.d/init-n8n-pg.sh` we get the script run on container startup (just by virtue of being placed at `/docker-entrypoint-initdb.d` folder) and our user to use with _n8n_ is created.

