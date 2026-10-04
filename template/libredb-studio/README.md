# Deploy and Host LibreDB Studio on Sealos

LibreDB Studio is an open-source database IDE for querying and managing PostgreSQL, MySQL, SQLite, MongoDB, and other supported data sources from a browser. This template deploys LibreDB Studio 0.18.0 with HTTPS access, persistent server-side storage, health probes, and deployment-defined administrator credentials on Sealos Cloud.

![LibreDB Studio Screenshot](https://raw.githubusercontent.com/labring-actions/templates/kb-0.9/template/libredb-studio/website-screenshot.webp)

## About Hosting LibreDB Studio

LibreDB Studio provides a web editor, schema explorer, result viewer, connection manager, monitoring pages, and administrative tools in one application. The default deployment stores saved application state in SQLite on a 1 GiB persistent volume. A deployment option provisions a dedicated KubeBlocks-managed PostgreSQL database for shared server-side state.

The PostgreSQL option stores LibreDB Studio settings such as saved connections and queries. Databases that you want to inspect remain separate data sources and can be added from the LibreDB Studio interface after login. A second option connects LibreDB Studio to an existing PostgreSQL, MySQL, MongoDB, or Redis database from the Sealos Database app in the same workspace, so that database is already in the connection list at first login.

## Common Use Cases

- **Browser-based database work**: Run queries and inspect schemas from any browser with a lightweight client footprint.
- **Shared administration workspace**: Keep connection profiles and saved queries on persistent server storage.
- **Development and debugging**: Explore application databases and review query results from one interface.
- **Self-hosted data tooling**: Operate the IDE inside your own Sealos workspace with a generated HTTPS endpoint.
- **Sealos database console**: Open a database created in the Sealos Database app without copying its password into a form.

## Dependencies for LibreDB Studio Hosting

The template includes the runtime components required for a working deployment:

- **LibreDB Studio**: `ghcr.io/libredb/libredb-studio:0.18.0`
- **Default SQLite storage**: `/app/data/libredb-storage.db` on a 1 GiB application PVC
- **Optional PostgreSQL storage**: KubeBlocks-managed `postgresql-16.4.0` with a 1 GiB data PVC
- **HTTPS entrypoint**: Sealos Service, Ingress, and App resources on port `3000`
- **Optional existing database connection**: a ConfigMap with one LibreDB Studio seed connection, filled at runtime from the database's KubeBlocks Secret

### Deployment Dependencies

- [LibreDB Studio Website](https://libredb.org) - Product information
- [LibreDB Studio GitHub Repository](https://github.com/LibreDB/libredb-studio) - Source code, releases, and issue tracking
- [Sealos Cloud](https://cloud.sealos.io) - Cloud workspace and App Launchpad
- [Sealos Documentation](https://sealos.io/docs) - Deployment and operations guides

### Implementation Details

**Architecture components:**

- **LibreDB Studio StatefulSet**: Runs one application replica; SQLite mode mounts persistent storage at `/app/data`.
- **SQLite mode**: Persists server-side application state in the application PVC and is selected by default.
- **PostgreSQL mode**: Creates a PostgreSQL cluster, database access resources, a `libredb_storage` initialization Job, and a database readiness init container.
- **Existing database connection**: Creates a ConfigMap that holds one seed connection with `${ENV}` references only, mounts it read-only at `/app/config/seed-connections.yaml`, sets `SEED_CONFIG_PATH`, and reads host, user, and password from the database's Secret through `secretKeyRef`.
- **Service + Ingress + App**: Routes the generated HTTPS domain to the application and opens the `/login` page.
- **Health checks**: Use the upstream `/api/db/health` endpoint for startup, readiness, and liveness probes.

**Resource baseline:**

Live Sealos testing established `100m` CPU and `128Mi` memory as the lowest stable application tier. The PostgreSQL initialization containers and Job use the same tier. The optional PostgreSQL cluster uses `500m` CPU and `512Mi` memory. Both storage modes passed cold start, administrator login, UI navigation, server storage write/read, query execution, and restart persistence checks.

**Security defaults:**

The application runs as a non-root user, drops Linux capabilities, disables service account token mounting, and receives a generated JWT signing secret. Administrator credentials come from required deployment inputs.

The existing database connection keeps the database password in its Kubernetes Secret, and the ConfigMap holds only variable names. The connection is managed: the password never reaches the browser and the connection cannot be edited in the UI. Only the administrator role can see it, because the Secret carries the database's root or superuser account.

## Why Deploy LibreDB Studio on Sealos?

Sealos combines Kubernetes deployment, networking, storage, and visual operations in one cloud workspace:

- **One-click topology**: Launch the application and its selected storage backend from one template.
- **Persistent state**: Use a PVC-backed SQLite database or a managed PostgreSQL cluster.
- **Immediate HTTPS access**: Receive a generated public domain after deployment.
- **App Launchpad and Canvas operations**: Inspect workloads, logs, storage, and networking from resource cards.
- **AI-assisted changes**: Describe resource or configuration updates in the Canvas dialog.
- **Pay-as-you-go resources**: Start from the tested minimum and scale with actual workload demand.

## Deployment Guide

1. Open the [LibreDB Studio template](https://sealos.io/products/app-store/libredb-studio) and click **Deploy Now**.
2. Enter an administrator email and a password of at least 8 characters.
3. Keep **Use PostgreSQL storage** disabled for the default SQLite deployment, or enable it to provision a dedicated PostgreSQL backend.
4. To open an existing Sealos database, choose its engine in **existing_database_type** and enter its name in **existing_database_name**. Keep `none` to add connections by hand later.
5. Wait for the deployment to complete, typically 2-3 minutes. PostgreSQL mode becomes ready after the database cluster and initialization Job complete.
6. Open the application from its Canvas card. The generated URL leads to `/login`.
7. Sign in with the administrator email and password entered during deployment.

## First Login

LibreDB Studio uses a sign-in-only account model for this deployment. The template provisions the administrator from the deployment form.

1. Open the generated application URL.
2. Enter the configured administrator email and password on `/login`.
3. Click **Sign In** to open the administration overview.
4. Choose **Editor** to run queries. When an existing Sealos database was selected, it is listed first as `<name> (Sealos)` and opens by default. Otherwise, add database connections from the Editor.

Store the deployment credentials in your password manager. Credential changes can be applied from the LibreDB Studio StatefulSet environment variables in Canvas, followed by a workload restart.

## Configuration

| Input | Description | Status | Default |
| --- | --- | --- | --- |
| `admin_email` | Administrator email used on the login page | Required | Set during deployment |
| `admin_password` | Administrator password, minimum 8 characters | Required | Set during deployment |
| `enable_postgres_storage` | Provisions PostgreSQL for LibreDB Studio server-side state | Optional | `false` (SQLite) |
| `existing_database_type` | Engine of an existing Sealos database to connect: `none`, `postgresql`, `mysql`, `mongodb`, or `redis` | Optional | `none` |
| `existing_database_name` | Name of that database as the Sealos Database app shows it | Required when a type is selected | Set during deployment |

The template generates the application name, public host, and JWT signing secret. SQLite data is stored at `/app/data/libredb-storage.db`. PostgreSQL mode creates the `libredb_storage` database and builds the connection URL from the KubeBlocks credential Secret.

The existing database connection reads these Secrets and addresses in the same namespace, and opens the listed database first:

| Engine | Secret | Host | Port | Database |
| --- | --- | --- | --- | --- |
| PostgreSQL | `<name>-conn-credential` | `host` key of the Secret | `5432` | `postgres` |
| MySQL | `<name>-conn-credential` | `host` key of the Secret | `3306` | `mysql` |
| MongoDB | `<name>-mongodb-account-root` | `<name>-mongodb` | `27017` | Not set, authenticates against `admin` |
| Redis | `<name>-redis-account-default` | `<name>-redis-redis` | `6379` | Not set |

With `none`, the template creates no ConfigMap and sets no `SEED_CONFIG_PATH`. LibreDB Studio then lists only its built-in sample connections, and you add your own from the Editor.

## Scaling

The template keeps one LibreDB Studio replica and supports vertical scaling from Canvas. Increase application CPU or memory for more concurrent sessions or heavier result sets. Expand the application PVC for SQLite mode, or the PostgreSQL data PVC for PostgreSQL mode, as stored state grows.

A multi-replica design requires PostgreSQL server storage plus a review of session handling and application behavior. The published template uses the verified single-replica topology.

## Troubleshooting

### The login credentials are rejected

Confirm that the email and password match the values entered during deployment. Review `ADMIN_EMAIL` and `ADMIN_PASSWORD` on the StatefulSet resource card, apply the intended values, and restart the workload.

### The application page is still starting

Open Canvas and check the LibreDB Studio StatefulSet. In PostgreSQL mode, also confirm that the PostgreSQL cluster is `Running` and the `-pg-init` Job is `Complete`.

### Saved settings are missing after a restart

In SQLite mode, confirm that the application PVC is bound and mounted at `/app/data`. In PostgreSQL mode, confirm that the cluster is healthy and the `STORAGE_PROVIDER` value is `postgres`.

### The application stays in CreateContainerConfigError after selecting an existing database

The Secret named in the table above does not exist in this namespace. Check the database name and engine in the Sealos Database app. MongoDB and Redis databases created by older Sealos versions keep their credentials in `<name>-conn-credential` instead, and are not supported by this option. Redeploy with the correct values, or with `none` and add the connection by hand.

### The selected database is missing from the connection list

Sign in with the administrator account: the connection is visible to the administrator role only. If it is still missing, check the LibreDB Studio logs in Canvas for a line about the seed configuration.

### A database connection fails

Use a database hostname that is reachable from the Sealos workspace, verify the port and credentials, and select the TLS settings required by the target database. After you change the password of the database selected at deployment, restart the LibreDB Studio workload from Canvas: it reads the password from the Secret only when it starts.

### Getting Help

- [LibreDB Studio GitHub Issues](https://github.com/LibreDB/libredb-studio/issues)
- [LibreDB Studio Releases](https://github.com/LibreDB/libredb-studio/releases)
- [Sealos Discord](https://discord.gg/wdUn538zVP)

## Additional Resources

- [LibreDB Studio Website](https://libredb.org)
- [LibreDB Studio Source Code](https://github.com/LibreDB/libredb-studio)
- [Sealos Cloud](https://cloud.sealos.io)
- [Sealos Documentation](https://sealos.io/docs)

## License

LibreDB Studio is licensed under the MIT License. This Sealos template follows the license of the templates repository.
