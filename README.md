PgBouncer Docker image
======================

This is a minimal PgBouncer image, based on Alpine Linux.

Features:

* Very small, quick to pull (just 8MB)
* Using LibreSSL
* Configurable using environment variables
* Uses standard Postgres port 5432, to work transparently for applications.
* MD5 authentication by default.
* `/etc/pgbouncer/pgbouncer.ini` and `/etc/pbbouncer/userlist.txt` are auto-created if they don't exist.


Why using PgBouncer
-------------------

PostgreSQL connections take up a lot of memory ([about 10MB per connection](http://hans.io/blog/2014/02/19/postgresql_connection)). There is also a significant startup cost to establish a connection with TLS, hence web applications gain performance by using persistent connections.

By placing PgBouncer in between the web application and the actual PostgreSQL database, the memory and start-up costs are reduced. The web application can keep persistent connections to PgBouncer, while PgBouncer only keeps a few connections to the actual PostgreSQL server. It can reuse the same connection for multiple clients.


Available tags
--------------

Base images:

- `1.8.1` ([Dockerfile](https://github.com/edoburu/docker-pgbouncer/blob/master/Dockerfile)) - Default and latest version.

Images are automatically rebuild on Alpine Linux updates.


Usage
-----

```sh
docker run --rm \
    -e DATABASE_URL="postgres://user:pass@postgres-host/database" \
    -e POOL_MODE=session \
    -e SERVER_RESET_QUERY="DISCARD ALL" \
    -e MAX_CLIENT_CONN=100 \
    -p 5432:5432
    edoburu/pgbouncer
```

Or using separate variables:

```sh
docker run --rm \
    -e DB_USER=user \
    -e DB_PASSWORD=pass \
    -e DB_HOST=postgres-host \
    -e DB_NAME=database \
    -e POOL_MODE=session \
    -e SERVER_RESET_QUERY="DISCARD ALL" \
    -e MAX_CLIENT_CONN=100 \
    -p 5432:5432
    edoburu/pgbouncer
```

Connecting should work as expected:

```sh
psql 'postgresql://user:pass@localhost/dbname'
```

Almost all settings found in the [pgbouncer.ini](https://pgbouncer.github.io/config.html) can be defined as environment variables, except a few that make little sense in a Docker environment (like port numbers, syslog and pid settings). See the [startup script](https://github.com/edoburu/docker-pgbouncer/blob/master/startup.sh) for details.


Kubernetes integration
----------------------

For example in Kubernetes, see the [example folder](https://github.com/edoburu/docker-pgbouncer/tree/master/examples/kubernetes).


PostgreSQL configuration
------------------------

Make sure PostgreSQL at least accepts connections from the machine where PgBouncer runs! Update `listen_addresses` in `postgresql.conf` and accept incoming connections from your IP range (e.g. `10.0.0.0/8`) in `pg_hba.conf`:

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             all             10.0.0.0/8              md5
```


Using a custom configuration
----------------------------

When the default `pgbouncer.ini` is not sufficient, or you'd like to let multiple users connect through a single PgBouncer instance, extend the `Dockerfile`:

```Dockerfile
FROM edoburu/pgbouncer:1.8.1

COPY pgbouncer.ini /etc/pgbouncer/pgbouncer.ini
COPY userlist.txt /etc/pgbouncer/userlist.txt
```

When the `pgbouncer.ini` file exists, the startup script will not override it. An extra entry will be written to `userlist.txt` when `DATABASE_URL` contains credentials, or `DB_USER` and `DB_PASSWORD` are defined.

The `userlist.txt` file uses the following format:

```
"username" "plaintext-password"
```

or:

```
"username" "md5<md5 of password + username>"
```