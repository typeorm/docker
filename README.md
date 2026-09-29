# PostgreSQL with PostGIS and pgvector for TypeORM

This Docker image extends the official PostgreSQL image with PostGIS and pgvector extensions, specifically designed for TypeORM development and testing environments. It provides a configurable PostgreSQL instance with spatial and vector search capabilities.

## Use Cases

- **TypeORM Testing**: Drop-in replacement for PostgreSQL in TypeORM test suites.
- **Development**: Local development with TypeORM projects requiring PostGIS or pgvector.
- **CI/CD**: GitHub Actions and other CI environments for TypeORM projects.

## Pre-built Images

Pre-built images are available on GitHub Container Registry (GHCR). Replace `yourusername/yourrepositoryname` with your actual GHCR path (e.g., `ghcr.io/naorpeled/typeorm-postgres-docker`).

Example tags (refer to `versions.json` and the `publish.yml` workflow matrix for all combinations):

- `ghcr.io/yourusername/yourrepositoryname:postgres-18.6-postgis-3.6.4-pgvector-0.8.6`
- `ghcr.io/yourusername/yourrepositoryname:latest` (points to the default latest combination)

## Build Arguments

The following build arguments can be used with `docker build --build-arg VAR=value` or via the `args` section in `docker-compose.yml` (which can use environment variables from your `.env` file):

- `PG_VERSION`: PostgreSQL image tag (default: 18.6). In `docker-compose.yml`, fed by the `PG_VERSION` env var.
- `POSTGIS_VERSION`: PostGIS package version (default: 3.6.4). In `docker-compose.yml`, fed by the `POSTGIS_VERSION` env var.
- `PGVECTOR_VERSION`: pgvector release version (e.g., `0.8.6`, default: `0.8.6`). In `docker-compose.yml`, fed by the `PGVECTOR_VERSION` env var.

## Building the Image

```bash
# Build with default versions
docker build -t your-image-name .

# Build with custom versions (using Docker build args)
docker build \\
  --build-arg PG_VERSION=17.9 \\
  --build-arg POSTGIS_VERSION=3.6.2 \\
  --build-arg PGVECTOR_VERSION=0.8.2 \\
  -t your-image-name:custom .
```

## Running the Container

### Using `docker run`

To ensure pgvector is properly preloaded for optimal performance, pass the `shared_preload_libraries` setting to the `postgres` command. This is handled by the `command` directive in the provided `docker-compose.yml`.

```bash
docker run -d \\
  --name postgres-gis-vector \\
  -e POSTGRES_PASSWORD=yourpassword \\
  -e POSTGRES_USER=youruser \\
  -e POSTGRES_DB=yourdb \\
  -e PGDATA=/var/lib/postgresql/pgdata \\
  -p 5432:5432 \\
  -v postgres_data_volume:/var/lib/postgresql \\
  your-image-name postgres -c shared_preload_libraries=vector
```

Mount the parent directory at `/var/lib/postgresql` and set `PGDATA` to a subdirectory inside that mount (for example `/var/lib/postgresql/pgdata`). This keeps persistence working for both PostgreSQL 18+ and PostgreSQL 17 and earlier, whose upstream images use different default data directories and volume layouts.

### Using `docker-compose.yml`

The provided `docker-compose.yml` is configured to build and run the image, including preloading pgvector via the `command` directive.

```yaml
# docker-compose.yml (snippet)
services:
  db:
    build:
      context: .
      args:
        PG_VERSION: ${PG_VERSION:-18.6}
        POSTGIS_VERSION: ${POSTGIS_VERSION:-3.6.4}
        PGVECTOR_VERSION: ${PGVECTOR_VERSION:-0.8.6}
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-test} # TypeORM test default
      POSTGRES_USER: ${POSTGRES_USER:-test} # TypeORM test default
      POSTGRES_DB: ${POSTGRES_DB:-test} # TypeORM test default
      PGDATA: /var/lib/postgresql/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql
    command: postgres -c shared_preload_libraries=vector # Ensures pgvector preloading
      # ... other environment variables ...
```

Create a `.env` file in the same directory as `docker-compose.yml` to set build arguments and PostgreSQL credentials (optional, defaults are provided):

```env
# .env (example)
PG_VERSION=18.6
POSTGIS_VERSION=3.6.4
PGVECTOR_VERSION=0.8.6

POSTGRES_PASSWORD=supersecret
POSTGRES_USER=myuser
POSTGRES_DB=mydb
POSTGRES_PORT=5432
# ADDITIONAL_DATABASES=db1,db2
```

Then run:

```bash
docker compose up --build -d
```

If you already have a PostgreSQL 17 data volume mounted directly at `/var/lib/postgresql/data`, treat the move to the compose defaults above as a migration rather than a drop-in path change. Back up and restore the database or follow the official major-upgrade process before reusing existing data with `PGDATA=/var/lib/postgresql/pgdata`.

## GitHub Actions

This repository includes GitHub Actions workflows:

- `.github/workflows/test.yml`: Builds the Docker image with a matrix of PostgreSQL, PostGIS, and pgvector versions from `versions.json`, and runs basic extension checks.
- `.github/workflows/publish.yml`: Builds and publishes the Docker image to GHCR on tagged releases (e.g., `v1.0.0`). The image name on GHCR will be based on your GitHub username/organization and repository name (e.g., `ghcr.io/yourusername/yourrepositoryname`).

## TypeORM Compatibility

This image is configured to be a drop-in replacement for PostgreSQL in TypeORM projects.

- Default credentials in `docker-compose.yml` (`test`/`test`/`test`) match common TypeORM test configurations.
- Ensure your TypeORM datasource configuration matches the environment variables used (e.g., `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `POSTGRES_PORT`).
- The image ensures `pgvector` is preloaded via `shared_preload_libraries=vector` when run with the provided `docker-compose.yml` or an equivalent `docker run` command, which is crucial for some pgvector features and performance.

This setup ensures that TypeORM can connect and utilize PostGIS and pgvector functionalities seamlessly.

## Environment Variables

Inherits all environment variables from the official PostgreSQL image. See the [official PostgreSQL image documentation](https://hub.docker.com/_/postgres/) for details.

The image build is parameterized by the `PG_VERSION`, `POSTGIS_VERSION`, and `PGVECTOR_VERSION` build arguments. These values are used only at build time unless you also set them explicitly as runtime environment variables.

Additional runtime environment variables for the entrypoint script:

- `ADDITIONAL_DATABASES`: Comma-separated list of additional databases to create and initialize with extensions.

### Local Testing with `docker-compose.test.yml`

The repository includes `docker-compose.test.yml` for more comprehensive local testing that mirrors some aspects of the CI tests.

To run tests locally using this file:

```bash
# Build and test with default versions using environment variables from your .env or defaults
docker compose -f docker-compose.test.yml up --build --exit-code-from test

# Test with specific versions by setting environment variables for the compose command:
PG_VERSION=17.9 PG_MAJOR=17 POSTGIS_VERSION=3.6.2 PGVECTOR_VERSION=0.8.2 docker compose -f docker-compose.test.yml up --build --exit-code-from test
```

## License

MIT
