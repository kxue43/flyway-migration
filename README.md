# Migrate MySQL Database with Flyway Docker Image

This repo provides a GitHub Docker container action that uses `Flyway` to migrate MySQL database. Since
`Flyway` also supports other database engines, this repo can be forked and modified easily to create actions
that work with other databases.

The `Flyway` Docker image version is chosen to be `7.15`, as higher version of `Flyway` drops support for MySQL
`5.7.*`.

## Inputs

This action expects the following inputs. `endpoint`, `port`, `user`, `password` and `locations` are passed to
the `flyway` executable in the docker container through environment variables. `defaultSchema` and `schemas` are
passed as CLI options of the form `-defaultSchema=${{ inputs.defaultSchema }}` and `-schemas=${{ inputs.schemas }}`.

### endpoint:

- description: Endpoint URL to the MySQL database instance. *Cannot be loopback IP address*.

- type: string

- required: true

### port:

- description: Port number of the MySQL database instance.

- type: string

- default: "3306"

### user:

- description: MySQL database user, default to `root`.

- type: string

- default: 'root'

### password:

- description: Password of the user.

- type: string

- required: true

### locations:

- description: The `-locations` CLI option passed to Flyway. For SQL-based migrations, the value could be
  `filesystem:<RELATIVE_PATH_TO_SQL_SCRIPT_FOLDER>`. The "relative path" is relative to the root directory of
  repo that uses this action. This is because GitHub Actions runs a Docker container by mounting the repo as
  the `/github/workspace` folder inside the container and automatically changes `WORKDIR` to `/github/workspace`.

- type: string

- required: true

### defaultSchema:

- description: The default schema.

- type: string

- required: true

### schemas:

- description: Comma-separated list of database schemas.

- type: string

- required: true

## Example

```yaml
name: Migrate DB with flyway
on: workflow_dispatch
jobs:
  migrate-db:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: kxue43/flyway-migration@v1
        with:
          endpoint: ${{ secrets.MYSQL_ENDPOINT }}
          password: ${{ secrets.MYSQL_ROOT_PASSWORD }}
          locations: filesystem:./database/flywayfiles/sql
          defaultSchema: test_db_1
          schemas: test_db_1,test_db_2
```

This example assumes that *in the repo that uses this action*, there is a folder `database/flywayfiles/sql` that
contains all the SQL-based migration scripts for `Flyway`.

## Loopback IP address

The `endpoint` input parameter cannot be loopback IP address such as `127.0.0.1`. This is because `127.0.0.1` refers
to the container that `Flyway` is running in, not the host machine (in this case the GitHub Actions runner VM).
Therefore, this action cannot be used for migrating a MySQL service container on the runner VM. In order to do that,
use the following docker commands directly.

```yaml
steps:
  - run: docker run --name mysql -d -p 3306:3306 -e "MYSQL_ROOT_PASSWORD=password" mysql:5.7.32
  - run: >
    docker run --link mysql --rm
    -v /var/run/docker.sock:/var/run/docker.sock
    -v $GITHUB_WORKSPACE/****:/flyway/sql
    -v $GITHUB_WORKSPACE/****:/flyway/conf
    flyway/flyway:7.15 migrate
    -url=jdbc:mysql://mysql:3306?allowPublicKeyRetrieval=True
    -user=root
    -password=password
    -locations=filesystem:/flyway/sql/
    -connectRetries=3
```
