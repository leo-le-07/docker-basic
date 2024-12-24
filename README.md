# Table of Contents

- [Introduction](#introduction)
- [Images and Containers](#images-and-containers)
  - [Basic Relationship Between Image and Container](#basic-relationship-between-image-and-container)
  - [Understanding Docker Image Layers](#understanding-docker-image-layers)
  - [EXPOSE - Declaration for Developers or Tools](#expose---declaration-for-developers-or-tools)
  - [Use case: `RUN node app.js` is Incorrect in Dockerfile when Building Image](#use-case-run-node-appjs-is-incorrect-in-dockerfile-when-building-image)
- [Containers](#containers)
  - [Key Fundamentals](#key-fundamentals)
  - [Interactive Mode](#interactive-mode)
- [CMD and ENTRYPOINT in Dockerfile](#cmd-and-entrypoint-in-dockerfile)
- [`docker exec` command](#docker-exec-command)
  - [Common Use Cases for `docker exec`](#common-use-cases-for-docker-exec)
- [Managing Data & Working with Volumes](#managing-data--working-with-volumes)
  - [Data Types](#data-types)
  - [Volumes](#volumes)
  - [Bind Mount](#bind-mount)
  - [Comparing Volumes and Bind Mounts](#comparing-volumes-and-bind-mounts)
- [`.dockerignore`](#dockerignore)
  - [Use Case: Don't `COPY` everything](#use-case-dont-copy-everything)
- [Environment Variables with `ENV` and `--env`](#environment-variables-with-env-and---env)
  - [Setting `ENV` in Dockerfile](#setting-env-in-dockerfile)
  - [Setting with `--env` in `docker run`](#setting-with---env-in-docker-run)
  - [Using `.env` Files for Easier Management](#using-env-files-for-easier-management)
- [Using Build Arguments `ARG`](#using-build-arguments-arg)
  - [`ARG` in Dockerfile](#arg-in-dockerfile)
  - [Comparing `ARG` and `ENV` in Docker](#comparing-arg-and-env-in-docker)
- [Networking: (Cross-)Container Communication](#networking-cross-container-communication)
  - [Types of Docker Networks](#types-of-docker-networks)
  - [How Containers Communicate](#how-containers-communicate)
  - [Steps to Enable Communication Between 2 Containers](#steps-to-enable-communication-between-2-containers)
  - [Docker Compose for Multi-Container Communication](#docker-compose-for-multi-container-communication)


# Introduction

This repository contains my notes and understanding of Docker, Docker Compose, and containerized development. It is a personal reference to help me apply Docker concepts in daily work and future projects. The content covers fundamental topics like images, containers, networking, and data management, organized for easy access and updates.

# Images and Containers

## Basic Relationship Between Image and Container

- Images: Static blueprints for creating containers, containing the OS, app code, and dependencies.
- Containers: Isolated, running instances of images, with their own processes and file systems.

```
          [ Docker Image ]
   (Blueprint/Template for Containers)
             /    |    \
            /     |     \
    [Container 1] [Container 2] [Container 3]
      (Instance)    (Instance)    (Instance)
       - Runs a       - Runs a       - Runs a
         process        process        process
       - Isolated      - Isolated     - Isolated
```

## Understanding Docker Image Layers

Key Concepts:

- Layered Architecture:
  - Each Dockerfile instruction (like `FROM`, `RUN`, `COPY`, etc.) creates a new layer.
  - Layers are **read-only** and stacked to form the complete image.
- Reusability:
  - Docker uses **layer caching** to speed up builds. If a layer hasn’t changed, Docker reuses it instead of rebuilding it.
  - This improves efficiency and reduces redundancy.

How Layers Work in a Dockerfile

```docker
# Step 1: Base Image
FROM node:20         # Creates a layer from the Node.js base image

# Step 2: Set Working Directory
WORKDIR /app         # Creates a layer that sets /app as the working directory

# Step 3: Copy Dependency File
COPY package.json .  # Adds a layer with package.json

# Step 4: Install Dependencies
RUN npm install      # Adds a layer with installed dependencies

# Step 5: Copy Application Files
COPY . .             # Adds a layer with application code

# Step 6: Run the Application
CMD ["node", "app.js"] # Does NOT create a layer (used at runtime)
```

How Layers are Created:

- Each `FROM`, `WORKDIR`, `COPY`, `RUN`, and other commands creates a **new layer**.
- The `CMD` instruction does not create a layer; it specifies the command to run when a container starts.

Benefits of Image Layers:

1. Cache Reuse

- Docker caches layers to avoid rebuilding unchanged parts of the image.
- Example: if I change application code, but not dependencies (`RUN npm install`), Docker reuses the cached layer for dependencies.

2. Smaller Image Size

- Common base images (like `node:20`) are shared across multiple projects.

## EXPOSE - Declaration for Developers or Tools

`EXPOSE` instruction in Dockerfile does NOT publish a port or make it accessible to the host machine. It’s purpose during **build time**:

1. Documentation:

- `EXPOSE` indicate ports on which application inside container is designed to listen
- This is a DECLARATION for Developers or Tools, but it doesn’t actually affect runtime behavior

2. No Runtime Effect:

- By itself, `EXPOSE` does not map or publish port to host system. Containers using `EXPOSE` will still be isolated unless explicit actions (like port publishing) are take when running container.

### Comparison: EXPOSE vs Published Ports

| **Aspect**       | **EXPOSE (Build Time)**                         | **Publish Ports (Runtime)**                   |
| ---------------- | ----------------------------------------------- | --------------------------------------------- |
| Purpose          | Documentation and metadata for tools.           | Enables actual network communication.         |
| Effect           | No direct effect on network access.             | Opens and maps container ports to host ports. |
| Usage            | `EXPOSE <port>`                                 | `docker run -p <host_port>:<container_port>`  |
| Scope            | Build time                                      | Runtime                                       |
| Access from Host | No access unless ports are explicitly published | Host can access via the published port.       |

## Use case: `RUN node app.js` is Incorrect in Dockerfile when Building Image

**Problem: Misunderstanding of `RUN`**
Purpose of `RUN`:

- Used to execute commands **at build time** to modify image (e.g., install dependencies, copy files, or configure the environment).
- The output of `RUN` commands is baked into image, but the commands themselves are not part of container’s runtime process

Why it’s wrong here:

- `node app.js` starts Node.js application, which is **a runtime process** that should run when container starts, not when image is being built
- When `RUN node app.js` executes during the build, it will attempt to run application, and the build process will either hang indefinitely (waiting for app to stop) or fail.

**Correct Approach: Use `CMD` for Runtime Commands**
To run application when container starts, use `CMD` or `ENTRYPOINT` instruction in Dockerfile:

```bash
CMD ["node", "app.js"]
```

## Container - Key Fundamentals

### Main Process

- Container’s lifecycle is directly link to main process it runs
- When create / start a container, Docker runs process specified in image’s `ENTRYPOINT` or `CMD` directives
- **Once process finished, container exits**

Examples:

- Running a web server (e.g., `nginx`, `httpd`) keeps the container running as long as the server process is alive.
- Running a script that ends will cause the container to exit.

### Long-Running vs One-Off Containers

- **Long-Running Containers**: Keep running until explicitly stopped or killed
  - Examples: Web servers, databases, background services
- **One-Off Containers**: Run a single task or command, then exit
  - Examples: Run a script, execute a test, copy files

### Detached vs Interactive Mode

- Detached Mode (`-d`): runs container in the background without user interaction
- Interactive Mode (`-it`); allows user interaction, such as accessing a shel inside container

Example:

```bash
docker run -d nginx   # Detached mode
docker run -it ubuntu bash   # Interactive mode
```

### Process Isoation

- Containers are isolated from host and other containers
- Each container has its own process tree, network stack, and file system

### Container Logs

- Containers write logs to STDOUT and STDERR

```bash
docker logs <container_id_or_name>
```

## Interactive Mode

Related to a running container, ability to interact directly with the processes inside it through a terminal session. This mode allows users to provide input, run commands, and see real-time output inside container.

### Key Flags for Interactive Mode

Typically use following flags with `docker run`:

- `-i` (Keep STDIN Open): Keeps the standard input (**STDIN**) **open** so you can interact with the container.
- `-t` (Allow a TTY): Allocates a pseudo-terminal (TTY), providing a **terminal interface** to the container.

When combined, `-it` lets you interact with the container as if it were a **local shell session.**

### When to Use Interactive Mode

- Debugging issues inside the container.
- Testing commands inside a running container environment.
- Accessing a shell to manually manage files or processes.
- Exploring the container's runtime environment for learning or configuration purposes.

### Scenarios When Missing either `-i` or `-t` flag:

1. Only `-t` provided

```bash
docker run -t ubuntu bash
```

What happens:

- A pseudo-terminal (TTY) is created, but STDIN is not kept open.
- You won't be able to provide input to the container interactively.
- Example:
  - Running `bash` will display a terminal-like prompt, but you can't enter commands interactively.

1. Only `-i` provided

```bash
docker run -i ubuntu bash
```

What happens:

- STDIN remains open, so you can provide input, but the session lacks a TTY.
- The **output will be difficult to read because it won’t have the proper terminal formatting (e.g., no command prompt, garbled output).**
- Example:
  - You can type commands, but the lack of a terminal interface makes it hard to use for interactive purposes.

## `CMD` and `ENTRYPOINT` in Dockerfile

They are both used to specify what happens when a container starts. However, they behave differently and are suited for different use cases.

Use `CMD`: if container should have a default command but allow users to override it:

Example:

```dockerfile
CMD ["npm", "start"]
```

- Default: `npm start`
- Override: `docker run my-image node app.js`

Use `ENTRYPOINT`: if container must always run a specific command (append)

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

- Always runs `nginx` (by append) and allows addition arguments

## `docker exec` command

- Used to run commands inside a running container
- It’s incredibly useful for interacting with containers after they have started, enabling real-time debugging, monitoring, or performing maintenance tasks

```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

### Common Use Cases for `docker exec`

- Debugging: Access a shell inside a running container to inspect files, logs, or processes.

```bash
docker exec -it my-container bash
```

- Running One-Off Commands: Execute a single command inside a running container without starting a new one.

```bash
docker exec my-container ls -l /app
```

- Monitoring Logs or Processes

```bash
docker exec my-container ps aux
docker exec my-container tail -f /var/log/app.log
```

- Testing Configurations

```bash
docker exec my_container cat /etc/nginx/nginx.conf
```

- Restart service:

```bash
docker exec my_container service nginx restart
```

- Install dependencies:

```bash
docker exec my_container apt-get update && apt-get install curl
```

# Managing Data & Working with Volumes

## Data Types

| Data Type          | Description                                                                                           | Stored In                     | Lifecycle                                               | Example                                       |
| ------------------ | ----------------------------------------------------------------------------------------------------- | ----------------------------- | ------------------------------------------------------- | --------------------------------------------- |
| Application Data   | Code, libraries, environment, configuration baked into image                                          | Docker Image                  | Immutable, persists as long as image exists             | Source code, package.json, app.js, etc.       |
| Temporary App Data | Data generated during runtime of the container, not meant to persist beyond the container's lifecycle | Container Writable Layer      | Ephemeral; deleted when container is removed            | User input during session, cached files, logs |
| Permanent App Data | Critical, persistent data that must survive container restarts or removals                            | Volumes or external databases | Persistent; exists independently of container lifecycle | User accounts, databases, or uploaded files   |

> Two Types of External Data Storages: Volumes vs Bind Mounts

### Use case: Manage Code Changes in an Express App Without Volumes

Problem statement: When developing an Express app in Docker, every code change requires the following steps:

- Stop current running container
- Remove stopped container (optional but recommended)
- Rebuild Docker Image
- Start new container

Drawbacks of this workflow:

- Time-consuming
- Repetitive manual steps

Solution: use Docker Volumes or Bind Mounts

> Enables container to access files from your host or persist data generated by container.

## Volumes

- Volumes are folders on your host machine hard drive which are mounted (made available / mapped) into Containers
- Managed by Docker

### Key Benefits of Volumes

- **Data Persistence:** Ensures critical data (e.g., databases, uploaded files) remains intact even if the container is removed or restarted.
- **Container Independence**: Volumes are managed outside the container, allowing multiple containers to share the same data.
- **Improved Performance**: Volumes are optimized for container data access, often faster than bind mounts (direct host file system access).
- **Portability**: Volumes can be backed up, migrated, and restored easily across different environments or hosts.
- **Host File System Decoupling**: Volumes are isolated from the host’s file system structure, reducing the risk of unintended access or modification.
- **Ease of Management**: Docker provides commands to manage volumes (`docker volume create`, `docker volume inspect`, etc.), making it easier to organize persistent data

Example: Persisting MySQL database data with a named volume

```bash
docker run -d --name mydb -v my-db-data:/var/lib/mysql mysql
```

- Maps the named volume `my-db-data` to the container's `/var/lib/mysql` directory.
- Ensures database data persists even if the `mydb` container is removed.

### `VOLUME` Instruction in Dockerfile and Anonymous Volumes

- When we define `VOLUME` in Dockerfile, Docker automatically creates a volume for specified directory when a container is started
- If we **don’t explicitly specify a volume during container runtime**, Docker creates an **anonymous** volume for that path
- Anonymous Volumes will be removed when a container, that was started with `--rm` is stopped
- Anonymous Volume not really useless, we can use it to prioritize container-internal paths higher than external paths (when Bind Mount)

```dockerfile
FROM ubuntu
VOLUME /app/data
CMD ["echo", "Data volume created"]
```

- When a container is started from this image, Docker creates an anonymous volume with a randomly generated name (e.g., d3f3b4c0c7d2c3a8c) for `/app/data`
- The directory `/app/data` in container is now backed by this anonymous volume. Any data written to `/app/data` will be stored in the volume even after the container stops or removed.

```bash
docker run my-image
docker volume ls
```

We will see an automatically created volume like:

```
DRIVER    VOLUME NAME
local     d3f3b4c0c7d2c3a8c
```

### Better Practice: Use Named Volumes for Data Persistence

1. Specify named volumes at runtime

```bash
docker run -v my-data:/app/data my-image
```

2. Use Bind Mounts for Development

```bash
docker run -v $(pwd):/app my-image
```

3. Docker Composer: Define named volumes in `docker-compose.yml`

```yaml
services:
  app:
    image: my-app
    volumes:
      - my-data:/app/data
volumes:
  my-data:
```

## Bind Mount

- Bind Mount is a method in Docker to map a directory or file from Host System directly to a specific path in container. This allows the container to access and interact with the host's files in real-time.
- Managed by … You

### **Benefits of Bind Mounts**

- Real-Time Updates: no need to rebuild Docker Image for code changes
- Flexibility: directly interact with host files, making debugging and testing faster
- Simplicity: easy to set up with a single `-v` flag

Example: Run an Express app with a bind mount

```bash
docker run -v $(pwd):/app -w /app -p 3000:3000 node:14 node app.js
```

- `-v $(pwd):/app`: Maps the current directory on the host (`$(pwd)`) to `/app` in the container.
- Effect: Any code changes on the host are immediately reflected in the container, eliminating the need to rebuild.

### Use Case: Bind Mount Overwriting Dependencies in a Dockerized Express App

**Problem**: You have a simple Express project with a Dockerfile:

```dockerfile
FROM node:20

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

EXPOSE 80

CMD ["node", "server.js"]
```

```bash
docker build -t my-app .
docker run -d -p 3000:80 --name my-app -v feedback:/app/feedback -v $(pwd):/app my-app
```

**Issue**: Container fails to start due to bind mount overwriting `node_modules` directory

```bash
npm ERR! Cannot find module 'express'
```

How the Issue Occurs:

1. During the image build process, `COPY package.json .` and `RUN npm install` correctly install the `express` module in `/app/node_modules` **inside the image**.
2. The `v $(pwd):/app` bind mount **OVERRIDES** the `/app` directory in the container with the content from your host machine's current directory (`$(pwd)`). The host directory likely does **not include `node_modules`**, because `node_modules` is usually ignored via `.gitignore` or not copied manually.
3. Result: The container tries to run the application, but `express` (and other dependencies) are missing because the `/app/node_modules` directory was replaced by the empty or incomplete host directory.

**Solution 1**: Adjust Bind Mount and Volume Usage

- Use a named volume specifically for node_modules to avoid overwriting it:

```bash
docker run -d -p 3000:80 \
  --name feedback-app \
  -v feedback:/app/feedback \
  -v $(pwd):/app \
  -v /app/node_modules \
  example3-feedback-node:latest
```

- Exclude “node_modules” from bind mount: here, `/app/node_modules` is mounted as a anonymous volume. Preventing it from being affected (overridden) by the host's `node_modules`.
- **Note: the order of `-v` flags DOES matter because Docker processes volume mounting sequentially, and later mounts can overwrite earlier ones**

**Solution 2**: Install Dependencies on the Host (not recommended)

- Install node_modules on the host and let the bind mount provide them to the container:

```bash
npm install
docker run -d -p 3000:80 \
  --name feedback-app \
  -v feedback:/app/feedback \
  -v $(pwd):/app \
  example3-feedback-node:latest
```

**Solution 3**: Avoid overwriting `/app`

- Change the bind mount to include only necessary files (e.g., source files) instead of mounting the entire directory:

```bash
docker run -d -p 3000:80 \
  --name feedback-app \
  -v feedback:/app/feedback \
  -v $(pwd)/src:/app/src \
  example3-feedback-node:latest
```

**Best Practice:**

For development:

- Use bind mounts to synchronize only the code (`src`, `public`, etc.) while preserving the `node_modules` directory inside the container using a volume.

For production:

- Avoid bind mounts entirely, and ensure dependencies are fully included in the image during the build.

## Comparing Volumes and Bind Mounts

| Aspect             | Volumes                                                    | Bind Mounts                                                    |
| ------------------ | ---------------------------------------------------------- | -------------------------------------------------------------- |
| Definition         | Managed by Docker, stored in Docker's storage area         | Directly maps a host directory to a container directory        |
| Use Case           | Persistent data storage, sharing data between containers   | Development, sharing source code, configuration files          |
| Setup (command)    | `-v volume_name:/path/in/container`                        | `-v /host/path:/path/in/container`                             |
| Performance        | Generally better performance as Docker manages the storage | Performance depends on the host filesystem                     |
| Portability        | More portable, as volumes are managed by Docker            | Less portable, as it depends on the host's directory structure |
| Data Sharing       | Easily share data between multiple containers              | Can share data, but more complex to manage                     |
| Backup and Restore | Easier to backup and restore using Docker commands         | Requires manual backup and restore of host directories         |
| Example            | `-v my_volume:/app/data`                                   | `-v $(pwd)/src:/app/src`                                       |

**Problem**: `COPY . .` in Dockerfile copies everything from the host to the container, including unnecessary files like `node_modules`, `.git`, and build artifacts.
- Include unnecessary files: for example `.git`, temporary files, `.env`
- Increase image size
- Expose sensitive data: file like `.env` or SSH keys might be unintentionally included, leading to security vulnerabilities
- Break build caching: any change in build context (even a small, irrelevant file) invalidates entire build cache, forcing Docker to rebuild image from scratch

**Solution**: Use `.dockerignore` file to exclude unnecessary files and directories from the build context
```
node_modules
.git
*.log
.env
```

## `.dockerignore`
By using a .dockerignore file, you can exclude unnecessary files and directories from this build context. This has several benefits:

1. Reduced Build Context Size: Only the necessary files are sent to the Docker daemon, making the build process faster and more efficient.
2. Improved Performance: Smaller build contexts mean quicker builds and less resource usage.
3. Security: Sensitive files (e.g., .env files) are not included in the build context, reducing the risk of exposing sensitive information.
4. Cleaner Images: Excluding irrelevant files helps in creating cleaner and more maintainable Docker images.

Example:
If you have a project directory on your host with the following structure:

```text
/my-project
  |-- .git/
  |-- node_modules/
  |-- src/
  |-- Dockerfile
  |-- .dockerignore
  |-- .env
```

The `.dockerignore` file can be used to exclude unnecessary files and directories:

```text
node_modules
.git
*.log
.env
```
- When you build your Docker image, the `node_modules`, `.git` directories, all `.log` files, and the `.env` file will be excluded from the build context. 
- This means they won't be sent to the Docker daemon or included in the Docker image, resulting in a more efficient and secure build process.

### Use Case: Don't `COPY` everything
**Problem**: `COPY . .` in Dockerfile copies everything from the host to the container, including unnecessary files like `node_modules`, `.git`, and build artifacts.

**Solution**: Use `.dockerignore` file to exclude unnecessary files and directories from the build context

## Environment Variables with `ENV` and `--env`

### Setting `ENV` in Dockerfile

- Purpose: Define default environment variables that are baked into the Docker image. These variables are available whenever a container is started from the image.

```docker
ENV <key>=<value>
```

- Scope: variables set with `ENV` are available during image build process and in any container created from image

Example:

```docker
FROM node:20

WORKDIR /app

# Define environment variables
ENV NODE_ENV=production
ENV PORT=80

COPY package.json .
RUN npm install

COPY . .

EXPOSE 80

CMD ["node", "server.js"]
```

- `NODE_ENV` and `PORT` variables are included in image and available to any container created from it
- We can access variables inside container:

```bash
echo $NODE_ENV
# Output: production
```

### Setting with `--env` in `docker run`

- Purpose: pass or override environment variables at runtime when starting a container

```bash
docker run --env <key>=<value> <image>
```

- Scope: variables set with `--env` are available only to **specific container being started**

Example:

```bash
docker run --env NODE_ENV=development --env PORT=3000 -p 3000:3000 example-app 
```

- These values will **override the `ENV`** values defined in Dockerfile for this container’s runtime

### Using `.env` Files for Easier Management

Instead of setting variables individually, you can use a `.env` file to pass multiple environment variables at once.

```
NODE_ENV=development
PORT=3000
API_KEY=12345
```

```bash
docker run --env-file .env -p 3000:3000 example-app
```

- Variables from `.env` file are loaded into container’s runtime environment

### Common Mistakes to Avoid

Hardcoding secrets in Dockerfile:

- **Problem**: Sensitive data in `ENV` becomes part of the image layers, which could be shared.
- **Solution**: Use runtime variables (`-env`) or secret management tools (e.g., Docker Secrets).

Environment-Specific Configurations:

- Avoid defining environment-specific configurations (e.g., `NODE_ENV=development`) in the Dockerfile if the image is used across multiple environments.

## Using Build Arguments `ARG`

### `ARG` in Dockerfile

- Use to define variables available only during build phase `docker build`
- They’re not persisted in final image or available at runtime in container

```docker
ARG <key>[=<default_value>]
```

```docker
FROM node:20

# Define build-time variables
ARG APP_VERSION=1.0.0
ARG BUILD_ENV=development

# Use ARG values
RUN echo "Building version $APP_VERSION in $BUILD_ENV environment"

# Runtime environment variable
ENV NODE_ENV=production
CMD ["node", "server.js"]
```

Passing values during build:

```bash
docker build --build-arg APP_VERSION=2.1.0 --build-arg BUILD_ENV=production -t example-app .
# Output: Building version 2.1.0 in production environment
```

### Comparing `ARG` and `ENV` in Docker

| Aspect               | ARG                                               | ENV                                               |
|----------------------|---------------------------------------------------|---------------------------------------------------|
| Scope                | Available only during the build phase             | Available during both build and runtime           |
| Persisted in Image   | No, not persisted in the final image              | Yes, persisted in the final image                 |
| Used for             | Defining build-time variables                     | Defining environment variables for runtime        |
| Default Value        | Can have a default value specified in Dockerfile  | Can have a default value specified in Dockerfile  |

### Using `ARG` and `ENV` together

You can pass `ARG` values to `ENV` to make them available at runtime:

```docker
FROM node:20

# Build-time variables
ARG APP_VERSION

# Pass ARG to ENV
ENV APP_VERSION=$APP_VERSION
ENV NODE_ENV=production

CMD ["node", "server.js"]
```

```bash
docker build --build-arg APP_VERSION=2.1.0 -t example-app .
```

Runtime behavior - Inside container:
```bash
echo $APP_VERSION
# Output: 2.1.0
```

# Networking: (Cross-)Container Communication
## Types of Docker Networks

**Bridge Network (default):**

- Docker automatically creates a bridge network (`bridge`) for containers to communicate with each other on the same host.
- Use case: Standalone or small-scale applications running on a single host.
- Containers connected to the same bridge network can communicate with each other using **IP addresses**.

**Custom User-Defined Bridge:**

- A **custom bridge network that we create**, allowing more control over container communication.
- Use case: Recommended for multi-container applications.

## How Containers Communicate

**1. Default Bridge Network**

- Containers are assigned unique IP addresses.
- By default, they **cannot resolve each other by name** unless you use a custom network.

**2. Custom User-Defined Bridge Network**

- Containers can communicate by their **container names** as DNS names.
- Easier to manage and highly recommended for multi-container setups.

## Steps to Enable Communication Between 2 Containers

### Using Default Bridge Network

Example:

```bash
docker run -d --name container1 alpine tail -f /dev/null
docker run -d --name container2 alpine tail -f /dev/null
```

Find IP of `container1`

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container1
```

Use IP to ping:

```bash
docker exec container2 ping <container1-IP>
```

### Using a Custom Bridge Network

Create new bridge network:

```bash
docker network create my-custom-network
```

```bash
docker run -d --name container1 --network my-custom-network alpine tail -f /dev/null
docker run -d --name container2 --network my-custom-network alpine tail -f /dev/null
```

Inside `container2`, ping `container1` by name

```bash
docker exec container2 ping container1
```

### Docker Compose for Multi-Container Communication

Example:
```yaml
services:
  app1:
    image: alpine
    container_name: app1
    networks:
      - my-network
    command: tail -f /dev/null

  app2:
    image: alpine
    container_name: app2
    networks:
      - my-network
    command: tail -f /dev/null

networks:
  my-network:
    driver: bridge
```

```bash
docker-compose up -d
docker exec app2 ping app1
```