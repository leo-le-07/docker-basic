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