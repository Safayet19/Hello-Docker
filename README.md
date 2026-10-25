
<h1 align="center">Docker & Docker Compose - Practical Guide</h1>

<p align="center">Essential Docker commands, Dockerfiles, container networking, persistent storage, Compose, and real application examples.</p>

---

This guide uses **complete commands**, short explanations, and runnable examples. It is suitable for Docker Desktop on Windows or Docker Engine on Linux. Most terminal commands use Bash syntax (including Git Bash). Start Docker Desktop or the Docker daemon before running them.

> **Before starting:** Check `docker version` and `docker compose version`. Use test containers for practice. Commands that delete images, containers, or volumes can remove data; read the cleanup warnings before running them.

## Table of Contents

1. [Docker Basics](#docker-basics)
2. [Docker Images](#docker-images)
3. [Container Management](#container-management)
4. [Dockerfile and .dockerignore](#dockerfile-and-dockerignore)
5. [Ports and Networking](#ports-and-networking)
6. [Volumes and Storage](#volumes-and-storage)
7. [Docker Hub](#docker-hub)
8. [Docker Compose](#docker-compose)
9. [Docker Compose Commands](#docker-compose-commands)
10. [Troubleshooting and Cleanup](#troubleshooting-and-cleanup)
11. [Practical Projects](#practical-projects)

---

## Docker Basics

Docker packages an application and its dependencies into an **image**. A **container** is a running or stopped instance of an image. Docker Compose defines one or more container services in a YAML file.

| Term | Meaning | Practical example |
|---|---|---|
| Docker Engine | Builds and runs containers | The daemon used by `docker run` |
| Image | Read-only application template | `nginx:alpine` |
| Container | Instance created from an image | A running Nginx web server |
| Dockerfile | Instructions for building an image | `FROM node:24-alpine` |
| Registry | Place to store and retrieve images | Docker Hub |
| Volume | Docker-managed persistent storage | `redis-lab-data` |
| Network | Allows connected containers to communicate | `lab-net` |
| Compose service | Named container configuration in YAML | `app:` under `services:` |

| Task | Command | Explanation |
|---|---|---|
| Check client and engine | `docker version` | Shows Docker client and server versions; server errors indicate the engine is unavailable. |
| Check Compose | `docker compose version` | Confirms the modern Compose plugin is installed. |
| Inspect Docker status | `docker info` | Shows engine, storage, and runtime information. |
| Verify installation | `docker run --rm hello-world` | Pulls and runs the official test image; `--rm` removes its stopped container. |
| See command help | `docker run --help` | Shows complete options for `docker run`. |

**Image vs container:** `docker build` creates an image. `docker run` creates and starts a new container from an image. `docker start` starts an existing stopped container. `docker compose up` manages containers described as services in `compose.yaml`.

## Docker Images

| Task | Command | Explanation |
|---|---|---|
| Download an image | `docker pull nginx:alpine` | Downloads the Nginx image with the `alpine` tag. |
| List local images | `docker images` | Shows available local images and tags. |
| Build an image | `docker build -t docker-node-demo:1.0 .` | Builds from `Dockerfile` in the current directory (`.` is the build context); `-t` names and tags it. Requires Section 11 project files. |
| Build without cached layers | `docker build --no-cache -t docker-node-demo:1.0 .` | Rebuilds every layer; use only when cache troubleshooting requires it. |
| Add another image tag | `docker tag docker-node-demo:1.0 docker-node-demo:latest` | Gives the same image an additional tag; does not rebuild it. |
| Inspect image details | `docker image inspect nginx:alpine` | Displays JSON metadata about the image. |
| Show image history | `docker history nginx:alpine` | Shows the image's build layers. |
| Remove an unused image | `docker rmi docker-node-demo:latest` | Removes that image tag if it is not required by an existing container. |
| Show disk use | `docker system df` | Summarizes space consumed by images, containers, and volumes. |

**Tags:** `nginx:alpine` and `nginx:latest` are different image references. Omitting a tag normally means `:latest`; `latest` does not guarantee the newest or most stable version.

## Container Management

Use the name `web-demo` for the Nginx commands below. Create it first using the **Run Nginx** row.

| Task | Command | Explanation |
|---|---|---|
| Run Nginx in background | `docker run -d --name web-demo -p 127.0.0.1:8080:80 nginx:alpine` | `-d` detaches; `--name` names the container; `-p` maps localhost port 8080 to container port 80. |
| List running containers | `docker ps` | Displays running containers. |
| Include stopped containers | `docker ps -a` | `-a` includes containers that have exited. |
| See container logs | `docker logs web-demo` | Displays its stdout and stderr. |
| Follow new logs | `docker logs -f web-demo` | `-f` streams additional output; press Ctrl+C to stop following. |
| Show last 20 log lines | `docker logs --tail 20 web-demo` | Limits initial output to the last 20 lines. |
| Run command inside container | `docker exec web-demo nginx -t` | Checks Nginx configuration inside the running container. |
| Open an interactive shell | `docker exec -it web-demo sh` | `-i` keeps stdin open; `-t` creates a terminal. Type `exit` to leave. |
| Inspect a container | `docker inspect web-demo` | Displays configuration, mounts, network, and runtime details. |
| Check resource use | `docker stats --no-stream web-demo` | Shows a single resource-usage snapshot instead of streaming. |
| Stop gracefully | `docker stop web-demo` | Sends a stop signal and waits before forcing termination. |
| Restart an existing container | `docker restart web-demo` | Stops and starts the same container. |
| Start a stopped container | `docker start web-demo` | Starts without creating another container. |
| Remove stopped container | `docker rm web-demo` | Deletes the container, not its image. Stop it first. |
| Force-remove container | `docker rm -f web-demo` | Stops/removes a container immediately; use only when needed. |
| Run disposable interactive shell | `docker run --rm -it alpine:3 sh` | Opens Alpine's shell; `--rm` cleans up after you exit. |
| Pass an environment variable | `docker run --rm -e DEMO=hello alpine:3 sh -c 'echo "$DEMO"'` | `-e` injects `DEMO` into the container. |
| Give a container restart policy | `docker run -d --restart unless-stopped --name stable-web nginx:alpine` | Restarts after crashes/daemon restart unless explicitly stopped. Remove with `docker rm -f stable-web`. |

**Important difference:** `docker run` creates a **new** container. `docker start` reuses one. Changing your application's source code does not update an image or running container automatically; rebuild the image and recreate the container.

## Dockerfile and .dockerignore

The example below packages a small **Node.js / Express** application. Its `package.json`, `package-lock.json`, and `index.js` are shown in Section 11. Create them before running `docker build`.

**File: `Dockerfile`**

```dockerfile
FROM node:24-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY index.js ./

ENV NODE_ENV=production
EXPOSE 3000
USER node

CMD ["node", "index.js"]
```

| Instruction | Example from the file | Exact purpose |
|---|---|---|
| `FROM` | `FROM node:24-alpine` | Choose the base image. |
| `WORKDIR` | `WORKDIR /app` | Set the working directory inside the image. |
| `COPY` | `COPY package*.json ./` | Copy package files into `/app`. |
| `RUN` | `RUN npm ci --omit=dev` | Install production dependencies **while building** the image. |
| `ENV` | `ENV NODE_ENV=production` | Set a runtime environment variable. |
| `EXPOSE` | `EXPOSE 3000` | Document the app's container port; this does **not** publish the port. |
| `USER` | `USER node` | Run the app as the pre-existing non-root `node` user. |
| `CMD` | `CMD ["node", "index.js"]` | Default command **when a container starts**. |
| `ENTRYPOINT` (alternative) | `ENTRYPOINT ["node", "index.js"]` | Sets the executable for a container; an alternative to the sample `CMD` for this app, not an extra line to add here. |

**`RUN` vs `CMD` vs `ENTRYPOINT`:** `RUN` executes during image build. `CMD` sets a default command that `docker run IMAGE OTHER_COMMAND` can replace. `ENTRYPOINT` fixes the main executable (the additional arguments are passed to it); with `CMD`, it can provide default arguments. The `ENTRYPOINT` row is an **alternative** instruction, not part of the example Dockerfile. For this Node app, `CMD` alone is sufficient.

**File: `.dockerignore`**

```gitignore
node_modules
.git
.env
.env.*
coverage
npm-debug.log*
*.md
```

`.dockerignore` excludes matches from the **build context**, reducing upload/build size and avoiding accidental inclusion of local files. The `package-lock.json` file must **not** be excluded because `npm ci` needs it.

**Build context:** In `docker build -t docker-node-demo:1.0 .`, the final `.` is the folder containing the Dockerfile and app files. With `COPY index.js ./`, Docker can only copy `index.js` if it is included in that context.

## Ports and Networking

The left side of `HOST_PORT:CONTAINER_PORT` is the **host** port; the right side is the application's port inside the **container**. A container must actually listen on that internal port.

| Task | Command | Explanation |
|---|---|---|
| Publish Nginx to host | `docker run -d --name public-web -p 8081:80 nginx:alpine` | Maps host port 8081 to port 80; default bind may expose it on external host interfaces. |
| Publish only on localhost | `docker run -d --name local-web -p 127.0.0.1:8082:80 nginx:alpine` | Accessible on the host via `http://localhost:8082`, not intentionally on all interfaces. |
| Check published ports | `docker port local-web` | Prints the port mappings of a running container. |
| List networks | `docker network ls` | Shows Docker networks. |
| Create a user-defined network | `docker network create lab-net` | Makes a bridge network with built-in container-name DNS resolution. |
| Attach a container to network | `docker run -d --name lab-web --network lab-net nginx:alpine` | Runs Nginx on `lab-net` without publishing its ports to the host. |
| Test communication by name | `docker run --rm --network lab-net busybox:1.36 wget -qO- http://lab-web` | A second container reaches `lab-web` on the same network using its name. |
| Inspect network membership | `docker network inspect lab-net` | Displays attached containers and network details. |
| Delete the lab containers | `docker rm -f lab-web` | Removes the network's test server. |
| Remove network | `docker network rm lab-net` | Removes the now-unused lab network. |

Clean up the first two examples with `docker rm -f public-web local-web`.

**Container-to-container access:** On the same user-defined Docker network, use a service/container name such as `http://lab-web:80`, **not** `localhost` (which points back into the calling container).

## Volumes and Storage

A container's writable layer is usually lost when that container is removed. **Named volumes** persist independent of containers. **Bind mounts** map a path on your computer into a container.

| Task | Command | Explanation |
|---|---|---|
| Create named volume | `docker volume create redis-lab-data` | Creates reusable Docker-managed storage. |
| List volumes | `docker volume ls` | Shows available named volumes. |
| Inspect volume | `docker volume inspect redis-lab-data` | Shows mount point and metadata. |
| Run Redis using volume | `docker run -d --name redis-lab -v redis-lab-data:/data redis:7-alpine redis-server --appendonly yes` | Mounts the volume at Redis's data directory and enables append-only persistence. |
| Write a test value | `docker exec redis-lab redis-cli SET greeting hello` | Stores a sample key in Redis. |
| Remove the old container | `docker stop redis-lab && docker rm redis-lab` | Removes the container but **preserves** the named volume. |
| Start another container on same volume | `docker run -d --name redis-lab2 -v redis-lab-data:/data redis:7-alpine redis-server --appendonly yes` | Reuses the same persistent data. |
| Verify persistence | `docker exec redis-lab2 redis-cli GET greeting` | Should return `hello` once Redis is ready. |
| Remove test container | `docker rm -f redis-lab2` | Releases the volume for deletion. |
| Remove volume (deletes stored data) | `docker volume rm redis-lab-data` | Permanently removes that volume; do this only after the lab. |

### Bind-mount example

Create a small local website in **Git Bash or Linux Bash**:

```bash
mkdir -p site
printf '<h1>Hello from a bind mount</h1>\n' > site/index.html
```

Then run:

```bash
docker run -d --name bind-web -p 127.0.0.1:8083:80 \
  --mount type=bind,source=./site,target=/usr/share/nginx/html,readonly \
  nginx:alpine
```

Open `http://localhost:8083`, edit `site/index.html` on your computer, and refresh to see the change. Clean up with `docker rm -f bind-web`. If your Windows Git Bash installation rewrites Linux-style paths incorrectly, run this example in PowerShell/WSL or adjust Docker Desktop's path handling.

**Remember:** `docker rm` does not remove a named volume. `docker volume rm` deletes its contents. Do not delete database volumes unless you intentionally want to erase that data.

## Docker Hub

Docker Hub image names commonly use `USERNAME/REPOSITORY:TAG`. **Build locally first**, then give that existing image a Docker Hub tag. Replace `YOUR_USERNAME` with your actual Docker Hub username.

| Task | Command | Explanation |
|---|---|---|
| Sign in | `docker login` | Authenticates to Docker Hub using the supported interactive flow. |
| Confirm local image exists | `docker image ls docker-node-demo` | Shows the image built in Section 11. |
| Tag for your account | `docker tag docker-node-demo:1.0 YOUR_USERNAME/docker-node-demo:1.0` | Creates a registry-qualified tag for the **same image**; no rebuild. |
| Check the new tag | `docker image ls YOUR_USERNAME/docker-node-demo` | Verifies that the exact tag you plan to push exists locally. |
| Upload tagged image | `docker push YOUR_USERNAME/docker-node-demo:1.0` | Pushes this exact tag to Docker Hub. |
| Download it elsewhere | `docker pull YOUR_USERNAME/docker-node-demo:1.0` | Pulls it from your Docker Hub repository. |
| Sign out | `docker logout` | Ends the Docker CLI registry session. |

**Why `tag does not exist` happens:** `docker push YOUR_USERNAME/docker-node-demo:1.0` fails if you only built `docker-node-demo:1.0` but have **not** created the `YOUR_USERNAME/docker-node-demo:1.0` tag. `docker tag` fixes the name without rebuilding.

**Avoid unwanted overwrites:** Use a version like `1.0` instead of assuming `latest`. Never publish credentials, `.env`, or private files inside the image.

## Docker Compose

Compose uses **`compose.yaml`** to declare services and settings. The service name (here, `app`) is a YAML identifier, **not** automatically an image name or a container name. Compose normally generates container names such as `PROJECT-app-1` and a default network.

**File: `compose.yaml`** (for the Node application in Sections 4 and 11)

```yaml
services:
  app:
    build: .
    image: docker-node-demo:1.0
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      NODE_ENV: production
    restart: unless-stopped
```

| YAML field | Value in our file | Explanation |
|---|---|---|
| `services:` | Top-level key | Holds the service definitions. |
| `app:` | Service name | Used in `docker compose logs app` and `docker compose exec app sh`. |
| `build: .` | Current directory | Uses the project Dockerfile to build the image. |
| `image: docker-node-demo:1.0` | Explicit image tag | Prevents confusion over Compose's automatically generated build-image name. |
| `ports:` | `127.0.0.1:3000:3000` | Host-only port 3000 maps to app port 3000. |
| `environment:` | `NODE_ENV: production` | Sets an environment variable inside the container. |
| `restart: unless-stopped` | Restart policy | Restarts when appropriate unless explicitly stopped. |

**`build` vs `image`:** `build: .` tells Compose **how to build**. `image:` gives the resulting image a name/tag; with no `build`, it normally uses an existing or pulled image. Including both is valid.

**`container_name` is optional.** Compose already creates a container name. Adding `container_name: my-container` fixes that name but can interfere with scaling; don't add it without a reason. `docker compose` is the modern CLI form; `docker-compose` is the older standalone spelling.

**Multiple services:** Add another service under `services:` when your application actually needs it. Compose places services on the same default network; they can reach one another using their **service names**. You don't need to manually create the default network.

## Docker Compose Commands

Run these commands in the directory containing the `compose.yaml` file from the previous section.

| Task | Command | Explanation |
|---|---|---|
| Check Compose syntax and values | `docker compose config` | Renders the effective Compose configuration; review it before running. |
| Build app image | `docker compose build app` | Builds the image for the `app` service. |
| Start in background | `docker compose up -d` | Creates/starts the declared services; `-d` detaches. |
| Rebuild and recreate as needed | `docker compose up --build -d` | Rebuilds before starting, useful after changing source or Dockerfile. |
| Show Compose containers | `docker compose ps` | Lists this project's service containers. |
| Show service logs | `docker compose logs app` | Displays `app` logs. |
| Follow service logs | `docker compose logs -f app` | Streams new `app` output; Ctrl+C leaves services running. |
| Check a command inside service | `docker compose exec app node --version` | Runs `node --version` inside the running `app` container. |
| Open shell inside service | `docker compose exec app sh` | Opens a shell in the running app container. |
| Restart a service | `docker compose restart app` | Restarts without rebuilding; does not apply changed source/image. |
| Run a one-off command | `docker compose run --rm app node --version` | Creates a temporary service container and removes it afterward. |
| Stop without removing | `docker compose stop` | Stops service containers; they can be started again. |
| Start stopped services | `docker compose start` | Starts existing Compose containers. |
| Stop and remove services | `docker compose down` | Removes Compose containers and its default network; normally keeps named volumes. |
| Remove Compose volumes too | `docker compose down -v` | **Deletes** Compose-managed named/anonymous volumes and stored data. Do not use casually. |

**When to use which:** `docker run` starts an individual container from CLI options; `docker compose up` starts the services specified in YAML. `docker compose down` does **not** normally delete built images. `docker compose down -v` adds potentially destructive volume deletion (the example `compose.yaml` itself declares no volumes).

## Troubleshooting and Cleanup

| Problem | Likely cause | Specific check or fix |
|---|---|---|
| `Cannot connect to Docker daemon` | Docker Desktop/Engine is not running | Start Docker Desktop, then run `docker version` and check the **Server** section. |
| `pull access denied` | Image name is wrong, image is private, or it has not been pushed | Run `docker image ls`; use the correct local tag or `docker login` for a private repository. |
| `tag does not exist` on push | The exact Docker Hub-qualified tag is missing | Run `docker tag docker-node-demo:1.0 YOUR_USERNAME/docker-node-demo:1.0`, then push that exact tag. |
| `port is already allocated` | Another process/container is using the host port | Run `docker ps`; stop conflicting container or change `-p 8080:80` to another host port such as `8084:80`. |
| Container stops immediately | Its main process exits or crashes | Run `docker ps -a`, then `docker logs CONTAINER_NAME`; inspect the application error. |
| `npm ci` fails in Dockerfile | `package-lock.json` is missing or out of sync | Run `npm install` in the project folder, commit the lockfile, then rebuild. |
| `COPY ... not found` | File absent from build context or excluded by `.dockerignore` | Check the final `.` in `docker build ... .` and verify the file is present. |
| Browser cannot reach app | Wrong port mapping or app only listens on `127.0.0.1` inside the container | Verify `docker ps`; ensure Node listens on `0.0.0.0:3000`, then test `curl http://localhost:3000`. |
| Source change not visible | Image or container was not rebuilt/recreated | Run `docker compose up --build -d` or build image and recreate `docker run` container. |
| Compose service not found | Wrong service name or wrong working directory | Run `docker compose config --services`; use the listed name, such as `app`. |
| `docker compose exec` fails | Service container is not running | Run `docker compose ps` and `docker compose logs app` first. |
| Data disappears after deletion | Data lived only in the container layer, or `down -v` removed it | Use a named volume for persistent data and avoid unintended volume deletion. |
| Too much Docker disk usage | Unused images, stopped containers, or build cache | Run `docker system df` and remove only confirmed unused resources. |

### Safe cleanup commands

| Task | Command | Explanation |
|---|---|---|
| Remove one stopped container | `docker rm web-demo` | Deletes just that container. |
| Delete unused stopped containers | `docker container prune` | Prompts before deleting **all** stopped containers. |
| Delete dangling images | `docker image prune` | Removes untagged image data not referenced by containers. |
| Show disk consumption | `docker system df` | Review disk usage before cleanup. |
| Delete named lab volume | `docker volume rm redis-lab-data` | Permanently deletes data if the volume is unused. |

**Avoid blindly running** `docker system prune -a --volumes` or `docker compose down -v` on projects with valuable images/data. Also avoid using `docker rm -f` on containers you have not identified.

## Practical Projects

The steps below use only files and commands explained in this guide. They are independent except that Projects 2, 3, and 4 use the same Node.js application.

### Project 1: Run a ready-made Nginx web server

```bash
docker pull nginx:alpine
docker run -d --name web-demo -p 127.0.0.1:8080:80 nginx:alpine
docker ps
curl http://localhost:8080
docker logs --tail 20 web-demo
docker stop web-demo
docker rm web-demo
```

**Expected:** The `curl` command returns the default Nginx HTML page. Stopping/removing `web-demo` removes the container, but the downloaded Nginx image remains.

### Project 2: Build and run a Node.js / Express app

**Requirement:** Node.js and npm installed locally to generate the lockfile, plus a working Docker Engine. Use a new empty directory for this project.

```bash
mkdir docker-node-demo
cd docker-node-demo
```

**File: `package.json`**

```json
{
  "name": "docker-node-demo",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^5.1.0"
  }
}
```

**File: `index.js`**

```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.json({ message: 'Hello from Docker!' });
});

app.listen(3000, '0.0.0.0', () => {
  console.log('Server listening on port 3000');
});
```

Create the **`Dockerfile`** and **`.dockerignore`** exactly as shown in [Section 4](#dockerfile-and-dockerignore). Then generate `package-lock.json` and build/run the app:

```bash
npm install
docker build -t docker-node-demo:1.0 .
docker run -d --name node-demo -p 127.0.0.1:3000:3000 docker-node-demo:1.0
curl http://localhost:3000
docker logs node-demo
docker stop node-demo
docker rm node-demo
```

**Expected response:** `{"message":"Hello from Docker!"}`. The container runs `node index.js` because the Dockerfile defines `CMD ["node", "index.js"]`. If port 3000 is busy, change **only the host port**, for example `-p 127.0.0.1:3001:3000`, and open `http://localhost:3001`.

### Project 3: Run the same Node app with Compose

In the same `docker-node-demo` folder, create **`compose.yaml`** exactly as shown in [Section 8](#docker-compose). Then:

```bash
docker compose config
docker compose up --build -d
docker compose ps
curl http://localhost:3000
docker compose logs app
docker compose exec app node --version
docker compose down
```

**Expected:** The same JSON response. Compose manages the service named `app`; there is no need to give it a second custom container name. `docker compose up --build -d` builds an image and starts its container; the resulting image is tagged `docker-node-demo:1.0` because `image:` is set explicitly.

### Project 4: Publish the existing image to Docker Hub

Use the image already built in Projects 2/3. **Replace `YOUR_USERNAME`** in every command with your Docker Hub account name:

```bash
docker image ls docker-node-demo
docker login
docker tag docker-node-demo:1.0 YOUR_USERNAME/docker-node-demo:1.0
docker image ls YOUR_USERNAME/docker-node-demo
docker push YOUR_USERNAME/docker-node-demo:1.0
docker logout
```

**Expected:** Docker Hub displays the uploaded image under your account. No extra `docker build` is necessary to change the tag. If your repository name, access, or tag differs, fix the exact image reference instead of rebuilding blindly.

### Project 5: Practice diagnosing a stopped container

This disposable example deliberately exits immediately:

```bash
docker run --name exited-demo alpine:3 sh -c 'echo "Task finished"'
docker ps
docker ps -a
docker logs exited-demo
docker inspect exited-demo --format '{{.State.ExitCode}}'
docker rm exited-demo
```

**Expected:** `exited-demo` is absent from `docker ps` but present in `docker ps -a`. Its logs say `Task finished`, and its exit code is `0`. An exited container is **not always an error**; a command-line task is expected to stop when finished.

### Final checklist

- [ ] Explain the difference between image, container, Dockerfile, and Compose service.
- [ ] Pull, build, tag, inspect, and remove a test image.
- [ ] Run, inspect, log, enter, stop, restart, and remove containers.
- [ ] Explain why `-p 8080:80` is different from `EXPOSE 80`.
- [ ] Test a named network and a persistent volume.
- [ ] Run a real Node.js app with `docker run` and `docker compose up`.
- [ ] Publish a correctly tagged image to Docker Hub.
- [ ] Diagnose an exited container and clean up test resources safely.

---

## Official References

- [Docker overview](https://docs.docker.com/get-started/docker-overview/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Compose file reference](https://docs.docker.com/reference/compose-file/)
- [Compose CLI reference](https://docs.docker.com/reference/cli/docker/compose/)
- [Docker volumes](https://docs.docker.com/engine/storage/volumes/)
- [Docker Hub repositories](https://docs.docker.com/docker-hub/repos/)

*Commands require a compatible Docker Engine, permissions, and the necessary images/tools. Production deployments need separate security, availability, and monitoring design; this guide is for practical fundamentals.*
