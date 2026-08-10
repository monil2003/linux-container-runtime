# Linux Container Runtime

A minimal educational Linux container runtime built directly on top of
Linux kernel primitives such as namespaces, `chroot`, OverlayFS, virtual
Ethernet devices, and iptables.

This project was developed to understand the internals of Linux
containerization by progressively building a userspace container runtime
that creates isolated processes, manages container filesystems, builds
layered images, provides container networking, and allows interaction
with running containers.

The implementation works directly with Linux kernel interfaces and
standard Linux utilities rather than relying on a complete container
runtime such as Docker or Podman.

---

## Motivation

Container abstractions often hide the kernel mechanisms responsible for
process isolation, filesystem isolation, and networking.

This project explores those mechanisms directly by constructing a minimal
container runtime from Linux primitives.

The project focuses on understanding:

- Linux PID namespaces
- UTS namespaces
- Network namespaces
- Mount namespaces
- IPC namespaces
- `chroot`-based filesystem isolation
- `procfs` and `sysfs` inside containers
- OverlayFS and copy-on-write filesystems
- Layered container images
- `RUN` and `COPY` image layers
- Layer caching
- Virtual Ethernet (veth) pairs
- Container network configuration
- iptables-based packet forwarding
- Container-to-container networking
- `nsenter` and execution inside running containers
- Container lifecycle management

---

## Project Structure

```text
Conductor/
├── README.md
├── conductor.sh
├── setup.sh
└── Conductorfiles/
    ├── debian
    ├── ubuntu-1
    └── ubuntu-2
```

Runtime-generated directories are intentionally excluded from the
repository:

```text
.cache/
.images/
.containers/
```

These directories contain downloaded base filesystems, generated image
layers, and running container state.

---

## Container Runtime

The `conductor.sh` script implements the core container runtime.

It provides functionality for:

- Building container images
- Running containers
- Listing running containers
- Executing commands inside running containers
- Creating container networks
- Connecting container networks
- Stopping containers
- Removing images

The runtime combines multiple Linux primitives to construct an isolated
container environment.

---

## Process Isolation

Containers are created using Linux namespaces to isolate resources from
the host.

The runtime creates separate namespaces for:

- PID
- UTS
- Network
- Mount
- IPC

A container's init process becomes PID `1` inside its PID namespace while
remaining an ordinary process in the host's process hierarchy.

For example:

```text
Host
│
├── container process
│   └── PID namespace
│       ├── PID 1  → container init
│       └── PID 2  → child process
│
└── other host processes
```

The UTS namespace also provides an isolated hostname for each container.

---

## Filesystem Isolation

The container filesystem is isolated using `chroot`.

The container receives its own root filesystem containing the required
system binaries and libraries.

Standard kernel filesystems are mounted inside the container:

```text
/
├── proc
├── sys
├── dev
├── bin
├── etc
├── usr
└── var
```

This provides the basic filesystem environment required for normal
process execution inside the container.

---

## Layered Images

The runtime implements container images as a stack of filesystem layers.

A typical image consists of:

```text
Base Debian Layer
        │
        ▼
     RUN Layer
        │
        ▼
    COPY Layer
```

Instead of copying the complete root filesystem for every image and
container, the base filesystem is retained as a shared layer.

The image stores its layer stack in a `layers` file.

For example:

```text
.cache/base/debian-bookworm:
.cache/layers/<layer-hash>/diff:
.cache/layers/<layer-hash>/diff
```

This allows multiple images and containers to share the same underlying
filesystem layers.

---

## OverlayFS

Container filesystems are constructed using Linux OverlayFS.

The filesystem combines read-only image layers with a writable upper
layer:

```text
             Container Root
                   │
                   ▼
              OverlayFS
             /          \
            /            \
     Lower Layers       Upper Layer
          │                 │
     Image contents     Container changes
```

Changes made inside a running container are written to the container's
upper layer rather than modifying the base image.

For example:

```bash
touch /example
```

creates the file in the container's writable layer while leaving the
base image unchanged.

When the container stops, its OverlayFS mount is removed along with the
container-specific writable state.

---

## Image Building

Images are defined using Conductorfiles.

For example:

```dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y htop

COPY ./Conductorfiles/ /
```

The image builder processes the instructions and creates filesystem
layers for each `RUN` and `COPY` operation.

The resulting image can therefore be represented as:

```text
Debian Base
    │
    ▼
RUN apt-get update && apt-get install -y htop
    │
    ▼
COPY ./Conductorfiles/ /
```

---

## Layer Caching

Each `RUN` and `COPY` layer is identified using a SHA-256 based hash.

A layer depends on both its operation and its parent layer.

Conceptually:

```text
Layer Hash =
    Operation
    +
    Parent Layer
    +
    Operation Input
```

For `RUN`, the command contributes to the layer identity.

For `COPY`, the source content contributes to the layer identity.

If an identical layer already exists, the runtime can reuse the cached
layer instead of rebuilding it.

This provides basic content-addressed layer reuse.

---

## Container Networking

Each container receives an isolated network namespace.

Networking is implemented using virtual Ethernet pairs.

The topology is:

```text
Container Network Namespace
          │
          │ veth-inside
          │
          │
          │ veth-outside
          │
          ▼
         Host
```

The inside interface is placed in the container's network namespace,
while the outside interface remains on the host.

Each container is assigned an IP address on its own private subnet.

For example:

```text
Container A                  Container B
192.168.4.2                  192.168.5.2
     │                            │
     │                            │
192.168.4.1                  192.168.5.1
     └────────── Host ────────────┘
```

The container also receives a route through its host-side peer.

---

## Container Peering

The runtime supports connecting two container networks using the
`peer` operation.

The host forwards packets between the two container-facing veth
interfaces using iptables.

Conceptually:

```text
Container A
192.168.4.2
     │
     ▼
veth pair
     │
192.168.4.1
     │
     │
     │ iptables FORWARD
     │
     │
192.168.5.1
     │
     ▼
veth pair
     │
192.168.5.2
Container B
```

The `peer` operation installs forwarding rules in both directions.

---

## Namespace Entry

A running container can be accessed without creating another container
using `nsenter`.

The runtime locates the container's existing process and enters its
namespaces.

For example:

```bash
sudo ./conductor.sh exec test /bin/bash
```

The executed process joins the container's:

- PID namespace
- UTS namespace
- Network namespace
- Mount namespace
- IPC namespace

and uses the container's root filesystem.

This allows tools such as `ps`, `ip`, and `bash` to operate within the
existing container environment.

---

## Container Lifecycle

The runtime supports the following basic lifecycle:

```text
             Build
               │
               ▼
          +----------+
          |   Image  |
          +----+-----+
               │
               ▼
             Run
               │
               ▼
       +---------------+
       |    Running    |
       |   Container   |
       +-------+-------+
               │
        ┌──────┴──────┐
        │             │
       Exec          Stop
        │             │
        │             ▼
        │       Unmount OverlayFS
        │             │
        │             ▼
        │        Remove State
        │
        ▼
  Execute Command
```

The runtime manages:

- Container creation
- Namespace creation
- Root filesystem setup
- OverlayFS mounting
- Network setup
- Process execution
- Namespace entry
- Container shutdown
- Network cleanup
- Filesystem cleanup

---

## Building

Make the scripts executable:

```bash
chmod +x conductor.sh setup.sh
```

Initialize the runtime environment:

```bash
sudo ./setup.sh
```

Build the provided Debian image:

```bash
sudo ./conductor.sh build debian Conductorfiles/debian
```

List available images:

```bash
sudo ./conductor.sh images
```

---

## Running a Container

Start an interactive container:

```bash
sudo ./conductor.sh run debian test /bin/bash
```

Inside the container:

```bash
echo "hello from conductor"
```

Exit with:

```bash
exit
```

A long-running container can be started with:

```bash
sudo ./conductor.sh run debian test sleep infinity
```

List running containers:

```bash
sudo ./conductor.sh ps
```

---

## Executing Commands

Commands can be executed inside an existing container:

```bash
sudo ./conductor.sh exec test /bin/bash
```

For example:

```bash
sudo ./conductor.sh exec test ps
```

The command runs inside the existing container namespaces rather than
creating a new container.

---

## Networking

Create a network for a running container:

```bash
sudo ./conductor.sh addnetwork test
```

Inspect the container network namespace:

```bash
sudo ./conductor.sh exec test /bin/bash
```

Inside the container:

```bash
ip addr
ip route
```

---

## Container Peering

Two running containers can be connected using:

```bash
sudo ./conductor.sh peer container-a container-b
```

This enables forwarding between the two container networks.

---

## Stopping Containers

Stop a running container:

```bash
sudo ./conductor.sh stop test
```

Stopping a container removes its running network state, unmounts the
OverlayFS filesystem, and removes its container-specific writable
filesystem.

---

## Removing Images

Remove an image using:

```bash
sudo ./conductor.sh rmi debian
```

Generated image and container data are kept outside version control.

---

## Requirements

The runtime requires a Linux system with support for the following
kernel features:

- PID namespaces
- UTS namespaces
- Mount namespaces
- Network namespaces
- IPC namespaces
- OverlayFS
- Virtual Ethernet devices
- Linux packet forwarding

Required userspace utilities include:

- Bash
- `ip`
- `iptables`
- `mount`
- `umount`
- `unshare`
- `nsenter`
- `chroot`
- `debootstrap`
- `sha256sum`

Root privileges are required for namespace, filesystem, networking, and
iptables operations.

---

## Learning Outcomes

This project provides hands-on experience with:

- Linux namespaces
- Process isolation
- PID namespace semantics
- `chroot` filesystem isolation
- procfs and sysfs
- Linux OverlayFS
- Copy-on-write filesystems
- Layered container images
- Content-based layer caching
- Container image construction
- Virtual Ethernet networking
- Network namespaces
- Linux routing
- iptables packet forwarding
- Namespace entry using `nsenter`
- Container lifecycle management
- Linux kernel primitives underlying container runtimes

---

## Acknowledgement

This project was developed as an independent learning exercise inspired
by a post-graduate level course assignment on Linux containers and operating
system virtualization.

The course assignment provided the initial progression of concepts
involving Linux namespaces, container filesystems, OverlayFS, container
networking, and process isolation. The work was subsequently organized
and developed as a standalone educational project focused on
understanding the Linux mechanisms underlying container runtimes.

The resulting implementation is maintained as a self-project for
experimentation and learning about Linux container internals.

Course project reference:

https://docs.google.com/document/d/e/2PACX-1vS40zLjy10EsBbc6U8eSlGAQTvyT45oXJ_I7tGeDMxmBeg9crwNId6WCSpTAkb2GCFxuXyxtnX1UaGC/pub
