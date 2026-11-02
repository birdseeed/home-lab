# Home lab

I have run some form of home lab for most of my career, starting with HP MicroServers that had iLO for out of band management, then a Windows host running Hyper-V virtual machines, and now an HP ProDesk 600 G1 Desktop Mini running Proxmox in a wall mounted rack. It is where I try things before I trust them, and through 2026 it was also the build and test environment for Patched, the products in my [endpoint-assurance](https://github.com/birdseeed/endpoint-assurance) and [macos-updater](https://github.com/birdseeed/macos-updater) repositories.

![Home lab network](assets/network-overview.svg)

## The network

The network is built on UniFi: a UCG Ultra as the gateway and firewall, a USW Lite 8 PoE switch feeding two U7 Lite access points, and a Netgear M4100-26G 24 port managed switch in the rack for wired kit. It is split into five VLANs, for trusted devices and servers, smart home devices, guests, CI runners and AI agent workloads, and every one except the main network is isolated, so it can reach the internet but not the home network or the other VLANs. Where something genuinely needs a path across, such as SSH from my Mac to the CI runner, it gets a firewall rule for that one port rather than a broad exception. The [network notes](docs/network-topology.md) have the detail.

## What runs on it

The Proxmox host runs three virtual machines, each with a single job:

| Virtual machine | What it does |
| --- | --- |
| Storage | A pool of about 28 TB shared over SMB, plus Docker services including media and the dashboards displayed around the house |
| Home Assistant | Home automation |
| CI runner | A self hosted, isolated GitHub Actions runner for my own repositories |

The storage is four mismatched disks, a 12 TB Seagate IronWolf, a 10 TB Toshiba, a 3 TB Seagate and a 3 TB WD Red, in an external USB C enclosure. Proxmox passes the physical disks straight through to the Storage VM, which mounts each one and combines them into a single pool with mergerfs. Before any service that depends on the pool starts, a mount guard checks it is looking at the right disk and partition, so a disk that comes back under a different device name cannot quietly leave a service writing into an empty folder.

The pool has no redundancy, and that is a deliberate trade off rather than an oversight. It holds only media that can all be downloaded again, so losing a disk costs me time rather than data, and I would rather spend the capacity than mirror something replaceable. Everything that is not replaceable lives elsewhere: the virtual machines' operating system disks, the Docker services and the dashboards run from the host's internal SSD, and their configuration is kept in GitHub, so the host can be rebuilt without the pool. The USB C enclosure was chosen for the same reason: it is the cheapest way to add that much capacity to a micro PC, and for this data a disk dropping off the bus is an inconvenience, not a loss. The passed through disks are excluded from the VM backups for the same reason.

## Moving off Windows

In October 2026 I moved the whole setup from a Windows host, where the disks were pooled with StableBit DrivePool and the other services ran under Hyper-V, onto Proxmox. Rather than copying everything off and reformatting, I kept the existing NTFS filesystems and their data in place and built the mergerfs pool over the top of them. I kept the original Windows backups, application exports and cold exports of the old virtual machines through the migration, and added the fourth disk to the pool once the first three were settled.

## A hardware fault

After the move the host kept losing its network connection while staying up itself. The first problem was simple: the link had negotiated at 100 Mb/s, and replacing the cable brought it back to 1 Gb/s full duplex. The drops carried on, though, and the kernel log showed repeated hardware unit hangs from the Intel I217-LM's e1000e driver, which is a known weakness of that NIC under load with hardware offloads enabled. Turning the offloads off stopped the hangs. I added the change to the interface configuration so it is applied when the bridge comes up, tested transfers and the services that depend on the pool afterwards, and then rebooted the host to confirm the fix survives a restart.

## CI runners

A GitHub Actions job runs whatever code is in the workflow, so the Linux runner lives on its own VLAN, tagged through to the Proxmox host, and I tested from inside it that the household and management networks could not be reached. Running my own runners also cut the cost of GitHub hosted minutes.

For Patched I ran two kinds of self hosted runner. The Linux runner on Proxmox handled the service and licensing checks, and a persistent macOS ARM64 virtual machine on my M4 MacBook Pro, run under Parallels with four virtual CPUs, 8 GB of memory and a 100 GB disk, handled the native Swift and browser checks that need real macOS.

Labels alone are not enough to make sure a job lands on the right machine, so every job checked the exact runner name, operating system and architecture before checking out any code, refused to run as root, and recorded the runner and commit it ran against. Workflows only ran for pushes and pull requests from the repository itself, never from forks, and checkout did not leave credentials on the runner afterwards. For the riskier native tests I added a disposable macOS guest that was checked for the expected account, OS version, architecture and identity before any work ran, so a test could not silently run against the wrong machine.

## Local AI

I run local models on my M4 MacBook Pro, using Ollama to serve Qwen. I use it to compare running models locally with paying per token for hosted ones, and to keep work that should not leave the house on my own hardware.

## Looking after it

When I moved workloads between hosts I checked the storage mounts, scheduled timers and application health before and after, kept queues, configuration and data intact, and confirmed the backups were readable before switching anything over. I reuse hardware where I can, and I keep test environments separate from the Mac I use every day.
