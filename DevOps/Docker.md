
##### Why Docker (Problem before Containerization) ?
"But it works on my machine".

##### What is Docker?
[What is a Container? \| Docker](https://www.docker.com/resources/what-container)
![[Container.png]]

Imp points : 
- Docker is fundamentally a Linux-native technology. It was built to leverage the **monolithic kernel** of Linux.
- Linux allows a process to be "tricked" into thinking it's isolated while still talking directly to the host kernel. This is why Docker on Linux is so fast—there is no "middleman."
- Docker cannot run directly on the Windows kernel (NT Kernel) the same way it runs on Linux. It requires a Linux environment to function because it relies on those specific low-level features we discussed: **Namespaces** and **cgroups**.
- **Docker Desktop** on Windows, it sets up a lightweight Linux bridge for you : 
	- **WSL 2 (Windows Subsystem for Linux):** This is the modern, preferred method. Windows runs a real, highly optimized Linux kernel alongside your Windows kernel. Docker sits on top of that. 
	- **Hyper-V (Legacy):** Before WSL 2, Docker would create a small VM (using the Hypervisor) running a stripped-down Linux OS just to host your containers.


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
