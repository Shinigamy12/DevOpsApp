## How to run the project

| Container | Folder | Role |
|-----------|--------|------|
| `database` | `DataBase/` | Postgres database |
| `simpleapi` | `Backend/` | Spring Boot API |
| `front` | `Frontend/` | Vue.js front (nginx) |
| `httpd` | `HttpServer/` | Apache reverse proxy, the only entry point |

```
Browser --:80--> httpd --/-----> front
                       --/api/--> simpleapi:8080 --> database:5432
```

### Run locally

Only Docker is needed.

```bash
cp .env.example .env    # fill in POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
docker compose up -d --build
```

- **http://localhost/**: the front.
- **http://localhost/api/departments**: the API.
- Problem? `docker compose ps` and `docker compose logs <service>`.

> The front calls the API URL set in `Frontend/.env.production` (the production server). Change it to `localhost/api` to use the local API.

### Deploy to the server

Needs Ansible (in WSL on Windows), the server SSH key in `~/.ssh/id_rsa_takima` and the vault password.

```bash
cd ansible
ansible-playbook -i inventories/setup.yml playbook.yml --ask-vault-pass
```

On every push to `main`, GitHub Actions tests the API, pushes the 4 images to Docker Hub and runs this playbook. Required repository secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `SSH_PRIVATE_KEY`, `VAULT_PASSWORD`.

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
| `docker compose logs simpleapi` | Shows the logs of one service only. |
| `docker compose build` | Rebuilds the images without starting them. |
| `docker compose restart simpleapi` | Restarts one service. |
| `docker compose stop` / `start` | Stops / starts the stack without removing anything. |
| `docker compose exec database sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"'` | Runs a command inside a running service (here: opens a Postgres shell). |
| `docker compose down` | Stops and removes the containers and the network. |
| `docker compose down -v` | Same, **but also deletes the volumes, so all the data**. |
| `docker compose config` | Checks `docker-compose.yml` and prints the final config. |

### 1-8 Document your docker-compose file.

```yaml
services:
  simpleapi:
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

  front:
    build: ./Frontend
    networks:
      - app-network

  httpd:
    build: ./HttpServer
    ports:
      - "80:80"
    networks:
      - app-network
    depends_on:
      - simpleapi
      - front

networks:
  app-network:
  db-network:

volumes:
  db-data:
```

The file declares **four services**. Each one can reach the others by its service name.

**`simpleapi`: the Spring Boot API**

| Option | Why |
|--------|-----|
| `build: ./Backend` | Builds the image from the multistage `Dockerfile` (see 1-4). |
| `env_file: .env` | Database name, user and password, read in `application.yml`. |
| `networks` | `app-network` to be reached by the proxy, `db-network` to reach the database. |
| `depends_on: database` | Starts the database first. |
| `restart: unless-stopped` | Restarts the API if it crashes (e.g. started before Postgres was ready). |

The service **must** be called `simpleapi`: it's the host name used in `httpd.conf`.

**`database`: Postgres**

| Option | Why |
|--------|-----|
| `build: ./DataBase` | Image that runs the SQL scripts on first start (see 1-3). |
| `env_file: .env` | Creates the database with the credentials from `.env`. |
| `volumes` | `db-data` keeps the data when the container is removed (see 1-2). |
| `networks: db-network` | Only reachable by the API, as `database:5432`. |

**`front`: the Vue.js front**

| Option | Why |
|--------|-----|
| `build: ./Frontend` | Builds the app with Node, then serves it with nginx. |
| `networks: app-network` | Only reachable through the proxy. |

**`httpd`: the Apache reverse proxy**

| Option | Why |
|--------|-----|
| `build: ./HttpServer` | Image with our `httpd.conf`: `/api/` → `simpleapi`, `/` → `front` (see 1-5). |
| `ports: "80:80"` | The **only** port published on the host. |
| `depends_on` | Starts the API and the front first. |

**Networks and volumes**

- `app-network`: proxy, front and API. `db-network`: API and database. **Only the API can talk to the database.**
- `db-data`: named volume holding the database data.

> ⚠️ `depends_on` only waits for the container to **start**, not for the app inside to be **ready**, hence `restart: unless-stopped` on the API.

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

### 2-3 Why did we put `needs: build-and-test-backend` on this job?

In our workflow the test job is called `test-backend`, so the line is `needs: test-backend`.

By default, **all the jobs of a workflow start at the same time**, each on its own machine. Without `needs`, the Docker images would be built in parallel with the tests, and nothing would stop them if the tests fail.

`needs` creates a dependency between jobs:

- **Order:** `build-and-push-docker-image` waits until `test-backend` is finished before starting.
- **Condition:** if `test-backend` fails (code doesn't compile, a test is red), the image job is **skipped**. No image is built from broken code.
- **Safety for the next step:** once we push the images to Docker Hub, this is what prevents publishing a broken version as `latest`, which anyone (or a deployment) could pull.

What happens without it:

| | With `needs` | Without `needs` |
|--|--------------|-----------------|
| Start | Tests first, then images | Both jobs at the same time |
| A test fails | Image job skipped | Images built (and later pushed) anyway |
| In the Actions tab | Two jobs linked by an arrow | Two separate, unrelated jobs |

It also saves time and CI minutes: no need to build three images when we already know the code is broken.

### 2-4 For what purpose do we need to push docker images?

Building an image in the CI is not enough: the image only exists on the GitHub runner, and **the runner is destroyed at the end of the job**. Pushing it to a registry (here Docker Hub) **stores it somewhere everyone can get it**.

- **Deployment:** a server (or the next part of the pipeline, Ansible for example) can simply `docker pull` the image and run it. No need to clone the code, install Java or Maven, or build anything on the server.
- **Build once, run anywhere:** the exact same image, tested by the CI, runs on a teammate's laptop, on staging and in production. No more "it works on my machine".
- **Only validated code is published:** thanks to `needs: test-backend` and `push: ${{ github.ref == 'refs/heads/main' }}`, an image is only pushed when the tests pass on `main`. The `latest` tag on Docker Hub is therefore always a working version.
- **Sharing:** anyone can start the whole app with `docker pull` (or a `docker-compose.yml` that uses `image:` instead of `build:`) without the source code.
- **History and rollback:** the registry keeps the images. With version tags (e.g. `1.0`, `1.1` or the commit SHA, not only `latest`), we can go back to a previous version in seconds if a new one is broken.

In short: **the CI builds and tests, the registry stores, the servers pull and run.**

### 3-1 Document your inventory and base commands

Ansible runs from our machine (in WSL, since Ansible doesn't run natively on Windows) and connects to the server over SSH. Nothing has to be installed on the server except Python, which Debian already has.

**Inventory** (`ansible/inventories/setup.yml`):

```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: ~/.ssh/id_rsa_takima
    ansible_python_interpreter: /usr/bin/python3
  children:
    prod:
      hosts:
        bastien.conrad.takima.school:
```

- **`all`**: the root group, it contains every host.
- **`vars`**: variables applied to all the hosts below:
  - `ansible_user`: the user Ansible logs in with over SSH (`admin` on our Debian server).
  - `ansible_ssh_private_key_file`: the private key used to connect. It stays in `~/.ssh` on the WSL side with `chmod 600`, **never in the repository**.
  - `ansible_python_interpreter`: the Python to use on the server. Setting it removes the "discovered Python interpreter" warning.
- **`children`**: sub-groups. Here `prod` contains our server, so we can target `all` or only `prod`.
- **`hosts`**: the servers of the group, by domain name or IP.

**Base commands** (run from the `ansible/` folder):

| Command | What it does |
|---------|--------------|
| `ansible all -i inventories/setup.yml -m ping` | Checks that Ansible can connect to every host and run Python (answers `pong`). |
| `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` | Gathers *facts* about the server (here: the OS and its version). |
| `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=present" --become` | Installs Apache. `--become` runs the task with `sudo`, needed to install packages. |
| `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become` | Removes Apache. |
| `ansible all -i inventories/setup.yml -m shell -a "uptime"` | Runs any shell command on the hosts. |

### 3-2 Document your playbook

The playbook (`ansible/playbook.yml`) installs Docker on the server, following the [Docker documentation for Debian](https://docs.docker.com/engine/install/debian/#install-using-the-repository).

```bash
ansible-playbook -i inventories/setup.yml playbook.yml
```

**Header:**

- `hosts: all`: runs on every host of the inventory.
- `gather_facts: true`: collects info about the server first. We need it for `ansible_facts['distribution_release']` (the Debian version name, e.g. `bookworm`).
- `become: true`: every task runs with `sudo`, required to install packages.

**Tasks:**

| Task | Module | Role |
|------|--------|------|
| Install required packages | `apt` | Installs the tools needed to add an external repository (`curl`, `ca-certificates`, `gnupg`...). `update_cache` runs `apt update` first. |
| Add Docker GPG key | `apt_key` | Adds Docker's signing key so apt trusts its packages. |
| Add Docker APT repository | `apt_repository` | Adds Docker's official repository, for the server's Debian version. |
| Install Docker | `apt` | Installs `docker-ce`. |
| Install Python3 and pip3 | `apt` | Makes sure Python and pip are present. |
| Create a virtual environment | `command` | Creates `/opt/docker_venv`. `creates:` skips the task if the folder already exists. |
| Install Docker SDK for Python | `command` | Installs the `docker` Python library, needed later by Ansible's Docker modules (`docker_container`...). |
| Make sure Docker is running | `service` | Starts the Docker service. |

The order matters: the key and the repository must be added before installing `docker-ce`, because the package isn't in Debian's default repositories.

### 3-3 Document your docker_container tasks configuration.

Each part of the app has its own role, run in this order: `docker` → `create_network` → `launch_database` → `launch_app` → `launch_proxy`.

| Option | Role |
|--------|------|
| `name` | Container name, also its hostname on the network: `database` (used in the backend's JDBC URL) and `simpleapi` (used in `httpd.conf`). |
| `image` | Image pulled from Docker Hub: `"{{ dockerhub_user }}/tp-devops-...:latest"`. |
| `pull: true` | Always pulls the latest image, so a new push is deployed on the next run. |
| `restart_policy: unless-stopped` | Restarts the container if it crashes or if the server reboots. |
| `env` | Database credentials, read from `group_vars/all/vault.yml` (encrypted with Ansible Vault). |
| `volumes` | `db-data` keeps the database data when the container is recreated. |
| `networks` | `db-network`: database + API. `app-network`: API + proxy. The proxy can't reach the database. |
| `ports: "80:80"` | Only on the proxy: the single entry point from outside. |

Run with: `ansible-playbook -i inventories/setup.yml playbook.yml --ask-vault-pass`

### Is it really safe to deploy automatically every new image on the hub? What can I do to make it more secure?

**Not completely.** With continuous deployment, every image pushed on `main` goes straight to production with no human check. The tests protect us against obvious bugs, but:

- **Tests don't catch everything:** a bug that no test covers, a performance problem or a broken database migration still reaches production.
- **A bad merge is enough:** anyone who can push on `main` (or a compromised GitHub account) can deploy any code to the server.
- **Secrets in the CI:** the workflow holds the SSH key and the vault password. If they leak, an attacker gets full access to the server.
- **`latest` is a moving tag:** we don't know exactly which version runs, and the image on Docker Hub could be replaced without the code changing.
- **No rollback:** if the new version is broken, the app stays down until someone fixes it.

**How to make it safer:**

| Measure | Why |
|---------|-----|
| Protect `main` (pull requests + required review + required checks) | Nothing is deployed without being reviewed and tested. |
| Deploy only on version tags (`v1.2.3`) instead of every push on `main` | Deploying becomes a deliberate decision. |
| Use fixed image tags (version or commit SHA) instead of `latest` | We know exactly what runs, and rolling back = redeploying the previous tag. |
| GitHub **Environments** with required reviewers | A manual "approve" button before the `deploy` job runs. |
| Deploy to a **staging** server first | Test the real deployment before production. |
| Scan the images (Trivy, Docker Scout) and the dependencies (OWASP Dependency-Check) | Block images with known vulnerabilities. |
| Limit the secrets | A dedicated SSH key and user for the CI, secrets scoped to the deployment environment, Docker Hub token with minimal rights. |
| Health check after deployment | Check the app answers (e.g. `curl`), and redeploy the previous version if not. |

In short: automatic deployment is safe **only if the pipeline is trusted**. We keep the automation, but add controls before it (reviews, tests, scans, approval) and a way back after it (fixed tags, rollback).
