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


docker run -d --name backend Bastien/backend  
### 1-4 Why do we need a multistage build? And explain each step of this dockerfile.

A multistage build lets us use several `FROM` stages in one Dockerfile and keep only what we need in the final image:

- **Smaller image:** the JDK and Maven (needed to compile and package the app) stay in the build stage. The final image only contains the JRE and the `.jar`, so it is much lighter.
- **Security:** fewer tools in the final image (no compiler, no Maven, no source code) means a smaller attack surface.
- **One single Dockerfile:** build and run are described in the same file, no need to have Java or Maven installed on the host. The build is reproducible.

**Dockerfile:**

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME

RUN apk add --no-cache maven

COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM eclipse-temurin:21-jre-alpine
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT ["java", "-jar", "myapp.jar"]
```

**Build stage:**

- `FROM eclipse-temurin:21-jdk-alpine AS myapp-build`: start from a JDK 21 image (needed to compile Java). The stage is named `myapp-build` so the next stage can refer to it.
- `ENV MYAPP_HOME=/opt/myapp`: define a variable with the app folder, to avoid repeating the path.
- `WORKDIR $MYAPP_HOME`: create this folder and move into it for the next commands.
- `RUN apk add --no-cache maven`: install Maven with the Alpine package manager. `--no-cache` avoids keeping the package index in the image.
- `COPY pom.xml .`: copy the Maven project file (dependencies and build config).
- `COPY src ./src`: copy the source code.
- `RUN mvn package -DskipTests`: compile the code and package it into a `.jar` in `target/`. Tests are skipped to make the build faster.

**Run stage:**

- `FROM eclipse-temurin:21-jre-alpine`: start a new clean image with only the JRE 21 (enough to run Java, no compiler). Everything from the build stage is dropped except what we copy.
- `ENV MYAPP_HOME=/opt/myapp`: define the variable again, because `ENV` values are not shared between stages.
- `WORKDIR $MYAPP_HOME`: create and move into the app folder.
- `COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar`: copy only the built `.jar` from the build stage and rename it `myapp.jar`.
- `ENTRYPOINT ["java", "-jar", "myapp.jar"]`: command run when the container starts: launch the application with the JRE.


docker run -d -p 8080:8080 --name simpleapi --network app-network bastien/simpleapi