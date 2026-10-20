## How to run the project

The app runs in 3 containers that talk to each other over a Docker network:

| Container | Folder | Role |
|-----------|--------|------|
| `database` | `DataBase/` | Postgres database |
| `simpleapi` | `Backend/` | Spring Boot API |
| `httpserver` | `HttpServer/` | Apache reverse proxy, the only entry point |

Adminer is optional, it's just handy to look inside the database.

```
Browser --:80--> httpserver (Apache) --> simpleapi:8080 (Spring Boot) --> database:5432 (Postgres)
```

**What you need:** only Docker. Java and Maven are not needed on your machine, the API is compiled inside Docker thanks to its multistage `Dockerfile` (see 1-4).

**Before starting:** copy `.env.example` to `.env` and fill it in. It holds the database name, user and password. It is not committed, so the secrets never end up in Git.

> The fastest way is `docker compose up -d --build` (see 1-6 to 1-8). The steps below do the same thing by hand.

### 1. Create the network (only once)

```bash
docker network create app-network
```

### 2. Start the database

```bash
docker build -t bastien/database ./DataBase
docker run -d -v db-data:/var/lib/postgresql/data --network app-network --env-file .env --name database bastien/database
```

> ⚠️ The container **must** be called `database`. The API connects to `jdbc:postgresql://database:5432/${POSTGRES_DB}` (see `Backend/src/main/resources/application.yml`).

### 3. Start the API

```bash
docker build -t bastien/simpleapi ./Backend
docker run -d --name simpleapi --network app-network --env-file .env bastien/simpleapi
```

> ⚠️ The container **must** be called `simpleapi`, that's the name the reverse proxy looks for.
>
> No `-p` here: the API is only reachable through the proxy. Add `-p 8080:8080` if you want to test it directly.

### 4. Start the reverse proxy

```bash
docker build -t bastien/httpserver ./HttpServer
docker run -d --name httpserver --network app-network -p 80:80 bastien/httpserver
```

### 5. Adminer (optional)

```bash
docker run -d -p "8090:8080" --net=app-network --name=adminer adminer
```

### Is it working?

- **http://localhost/** should answer `{"id":1,"content":"Hello, World!"}` (the API, through the proxy).
- **http://localhost:8090** opens Adminer. Log in with:
  - System: `PostgreSQL`
  - Server: `database`
  - Username / Password / Database: `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` from your `.env`
- Something doesn't answer? Check `docker ps` and `docker logs <container>`.

> The start order matters: **database → API → proxy**.

---

## Questions

### 1-1 Why use `-e` at runtime instead of putting environment variables in the Dockerfile?

In short: **secrets don't belong in an image.**

- **Security.** Anything you write with `ENV` in a Dockerfile is baked into the image layers. Anyone who has the image can read it with `docker history` or `docker inspect`, and anyone who can see the repo can read the Dockerfile. Passwords, API keys and tokens should never be there.
- **One image, many environments.** The same image works in dev, test and prod. You just give it different values (database name, user, password) when you start it, no rebuild needed.
- **Code and config stay separate.** The image is the application, the config comes at runtime. That's one of the "12-factor app" principles.

### 1-2 Why do we need a volume to be attached to our postgres container?

Because **a container forgets everything when it's removed.** If you `docker rm` the Postgres container, all the files in `/var/lib/postgresql/data`, so the whole database, are gone.

A volume stores those files outside the container. Here we use a named volume, `db-data`, managed by Docker (not a folder on the host). That gives us:

- **Persistence:** the data survives a stop, a removal or a rebuild of the container.
- **Replaceable containers:** we can throw the container away and create a new one anytime without losing anything.
- **Initialization only once:** Postgres only runs the scripts in `/docker-entrypoint-initdb.d/` when the data folder is empty. With a volume, the database is set up the first time and simply reused afterwards.

### 1-3 Document your database container essentials: commands and Dockerfile.

**The Dockerfile** (`DataBase/Dockerfile`):

```dockerfile
FROM postgres:17.2-alpine

COPY CreateScheme.sql /docker-entrypoint-initdb.d/01-CreateScheme.sql
COPY InsertData.sql /docker-entrypoint-initdb.d/02-InsertData.sql
```

| Line | What it does |
|------|--------------|
| `FROM postgres:17.2-alpine` | Starts from the official Postgres image, in its lightweight Alpine version. |
| `COPY ... /docker-entrypoint-initdb.d/` | Postgres runs every script in this folder automatically, in alphabetical order, the first time the database is created. The `01-` / `02-` prefixes make sure the tables exist before the data is inserted. |

You'll notice there is **no `ENV`**. The database name, user and password (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`) come from the `.env` file at runtime (`--env-file .env`, or `env_file` in docker-compose). That way they are neither in the image nor in Git (see 1-1).

**The commands:**

```bash
# Create a network so the containers can find each other by name
docker network create app-network

# Build the image (from the repository root)
docker build -t bastien/database ./DataBase

# Run the database:
#  - with the db-data volume to keep the data (Docker creates it if needed)
#  - with the credentials from .env
docker run -d -v db-data:/var/lib/postgresql/data --network app-network --env-file .env --name database bastien/database

# Run Adminer to browse the database from the browser
docker run -d -p "8090:8080" --net=app-network --name=adminer adminer
```

**Connecting with Adminer** (http://localhost:8090): System `PostgreSQL`, Server `database`, and for Username / Password / Database the values of `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` in `.env`.

### 1-4 Why do we need a multistage build? And explain each step of this dockerfile.

To build a Java app you need a JDK and Maven. To run it, you only need a JRE and the `.jar`. A multistage build lets us use the heavy tools in a first stage, then **keep only what's needed to run the app** in the final image.

- **Smaller image:** the JDK and Maven stay behind in the build stage. The final image only has the JRE and the `.jar`.
- **More secure:** no compiler, no Maven, no source code in production means fewer things to attack.
- **Everything in one file:** build and run are described in the same Dockerfile. Nobody needs Java or Maven installed, and the build is the same on every machine.

**The Dockerfile:**

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

**Stage 1: build the app**

| Line | What it does |
|------|--------------|
| `FROM eclipse-temurin:21-jdk-alpine AS myapp-build` | Starts from a JDK 21 image, needed to compile Java. The stage is named `myapp-build` so the next stage can refer to it. |
| `ENV MYAPP_HOME=/opt/myapp` | Stores the app folder in a variable so we don't repeat the path. |
| `WORKDIR $MYAPP_HOME` | Creates that folder and moves into it. |
| `RUN apk add --no-cache maven` | Installs Maven with Alpine's package manager. `--no-cache` avoids keeping the package index in the image. |
| `COPY pom.xml .` | Copies the Maven project file (dependencies and build settings). |
| `COPY src ./src` | Copies the source code. |
| `RUN mvn package -DskipTests` | Compiles the code and packages it into a `.jar` in `target/`. Tests are skipped to build faster. |

**Stage 2: run the app**

| Line | What it does |
|------|--------------|
| `FROM eclipse-temurin:21-jre-alpine` | Starts a brand new, clean image with only the JRE 21 (enough to run Java, no compiler). Nothing from stage 1 is kept unless we copy it. |
| `ENV MYAPP_HOME=/opt/myapp` | Defines the variable again, because `ENV` is not shared between stages. |
| `WORKDIR $MYAPP_HOME` | Creates the app folder and moves into it. |
| `COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar` | Grabs only the built `.jar` from stage 1 and renames it `myapp.jar`. |
| `ENTRYPOINT ["java", "-jar", "myapp.jar"]` | The command run when the container starts: launches the app. |

### 1-5 Why do we need a reverse proxy?

A reverse proxy is a server that **sits in front of the application**. Clients only ever talk to it: it forwards their requests to the backend and sends the answer back. Here, the Apache `httpserver` container listens on port 80 and forwards everything to `simpleapi` on port 8080.

Why it's worth it:

- **Security:** only the proxy is exposed. The API and the database stay hidden inside the Docker network, clients can't reach them and don't even know they exist.
- **One entry point:** all services (API, front-end, ...) are behind the same address and port. The proxy sends each request to the right container based on the URL.
- **HTTPS in one place:** certificates are only set up on the proxy. Behind it, the backend can keep using plain HTTP.
- **Load balancing:** the proxy can spread requests over several backend instances and skip the ones that are down (`balancer://` in Apache).
- **Extras for free:** static files, compression, caching, logging, authentication... all without touching the app code.

**The configuration** (`HttpServer/httpd.conf`, copied into the image by the Dockerfile):

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://simpleapi:8080/
ProxyPassReverse / http://simpleapi:8080/
</VirtualHost>
```

| Directive | What it does |
|-----------|--------------|
| `LoadModule proxy_module` / `proxy_http_module` | Enables Apache's proxy features, and HTTP support for them. |
| `ProxyPass` | Forwards every request received on `/` to the backend. `simpleapi` is the backend container's name, which Docker resolves because both containers are on `app-network`. |
| `ProxyPassReverse` | Rewrites the `Location` headers of the backend's redirects so they point to the proxy, not to the internal backend address. |
| `ProxyPreserveHost On` | Keeps the client's original `Host` header when forwarding the request. |

### 1-6 Why is docker-compose so important?

Without Compose, starting the stack means about **ten commands typed by hand**, in the right order, with the right options: create the network, build three images, run three containers with their names, volumes, ports and network. Forget one `--network` and nothing works.

Compose puts all of that in a single file, `docker-compose.yml`, and runs it with **one command**.

- **One command for everything:** `docker compose up --build` builds the images, creates the network and the volume, and starts the containers in order.
- **Config as code:** the setup lives in a file versioned in Git. It's documented, reviewable and identical for everyone.
- **Reproducible:** any developer (or the CI) gets the exact same environment just by cloning the repo.
- **Start order handled:** `depends_on` starts the database, then the API, then the proxy.
- **Easy to manage:** stop, restart, rebuild, read logs or delete the whole app at once instead of container by container.

### 1-7 Document docker-compose most important commands.

| Command | What it does |
|---------|--------------|
| `docker compose up -d --build` | Builds the images if needed and starts everything in the background. |
| `docker compose ps` | Lists the project's containers and their status. |
| `docker compose logs -f` | Shows the logs of all services and keeps following them. |
| `docker compose logs backend` | Shows the logs of one service only. |
| `docker compose build` | Rebuilds the images without starting them. |
| `docker compose restart backend` | Restarts one service. |
| `docker compose stop` / `start` | Stops / starts the stack without removing anything. |
| `docker compose exec database sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'` | Runs a command inside a running service (here: opens a Postgres shell). |
| `docker compose down` | Stops and removes the containers and the network. |
| `docker compose down -v` | Same, **but also deletes the volumes, so all the data**. |
| `docker compose config` | Checks `docker-compose.yml` and prints the final config. |

### 1-8 Document your docker-compose file.

```yaml
services:
  backend:
    build: ./Backend
    env_file: .env
    networks:
      - app-network
      - db-network
    depends_on:
      - database
    restart: unless-stopped

  database:
    build: ./DataBase
    env_file: .env
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - db-network

  httpd:
    build: ./HttpServer
    ports:
      - "80:80"
    networks:
      - app-network
    depends_on:
      - backend

networks:
  app-network:
  db-network:

volumes:
  db-data:
```

The file declares **three services**. Compose names the containers `devopsapp-<service>-1`, and each service can reach the others just by using their service name.

**`backend`: the Spring Boot API**

| Option | Why |
|--------|-----|
| `build: ./Backend` | Builds the image from the multistage `Dockerfile` in `Backend/` (see 1-4). |
| `env_file: .env` | Gives the API the database name, user and password, used in `application.yml` as `${POSTGRES_DB}`, `${POSTGRES_USER}`, `${POSTGRES_PASSWORD}`. |
| `networks` | On `app-network` so the proxy can reach it, and on `db-network` so it can reach the database. |
| `depends_on: database` | Starts the database first. |
| `restart: unless-stopped` | Restarts the API if it crashes (for example if it started before Postgres was ready), unless you stopped it yourself. |

**`database`: Postgres**

| Option | Why |
|--------|-----|
| `build: ./DataBase` | Builds the image that runs the SQL scripts on first start (see 1-3). |
| `env_file: .env` | Creates the database with the name, user and password from `.env`. |
| `volumes: db-data:/var/lib/postgresql/data` | Keeps the data in a named volume so it survives the container being removed (see 1-2). |
| `networks: db-network` | Only on the database network. The API reaches it as `database:5432`, the host set in `application.yml`. |

**`httpd`: the Apache reverse proxy**

| Option | Why |
|--------|-----|
| `build: ./HttpServer` | Builds the image with our `httpd.conf` (see 1-5). |
| `ports: "80:80"` | The **only** port published on the host. Everything goes through the proxy, the API and the database can't be reached from outside. |
| `depends_on: backend` | Starts the API first. |

**Networks and volumes**

- `networks`: two bridge networks created by Compose. `app-network` links the proxy and the API, `db-network` links the API and the database. Result: **the proxy can't talk to the database directly.**
- `volumes: db-data`: the named volume created by Compose (it shows up as `devopsapp_db-data`).

> ⚠️ `depends_on` only waits for the container to **start**, not for the app inside to be **ready**. Postgres can still be booting when the API starts, which is why the backend has `restart: unless-stopped`.

### 2-1 What are testcontainers?

Testcontainers is a Java library that **starts real Docker containers during the tests** and throws them away at the end.

Instead of testing against a fake, in-memory database (like H2) or a shared database that someone has to install, every test run gets a fresh, real Postgres in a container, the same engine we use in production.

In this project:

- The test config ([Backend/src/test/resources/application.yml](Backend/src/test/resources/application.yml)) uses the URL `jdbc:tc:postgresql://hostname/simple-api`. The `tc:` prefix tells Testcontainers to start a Postgres container automatically when the app connects to the database.
- Each integration test (`*IT`) runs `InsertData.sql` with `@Sql`, so it always starts from the same data.
- At the end of the tests, the container is stopped and removed. Nothing is left behind.

Why it's useful:

- **Realistic tests:** the SQL runs on a real Postgres, so we catch problems an in-memory database would hide.
- **Nothing to install:** you only need Docker. No local database to set up, and the CI runner (which has Docker) works the same way.
- **Isolated and reproducible:** every run starts from a clean database, so tests don't depend on leftovers from a previous run.

> ⚠️ Testcontainers needs a Docker it can talk to. With Docker 29+, Testcontainers must be at least **1.21.4**, otherwise it fails with `Could not find a valid Docker environment`.

### 2-2 For what purpose do we need to use secured variables?

To **use secrets in the CI without writing them in the code.**

To publish our images, the workflow has to log in to Docker Hub, so it needs a username and a password or token. If we wrote them in `main.yml`, they would be in the Git history forever, readable by anyone who can see the repository (anyone at all if it's public). Even deleting them in a later commit doesn't help, the old commits still contain them.

GitHub **secrets** solve that:

- **Stored outside the code:** they are saved in the repository settings (`Settings → Secrets and variables → Actions`), never in a file or a commit.
- **Write-only:** once saved, nobody can read the value again, not even the repository owner. It can only be replaced or deleted.
- **Hidden in the logs:** if a step prints a secret, GitHub replaces it with `***`.
- **Not shared with forks:** a pull request from a fork doesn't get the secrets, so a stranger can't steal them by changing the workflow.
- **Easy to change:** if a token leaks or expires, we update the secret in one place, no code change needed.

In our workflow we read them like this:

```yaml
${{ secrets.DOCKERHUB_USERNAME }}
${{ secrets.DOCKERHUB_TOKEN }}
```

We use a Docker Hub **access token** rather than the account password: it only has the rights it needs (read & write on images), and it can be revoked at any time without touching the account.
