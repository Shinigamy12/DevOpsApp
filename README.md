**commande docker database:** docker run -d -v "./DataBase/data:/var/lib/postgresql/data" --network app-network --name database Bastien/database
**commande docker adminer** docker run -p "8090:8080" --net=app-network --name=adminer -d adminer



## Question
### 1-1 Why use `-e` at runtime instead of putting environment variables in the Dockerfile?

It is better to pass environment variables with `-e` when running the container because:

- **Security:** sensitive values (passwords, API keys, tokens) are not hardcoded in the Dockerfile. Anything written in the Dockerfile with `ENV` is stored in the image layers and can be read by anyone who has the image (`docker history`, `docker inspect`), or by anyone who can see the Dockerfile in the Git repository.
- **Flexibility / reusability:** the same image can be used in different environments (dev, test, prod) with different values (database name, user, password) without rebuilding the image.
- **Separation of configuration and code:** the image contains the application, and the configuration is given at runtime. This follows the "12-factor app" principle.

### 1-2 Why do we need a volume to be attached to our postgres container?

A container's filesystem is temporary: when the container is removed (`docker rm`), everything written inside it is lost, including the database files stored in `/var/lib/postgresql/data`.

Attaching a volume (here a bind mount to `DataBase/data` on the host) stores these files outside the container, so:

- **Persistence:** the data survives when the container is stopped, removed, or recreated (for example after rebuilding the image).
- **Decoupling data from the container:** the container can be replaced at any time without losing the database.
- **Initialization only once:** Postgres runs the scripts in `/docker-entrypoint-initdb.d/` only if the data directory is empty. With a volume, the database is initialized the first time and then reused on every restart.

### 1-3 Document your database container essentials: commands and Dockerfile.

**Dockerfile** (`DataBase/Dockerfile`):

```dockerfile
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY CreateScheme.sql /docker-entrypoint-initdb.d/01-CreateScheme.sql
COPY InsertData.sql /docker-entrypoint-initdb.d/02-InsertData.sql
```

- `FROM postgres:17.2-alpine`: official Postgres image, lightweight Alpine version.
- `ENV`: database name, user and password used when the database is created (they could also be passed at runtime with `-e`, see 1-1).
- `COPY ... /docker-entrypoint-initdb.d/`: SQL scripts run automatically, in alphabetical order, the first time the database is initialized. The `01-` / `02-` prefixes make sure the tables are created before the data is inserted.

**Commands:**

```bash
# Create a network so the containers can reach each other by name
docker network create app-network

# Build the image (from the DataBase/ folder)
docker build -t Bastien/database .

# Run the database with a volume to persist the data (run from the project root, the path must start with ./)
docker run -d -v "./DataBase/data:/var/lib/postgresql/data" --network app-network --name database Bastien/database

# Run Adminer to manage the database from the browser
docker run -d -p "8090:8080" --net=app-network --name=adminer adminer
```

**Adminer connection** (http://localhost:8090): System `PostgreSQL`, Server `database`, Username `usr`, Password `pwd`, Database `db`.
