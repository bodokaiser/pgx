# Contributing

## Development Environment Setup

pgx tests naturally require a PostgreSQL database. It will connect to the database specified in the `PGX_TEST_DATABASE`
environment variable. The `PGX_TEST_DATABASE` environment variable can either be a URL or key-value pairs. In addition,
the standard `PG*` environment variables will be respected. Consider using [direnv](https://github.com/direnv/direnv) to
simplify environment variable handling.

### Using an Existing PostgreSQL Cluster

If you already have a PostgreSQL development server this is the quickest way to start and run the majority of the pgx
test suite. Some tests will be skipped that require server configuration changes (e.g. those testing different
authentication methods).

Create and setup a test database:

```
export PGDATABASE=pgx_test
createdb
psql -c 'create extension hstore;'
psql -c 'create domain uint64 as numeric(20,0);'
```

Ensure a `postgres` user exists. This happens by default in normal PostgreSQL installs, but some installation methods
such as Homebrew do not.

```
createuser -s postgres
```

Ensure your `PGX_TEST_DATABASE` environment variable points to the database you just created and run the tests.

```
export PGX_TEST_DATABASE="host=/private/tmp database=pgx_test"
go test ./...
```

This will run the vast majority of the tests, but some tests will be skipped (e.g. those testing different connection methods).

### Creating a New PostgreSQL Cluster Exclusively for Testing

The following environment variables need to be set both for initial setup and whenever the tests are run. (direnv is
highly recommended). Depending on your platform, you may need to change the host for `PGX_TEST_UNIX_SOCKET_CONN_STRING`.

```
export PGPORT=5015
export PGUSER=postgres
export PGDATABASE=pgx_test
export POSTGRESQL_DATA_DIR=postgresql

export PGX_TEST_DATABASE="host=127.0.0.1 database=pgx_test user=pgx_md5 password=secret"
export PGX_TEST_UNIX_SOCKET_CONN_STRING="host=/private/tmp database=pgx_test"
export PGX_TEST_TCP_CONN_STRING="host=127.0.0.1 database=pgx_test user=pgx_md5 password=secret"
export PGX_TEST_SCRAM_PASSWORD_CONN_STRING="host=127.0.0.1 user=pgx_scram password=secret database=pgx_test"
export PGX_TEST_MD5_PASSWORD_CONN_STRING="host=127.0.0.1 database=pgx_test user=pgx_md5 password=secret"
export PGX_TEST_PLAIN_PASSWORD_CONN_STRING="host=127.0.0.1 user=pgx_pw password=secret"
export PGX_TEST_TLS_CONN_STRING="host=localhost user=pgx_ssl password=secret sslmode=verify-full sslrootcert=`pwd`/.testdb/ca.pem"
export PGX_SSL_PASSWORD=certpw
export PGX_TEST_TLS_CLIENT_CONN_STRING="host=localhost user=pgx_sslcert sslmode=verify-full sslrootcert=`pwd`/.testdb/ca.pem database=pgx_test sslcert=`pwd`/.testdb/pgx_sslcert.crt sslkey=`pwd`/.testdb/pgx_sslcert.key"
```

Create a new database cluster.

```
initdb --locale=en_US -E UTF-8 --username=postgres .testdb/$POSTGRESQL_DATA_DIR

echo "listen_addresses = '127.0.0.1'" >> .testdb/$POSTGRESQL_DATA_DIR/postgresql.conf
echo "port = $PGPORT" >> .testdb/$POSTGRESQL_DATA_DIR/postgresql.conf
cat testsetup/postgresql_ssl.conf >> .testdb/$POSTGRESQL_DATA_DIR/postgresql.conf
cp testsetup/pg_hba.conf .testdb/$POSTGRESQL_DATA_DIR/pg_hba.conf
cp testsetup/ca.cnf .testdb
cp testsetup/localhost.cnf .testdb
cp testsetup/pgx_sslcert.cnf .testdb

cd .testdb

# Generate a CA public / private key pair.
openssl genrsa -out ca.key 4096
openssl req -x509 -config ca.cnf -new -nodes -key ca.key -sha256 -days 365 -subj '/O=pgx-test-root' -out ca.pem

# Generate the certificate for localhost (the server).
openssl genrsa -out localhost.key 2048
openssl req -new -config localhost.cnf -key localhost.key -out localhost.csr
openssl x509 -req -in localhost.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out localhost.crt -days 364 -sha256 -extfile localhost.cnf -extensions v3_req

# Copy certificates to server directory and set permissions.
cp ca.pem $POSTGRESQL_DATA_DIR/root.crt
cp localhost.key $POSTGRESQL_DATA_DIR/server.key
chmod 600 $POSTGRESQL_DATA_DIR/server.key
cp localhost.crt $POSTGRESQL_DATA_DIR/server.crt

# Generate the certificate for client authentication.
openssl genrsa -des3 -out pgx_sslcert.key -passout pass:certpw 2048
openssl req -new -config pgx_sslcert.cnf -key pgx_sslcert.key -passin pass:certpw -out pgx_sslcert.csr
openssl x509 -req -in pgx_sslcert.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out pgx_sslcert.crt -days 363 -sha256 -extfile pgx_sslcert.cnf -extensions v3_req

cd ..
```


Start the new cluster. This will be necessary whenever you are running pgx tests.

```
postgres -D .testdb/$POSTGRESQL_DATA_DIR
```

Setup the test database in the new cluster.

```
createdb
psql --no-psqlrc -f testsetup/postgresql_setup.sql
```

### PgBouncer

There are tests specific for PgBouncer that will be executed if `PGX_TEST_PGBOUNCER_CONN_STRING` is set.

### Optional Tests

pgx supports multiple connection types and means of authentication. These tests are optional. They will only run if the
appropriate environment variables are set. In addition, there may be tests specific to particular PostgreSQL versions,
non-PostgreSQL servers (e.g. CockroachDB), or connection poolers (e.g. PgBouncer). `go test ./... -v | grep SKIP` to see
if any tests are being skipped.
