
##### Why Docker (Problem before Containerization) ?
"But it works on my machine".

##### What is Docker?
[What is a Container? \| Docker](https://www.docker.com/resources/what-container)
![[Container.png]]

Imp points : 
- Docker is fundamentally a Linux-native technology. It was built to leverage the **monolithic kernel** of Linux.
- Linux allows a process to be "tricked" into thinking it's isolated while still talking directly to the host kernel. This is why Docker on Linux is so fast—there is no "middleman."
- Docker cannot run directly on the Windows kernel (NT Kernel) the same way it runs on Linux. It requires a Linux environment to function because it relies on those specific low-level features we discussed: **Namespaces** and **cgroups**. Namespaces control what a process can _see_; cgroups control what a process can _use_.
- **Docker Desktop** on Windows, it sets up a lightweight Linux bridge for you : 
	- **WSL 2 (Windows Subsystem for Linux):** This is the modern, preferred method. Windows runs a real, highly optimized Linux kernel alongside your Windows kernel. Docker sits on top of that. 
	- **Hyper-V (Legacy):** Before WSL 2, Docker would create a small VM (using the Hypervisor) running a stripped-down Linux OS just to host your containers.

Two key Linux kernel features necessary for Docker (other tools like Kubernetes, Podman, LXC, systemd all use the _same_ underlying isolation mechanism)
1. **cgroups**: added to the Linux kernel around 2007 (originally by Google engineers, for resource control on their servers)
2. **namespaces**: added incrementally between 2002–2013, piece by piece (PID namespaces, network namespaces, etc., each landed separately)
### Cgroups (control groups) — "how much can you use?"
**Cgroups** answer the question: **how much of the host's physical resources (CPU, memory, disk I/O, network bandwidth) is this process tree allowed to consume?**

Think of cgroups as a **resource meter with a hard cap**, not a separate pool of resources. The process is still drawing from the _same_ physical RAM and CPU cores as everything else on the machine — cgroups just put a ceiling on how much of that shared pool a particular group of processes can use.
#### What cgroups can limit

| Resource  | What it controls                                                |
| --------- | --------------------------------------------------------------- |
| CPU       | How many CPU cycles/cores a process group can use               |
| Memory    | RAM ceiling, swap usage, out-of-memory kill behavior            |
| Block I/O | Disk read/write throughput limits                               |
| Network   | (Indirectly, via combination with namespaces + traffic shaping) |
| PIDs      | Maximum number of processes/threads a group can spawn           |

### Namespaces — "what can you see?"
**Namespaces** answer a completely different question: **what subset of the system's global state is this process allowed to _see and interact with_?**

This is about **visibility and isolation**, not resource limiting. A process inside a namespace isn't sharing less of the CPU — it's looking at a **private, filtered view** of something that, underneath, is still one global kernel structure.

#### The seven core kinds of Linux namespaces

| Namespace         | What gets a private view                                                | Real-world effect                                                                                                               |
| ----------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **PID**           | Process IDs                                                             | Your container's main process sees itself as PID 1, even though the host sees it as PID 48213                                   |
| **Mount (mnt)**   | Filesystem mount points                                                 | Container gets its own root `/`, built from image layers — can't see the host's real filesystem                                 |
| **Network (net)** | Network interfaces, IP addresses, routing tables, ports                 | Container gets its own virtual `eth0`, can have port 80 even if host's port 80 is taken                                         |
| **UTS**           | Hostname and domain name                                                | Container can have a different hostname than the host machine                                                                   |
| **IPC**           | Inter-process communication (shared memory, semaphores, message queues) | Container's IPC objects are invisible to and unreachable from the host's processes                                              |
| **User**          | User and group ID mappings                                              | A process can be "root" (UID 0) inside the container while mapped to an unprivileged UID on the host — a major security feature |
| **Cgroup**        | The cgroup hierarchy view itself                                        | Container sees only its own slice of the cgroup tree, not the host's full hierarchy                                             |


##### How Docker better over VM ? 
**Architectural Comparison**

| Feature              | Virtual Machine (VM)                                                                         | Docker Container                                                                                                              |
| -------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Architecture**     | Runs on a **hypervisor** (e.g., VMware, Hyper-V) that **virtualizes the physical hardware**. | Runs on a single host OS, **sharing its kernel** with other containers.                                                       |
| **Abstraction**      | abstraction at the app layer that packages code and dependencies together                    | abstraction of physical hardware turning one server into many servers                                                         |
| **Operating System** | Each VM includes a complete, isolated guest OS (e.g., Linux or Windows).                     | Only packages the application, binaries, and libraries needed to run, not a full OS.                                          |
| **Resource Usage**   | **Resource-intensive** (requires more RAM, CPU, and storage) due to the full OS overhead.    | **Lightweight** and efficient, as resources are shared with the host OS.                                                      |
| **Boot Time**        | Takes minutes to boot as the entire OS must load.                                            | Starts in milliseconds or seconds.                                                                                            |
| **Isolation**        | Provides strong, hardware-level isolation.                                                   | Provides process-level isolation through Linux kernel features like cgroups and namespaces.                                   |
| **Portability**      | Less portable across different platforms without compatibility considerations.               | Highly portable; can run on any system with Docker installed, regardless of the underlying OS or hardware.                    |
| Analogy              | Like building an entire house for every guest.                                               | Like renting an apartment in a building; you have your own private space, but you share the plumbing and foundation (the OS). |

**Summary of difference: Docker vs. VM**

|                  |                                                                                                                                                                                            |                                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
|                  | **Docker container**                                                                                                                                                                       | **VM**                                                                                         |
| What is it?      | Docker is a software platform to create and run Docker containers. A Docker container is an emulation of a user-space instance, the part of the operating system where user processes run. | An emulation of a physical machine—including virtualized hardware—running an operating system. |
| Virtualization   | Container abstracts operating system details from the application code.                                                                                                                    | VM abstracts hardware details from the application code.                                       |
| Objective        | Abstract hardware details and increase hardware utilization.                                                                                                                               | Improve application environment management and bring consistency across multiple environments. |
| Managed by       | The **Docker Engine** coordinates between the operating system and Docker containers.                                                                                                      | The **hypervisor** coordinates between the machine’s physical hardware and virtual machines.   |
| Architecture     | Shares resources with the underlying host kernel.                                                                                                                                          | **Runs its own kernel** and operating system.                                                  |
| Resource sharing | On-demand.                                                                                                                                                                                 | A fixed amount, set in a virtual machine image’s configuration requirements.                   |
Summary Diagram
- **Virtual Machine Architecture**:
    - `Infrastructure` -> `Host OS` -> `Hypervisor` -> `Guest OS` (with bins/libs/app) for each VM.
- **Docker Container Architecture**:
    - `Infrastructure` -> `Host OS` -> `Docker Engine` -> `Containers` (with bins/libs/app only) sharing the host kernel

![[Docker-VM.png]]

![[Docker1.png]]

## Difference : Worm's-Eye View (Low-Level Detail)
The fundamental difference lies in **where the abstraction happens.**

#### 1. Virtual Machines: Hardware Abstraction
In a VM, the **Hypervisor** is the key player. It sits between the hardware and your operating system, tricking the OS into thinking it has its own dedicated CPU, RAM, and Disk.
- **Hypervisor:** A piece of software (like VMware or Hyper-V) that creates and runs VMs.
- **Guest OS:** Because the Hypervisor simulates hardware, you must install a full Operating System (Kernel + Binaries + UI) inside every VM.
- **The Weight:** If you have 5 VMs, you are running 5 separate Kernels. This is a massive "tax" on your hardware resources.

---
#### 2. Docker: Operating System Abstraction
*Docker doesn't simulate hardware*; it leverages features built directly into the **Linux Kernel**. It treats the host's Operating System as a shared pool.
- **The Kernel:** The core of the OS that manages hardware resources. In Docker, every container shares the **Host Kernel**.
- **Namespaces (Isolation):** This is a kernel feature that creates "walls." It tells a process, "You can only see this specific network interface and this specific folder." It’s like putting blinkers on a horse.
- **Control Groups / cgroups (Resource Limits):** This kernel feature limits how much CPU or RAM a container can use. It prevents one container from "eating" all the memory and crashing the server.

---
##### The Low-Level Comparison

|**Feature**|**Virtual Machines (VMs)**|**Docker Containers**|
|---|---|---|
|**Abstraction Layer**|Hardware level (Hypervisor)|OS level (Kernel)|
|**Guest OS**|Full OS required for each VM|No Guest OS (uses Host Kernel)|
|**Startup Time**|Minutes (boots an entire OS)|Milliseconds (starts a process)|
|**Size**|Gigabytes (GBs)|Megabytes (MBs)|
|**Performance**|Near-native (with overhead)|Native (zero hypervisor overhead)|
##### The "Worm's" Analogy: The House vs. The Office
- **VM (The House):** Each VM is a standalone house. It has its own foundation, plumbing, wiring, and roof. It’s completely private but very expensive and slow to build.
- **Docker (The Office Cubicle):** Every container is a cubicle in one big office building. Everyone shares the same foundation, plumbing, and electricity (the Kernel), but the walls (Namespaces) and desk limits (cgroups) keep your work private and contained.

## [**Frequently Asked Questions**](https://k21academy.com/kubernetes/docker-vs-virtual-machine/#10)

Q1) How do Containers **enhance theCI/CD** process?
Q2) What are the primary **security concerns** when using Containers, and how can they be mitigated?
Q3) Why might an organization **choose VMs over Containers** despite the latter's advantages?
Q4) How does the hypervisor in a VM environment affect overall system performance and resource allocation?
Q5) What are the **challenges of scaling applications with Containers** compared to VMs?

## Docker on Linux, Windows and MacOS

### Docker on Linux 
**The chain on Linux**
```
Ubuntu / Alpine / Debian container
              ↓
        Docker engine
              ↓   (namespaces + cgroups)
   Host's real Linux kernel   ← the only kernel anywhere
              ↓
      Physical / cloud hardware
```

![[Docker on Linux.png]]
That's the shortest of the three chains, and it's worth noticing exactly _why_ it's shorter.

### Docker on Windows

There are actually **two distinct chains on Windows**, depending on which mode Docker Desktop is set to — this is different from Mac, which only ever has one path.

![[Docker on Windows.png]]

### Path 1: Linux containers mode (the default, what almost everyone uses)
```
Ubuntu / Alpine container
        ↓
   Docker engine
        ↓
  Hidden Linux VM  (real Linux kernel)
        ↓
  WSL2 or Hyper-V  (Windows's virtualization layer)
        ↓
   Windows kernel
        ↓
 Physical hardware
```

Closer look at WSL2 
```
Linux process (inside WSL2 / docker-desktop distro)
        ↓ real Linux syscalls, unmodified
   Real Linux kernel (Microsoft-built, runs inside the VM)
        ↓ this is the genuine VM boundary
   Hyper-V (lightweight virtualization, "Virtual Machine Platform")
        ↓
   Windows NT kernel (the actual host kernel)
        ↓
   Physical hardware
```

### Path 2: Windows containers mode (opt-in, niche, no VM needed)


```
Nano Server / Server Core container
        ↓
       Docker engine
        ↓
    Windows kernel       ← directly, no VM layer
        ↓
   Physical hardware
```
### Docker on MacOS
![[Docker on MacOS.png]]

#### The chain on macOS
```
Ubuntu / Alpine / Debian container
              ↓
        Docker engine
              ↓
       Hidden Linux VM         ← real Linux kernel, runs here
              ↓
Apple Virtualization.framework  ← Apple's native hypervisor (replaced HyperKit)
              ↓
     Darwin (XNU kernel)       ← the actual Mac OS kernel
              ↓
   Apple Silicon / Intel hardware
```

Compare the three chains side by side:

|Host OS|Steps between container and hardware|Hidden VM required?|
|---|---|---|
|**Linux**|Docker engine → host kernel → hardware|No|
|**Windows** (Linux containers mode)|Docker engine → hidden Linux VM → WSL2/Hyper-V → Windows kernel → hardware|Yes|
|**macOS**|Docker engine → hidden Linux VM → Apple's Virtualization.framework → Darwin kernel → hardware|Yes|




----

Important lines : 
* A container shares the host's kernel — it never ships its own.
* One best practice for containers is that each container should do one thing and do it well.
* **Dockerfile versus Compose file** : A Dockerfile provides instructions to build a container image while a Compose file defines your running containers. Quite often, a Compose file references a Dockerfile to build an image to use for a particular service.


# Image Layers
![[docker image layer 0.png]]
![[docker image layers.png]]
# Container
A container is defined by its **image** as well as any **configuration options** you provide to it when you create or start it.
- containers provide isolated processes for each component of your application


## Overriding container defaults

```console
 docker run -d -p HOST_PORT:CONTAINER_PORT postgres
```

```console
 docker run -e foo=bar postgres env
```

```console
 docker run --env-file .env postgres env
```


```console
 docker run -e POSTGRES_PASSWORD=secret --memory="512m" --cpus="0.5" postgres
```

#### docker run vs docker exec
Unlike `docker run`, which creates a fresh container from scratch, `exec` hooks into an existing one.

This tells Docker: _"I want to execute a brand-new command inside a container that is **already running** right now."_


Run Postgres container in a controlled network
1. Create a new custom network by using the following command:
    ```console
     docker network create mynetwork
    ```
2. Verify the network by running the following command:
    ```console
     docker network ls
    ```
    This command lists all networks, including the newly created "mynetwork".
3. Connect Postgres to the custom network by using the following command:
    ```console
     docker run -d -e POSTGRES_PASSWORD=secret -p 5434:5432 --network mynetwork postgres
    ```
==On a custom network==, containers can resolve each other by name or alias.

 Manage the resources : 
```console
 docker run -d -e POSTGRES_PASSWORD=secret --memory="512m" --cpus=".5" postgres
```
### Docker Volume `--volume` & Bind Mount `--mount`

Docker offers two primary storage options for persisting data and sharing files between the host machine and containers: **volumes** and **bind mounts**.
- **VOLUME**  : If you want to ensure that data generated or modified inside the container persists even after the container stops running, you would opt for a volume.
- **BIND MOUNTS** : If you have specific files or directories on your host system that you want to directly share with your container, like configuration files or development code, then you would use a bind mount. It's like opening a direct portal between your host and container for sharing.
	- Using a bind mount, you can map the configuration file on your host computer to a specific location within the container.

```console
 docker run -d -p 80:80 -v log-data:/logs docker/welcome-to-docker
						   ^^^^^^^^^^^^^^
						-v HOST:CONTAINER
```



```dockerfile
docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=secret_password \
  -v /e/dev/mysql-data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0

===== same as =====

docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=secret_password \
  --mount type=bind,source=/e/dev/mysql-data,target=/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0
```
### Entrypoint & CMD
Docker uses a literal formula to decide what command to execute inside a container:
$$\text{Final Executed Command} = [\text{ENTRYPOINT}] + [\text{CMD or Terminal Arguments}]$$

| **Instruction**  | **Is it locked in?**                                                       | **Purpose**                                                   |
| ---------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **`ENTRYPOINT`** | **Yes.** Hard to override unless you use the explicit `--entrypoint` flag. | Defines the container's primary executable purpose.           |
| **`CMD`**        | **No.** Easily overridden by typing anything at the end of `docker run`.   | Provides default parameters that a user might want to change. |

```Dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/billing-service.jar app.jar

# ENTRYPOINT locks in the Java runtime execution and initial memory flags
ENTRYPOINT ["java", "-Xms512m", "-Xmx1024m", "-jar", "app.jar"]

# CMD provides safe default fallback configurations
CMD ["--spring.profiles.active=local", "--server.port=8080"]
```

### Applying the Formula at Runtime:
- **Scenario A: Standard Boot (No terminal arguments)**
    You just run `docker run billing-service`.
    $$\text{Final Command} = \text{["java", "-Xms512m", "-Xmx1024m", "-jar", "app.jar"]} + \text{["--spring.profiles.active=local", "--server.port=8080"]}$$
    
    **Executes:** `java -Xms512m -Xmx1024m -jar app.jar --spring.profiles.active=local --server.port=8080`
    
- **Scenario B: Deploying to Production (Overriding defaults via terminal)**
    You need it to run in production mode on port `8443`. You pass these properties directly to the terminal command:
    `docker run billing-service --spring.profiles.active=prod --server.port=8443`
    $$\text{Final Command} = \text{["java", "-Xms512m", "-Xmx1024m", "-jar", "app.jar"]} + \text{["--spring.profiles.active=prod", "--server.port=8443"]}$$
    
    **Executes:** `java -Xms512m -Xmx1024m -jar app.jar --spring.profiles.active=prod --server.port=8443`
    
    _(The default `local` profile and `8080` port are completely discarded, but the core JVM sizing flags remain safely intact)._

# Docker compose

not a new technology under the hood — it still uses the same Docker engine, networks, and volumes — it's a **CLI tool plus a YAML spec** that automates exactly the sequence of `docker run`/`network create`/`volume create` commands you'd otherwise type by hand.





**docker image** COMMAND
Manage images
Commands:
  build       Build an image from a Dockerfile
  history     Show the history of an image
  import      Import the contents from a tarball to create a filesystem image
  inspect     Display detailed information on one or more images
  load        Load an image from a tar archive or STDIN
  ls          List images
  prune       Remove unused images
  pull        Download an image from a registry
  push        Upload an image to a registry
  rm          Remove one or more images
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE

**docker container** COMMAND
Manage containers
Commands:
  attach      Attach local standard input, output, and error streams to a running container
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  diff        Inspect changes to files or directories on a container's filesystem
  exec        Execute a command in a running container
  export      Export a container's filesystem as a tar archive
  inspect     Display detailed information on one or more containers
  kill        Kill one or more running containers
  logs        Fetch the logs of a container
  ls          List containers
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  prune       Remove all stopped containers
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  run         Create and run a new container from an image
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  wait        Block until one or more containers stop, then print their exit codes

**docker network** COMMAND
Manage networks
Commands:
  create      Create a network
  connect     Connect a container to a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  prune       Remove all unused networks
  rm          Remove one or more networks

**docker volume** COMMAND
Manage volumes
Commands:
  create      Create a volume
  inspect     Display detailed information on one or more volumes
  ls          List volumes
  prune       Remove unused local volumes
  rm          Remove one or more volumes

**docker compose** `[OPTIONS] COMMAND`
Define and run multi-container applications with Docker
Options:
      --all-resources              Include all resources, even those not
                                   used by services
      --ansi string                Control when to print ANSI control
                                   characters ("never"|"always"|"auto")
                                   (default "auto")
      --compatibility              Run compose in backward compatibility mode
      --dry-run                    Execute command in dry run mode
      --env-file stringArray       Specify an alternate environment file
  -f, --file stringArray           Compose configuration files
      --parallel int               Control max parallelism, -1 for
                                   unlimited (default -1)
      --profile stringArray        Specify a profile to enable
      --progress string            Set type of progress output (auto,
                                   tty, plain, json, quiet)
      --project-directory string   Specify an alternate working directory
                                   (default: the path of the, first
                                   specified, Compose file)
  -p, --project-name string        Project name

Management Commands:
  bridge                  Convert compose files into another model

Commands:
  attach                  Attach local standard input, output, and error streams to a service's running container
  build                   Build or rebuild services
  commit                  Create a new image from a service container's changes
  config                  Parse, resolve and render compose file in canonical format
  cp                      Copy files/folders between a service container and the local filesystem
  create                  Creates containers for a service
  down                    Stop and remove containers, networks
  events                  Receive real time events from containers
  exec                    Execute a command in a running container
  export                  Export a service container's filesystem as a tar archive
  images                  List images used by the created containers
  kill                    Force stop service containers
  logs                    View output from containers
  ls                      List running compose projects
  pause                   Pause services
  port                    Print the public port for a port binding
  ps                      List containers
  publish                 Publish compose application
  pull                    Pull service images
  push                    Push service images
  restart                 Restart service containers
  rm                      Removes stopped service containers
  run                     Run a one-off command on a service
  scale                   Scale services
  start                   Start services
  stats                   Display a live stream of container(s) resource usage statistics
  stop                    Stop services
  top                     Display the running processes
  unpause                 Unpause services
  up                      Create and start containers
  version                 Show the Docker Compose version information
  volumes                 List volumes
  wait                    Block until containers of all (or specified) services stop.
  watch                   Watch build context for service and rebuild/refresh containers when files are updated


# Dockerfile

used to build new images . A Dockerfile typically follows these steps:
1. Determine your base image
2. Install application dependencies
3. Copy in any relevant source code and/or binaries
4. Configure the final image 

```dockerfile
FROM openjdk:11
WORKDIR /usr/src/myapp   # Container fs: cwd is now /usr/src/myapp (created if needed)
COPY . /usr/src/myapp/   # Host fs: take everything in build context (the JAVA-PROJECT folder)
                                # Container fs: place it at /usr/src/myapp/
RUN javac Test.java            # Container fs: runs in /usr/src/myapp (because of WORKDIR)
                                # finds Test.java there because COPY just put it there
CMD ["java","Test"]            # Container fs: runs in /usr/src/myapp at container start
```

Building Image using Dockerfile : 
```
docker build [OPTIONS] PATH | URL | -
                       ^^^^^^^^^^^^^^
```
The `docker build` and `docker buildx build` commands build Docker images from a Dockerfile and a context called **build context**. The build context is the **set of files** that your build can access.

`[OPTIONS]`
- **Dockerfile Search:** If you do not specify a separate path using the `-f` / `--file` flag, Docker automatically looks for a file named `Dockerfile` at the **root of this specified `PATH`**
- **Image Naming:** If you do not specify a name/tag using the `-t` / `--tag` flag, Docker builds the image without any repository or tag, leaving it identifiable only by its image ID (shown as `<none>:<none>` in `docker images`).

```
docker build .
```
The final `.` in the command provides the path or URL to the [build context](https://docs.docker.com/build/concepts/context/#what-is-a-build-context). At this location, the builder will find the `Dockerfile` and other referenced files.

---
### Case 0: Default (baseline, for reference)
**Repo structure:**

```
project/                  ← you run docker build from HERE
├── Dockerfile
├── Test.java
└── Test.class
```

**Command:**

```bash
cd project
docker build -t myapp .
```

Both the Dockerfile _and_ the build context default to the present directory (`.`).

---
### Case 1: Dockerfile is OUTSIDE the present directory (context stays as `.`)

**Repo structure:**
```
project/                  ← you run docker build from HERE (build context)
├── Test.java
└── Test.class

docker-configs/           ← Dockerfile lives elsewhere, sibling folder
└── Dockerfile.prod
```

**Command:**
```bash
cd project
docker build -f ../docker-configs/Dockerfile.prod -t myapp .
```
- `-f ../docker-configs/Dockerfile.prod` → explicitly points to the Dockerfile's real location
- `.` (the `PATH` argument) → build context is still the present directory (`project/`)
- `COPY . /app` inside this Dockerfile would still pull from `project/` (Test.java, Test.class) — Dockerfile's location has no bearing on this

---
### Case 2: Build context is OUTSIDE the present directory (Dockerfile stays as `.`, default lookup)

**Repo structure:**
```
scripts/                  ← you run docker build from HERE
└── (nothing relevant here)

project/                  ← Dockerfile + context both live here, but it's NOT pwd
├── Dockerfile
├── Test.java
└── Test.class
```

**Command:**
```bash
cd scripts
docker build -t myapp ../project
```
- `PATH = ../project` → build context is `project/`, **not** the present directory (`scripts/`)
- No `-f` needed → Docker defaults to looking for `Dockerfile` **inside the context** (`../project/Dockerfile`), which is correct here
- `COPY . /app` would pull from `project/` (Test.java, Test.class)

---
### Case 3: BOTH Dockerfile and build context are outside present directory (and don't even coincide with each other)

**Repo structure:**

```
scripts/                  ← you run docker build from HERE
└── (nothing relevant here)

docker-configs/
└── Dockerfile.prod       ← Dockerfile lives here

project/
├── Test.java             ← build context lives here (separately!)
└── Test.class
```

**Command:**
```bash
cd scripts
docker build -f ../docker-configs/Dockerfile.prod -t myapp ../project
```

- `-f ../docker-configs/Dockerfile.prod` → Dockerfile location, fully independent
- `PATH = ../project` → build context, fully independent
- Neither matches the present directory (`scripts/`), and neither matches each other. 
---

`Usage:  docker buildx build [OPTIONS] PATH | URL | -` 
the `PATH` ==represents the **local directory on your filesystem chosen as the primary build context**==.
When you pass a local path, Docker bundles the files and directories inside that location and sends them to the BuildKit backend daemon. This gives instructions like `COPY` or `ADD` inside your `Dockerfile` the ability to access those files.

**Dockerfile Search:** If you do not specify a separate path using the `-f` / `--file` flag, Docker automatically looks for a file named `Dockerfile` at the **root of this specified `PATH`**

```bash
$ cd my_project
$ docker build [-t IMG_NAME] .
```

```bash
docker build -f /elsewhere/Dockerfile.prod -t project .
```

Common Examples
- **Current Directory:** Uses the current folder as the context and looks for a `Dockerfile` inside it.
```bash
    docker buildx build .
```
- **Specific Subdirectory:** Uses `src/app` as the context folder.
```bash
    docker buildx build ./src/app
```
- **Separate Context & Dockerfile:** Sets the context to the current folder (`.`), but reads a `Dockerfile` located elsewhere.
```bash
    docker buildx build -f ./docker/production.Dockerfile .
```

When neither build context is in the current directory nor the Dockerfile : 

```bash
docker buildx build -f /path/to/my.Dockerfile /path/to/build/context
```

A concrete example : 
Imagine you are currently sitting in your `~/Desktop` directory, but your project files are organized like this:
- Your website files are inside: `/projects/my-web-app/src`
- Your Dockerfile is inside: `/projects/docker-templates/node.Dockerfile`

To run the build from your desktop, you execute:
```bash
docker buildx build -f /projects/docker-templates/node.Dockerfile /projects/my-web-app/src
```
# Multi-stage builds

```dockerfile
# Stage 1: Build Environment
FROM builder-image AS build-stage 
# Install build tools (e.g., Maven, Gradle)
# Copy source code
# Build commands (e.g., compile, package)

# Stage 2: Runtime environment
FROM runtime-image AS final-stage  
#  Copy application artifacts from the build stage (e.g., JAR file)
COPY --from=build-stage /path/in/build/stage /path/to/place/in/final/stage
# Define runtime configuration (e.g., CMD, ENTRYPOINT) 
```



Summary of Differences

| Feature            | `docker run --name`                          | `docker build -t`                              |
| ------------------ | -------------------------------------------- | ---------------------------------------------- |
| **Target Object**  | **Container** (an active, running instance). | **Image** (a static, reusable blueprint).      |
| **Stage**          | **Run stage** (executing an application).    | **Development stage** (packaging code).        |
| **Shorthand**      | None. Must type `--name`.                    | Shorthand for `--tag`.                         |
| **Example Output** | A running container called `my-web-app`.     | A local image file labeled `my-app-image:1.0`. |


Running docker image : 

```bash
docker run [-d] [-p 8080:8080] [-it] IMG_NAME [--name CONTAINER_NAME]
```

Binding ports  : 

**An IP address gets data to the right machine; a port number gets it to the right program on that machine — and because a Docker container has its own private, isolated set of ports (just like it has its own private filesystem and process list, via network namespaces), `-p HOST_PORT:CONTAINER_PORT` is the explicit tunnel you must build to let traffic from a port on your real machine reach a port inside that otherwise-sealed-off container.**

```bash
# Run two nginx containers, both internally listening on 80,
# but exposed on two DIFFERENT host ports since you can't reuse 8080 twice
docker run -d -p 8080:80 --name web1 nginx
docker run -d -p 8081:80 --name web2 nginx
```

-p flag memorization trick 
$$\text{Outside World (Host)} \longrightarrow \text{Inside World (Container)}$$

Publishing ports happens during container creation using the `-p` (or `--publish`) flag with `docker run`. The syntax is:
```console
 docker run -d -p HOST_PORT:CONTAINER_PORT nginx
```
- `HOST_PORT`: The port number on your host machine where you want to receive traffic
- `CONTAINER_PORT`: The port number within the container that's listening for connections

For example, to publish the container's port `80` to host port `8080`:
```console
 docker run -d -p 8080:80 nginx
```
Now, any traffic sent to port `8080` on your host machine will be forwarded to port `80` within the container.