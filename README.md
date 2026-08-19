# AosEdge Virtual Unit Orchestrator

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Linux-lightgrey.svg)](https://www.kernel.org/)
[![Architecture](https://img.shields.io/badge/arch-x86__64%20%7C%20aarch64-green.svg)](https://www.qemu.org/)

## Overview

AosEdge Unit is a systemd-integrated VM management system that orchestrates multiple QEMU/KVM virtual machines with deterministic networking, health monitoring, and automatic resource management.

### Key Features

- **Automated VM Orchestration** – Seamlessly launch and manage a fleet of AosCore emulation VMs from a single YAML-based configuration.  
- **Production-Oriented Design** – Features structured logging (journald + file-based logs), explicit startup/cleanup phases, and deterministic exit codes to distinguish between configuration errors and runtime failures.  
- **Minimal Runtime Dependencies** – Built as a lightweight Bash + Gawk engine. It avoids heavy runtimes like Python, Go, or NodeJS, relying instead on standard system utilities (qemu, nftables, iproute2).

## Quick Start

For complete configuration syntax, refer to the manual pages: **man aos-unit.conf(5)** and **man aos-unit(8)**.

1. **Install the Package:**

   ```
   sudo add-apt-repository ppa:aosedge/aos-unit
   sudo apt update
   sudo apt install aos-unit
   ```
3. **Prepare VM Images:** Get AosEdge unit images and place them in the system-managed state directory.  
     Download from official AosCore VM release (v5.2.2)
   ```
   wget https://github.com/aosedge/meta-aos-vm/releases/download/v5.2.2/aos-vm-image-qemux86-64-5.2.2.tar.xz
   ```
     Extract the archive
   ```
   tar -xf aos-vm-image-qemux86-64-5.2.2.tar.xz
   ```
     Install to system state directory
   ```
   sudo cp aos-vm-main-qemux86-64.qcow2 aos-vm-secondary-qemux86-64.qcow2 /var/lib/aos-unit/
   sudo chown aos-unit:aos-unit /var/lib/aos-unit/*.qcow2
   ```
5. **Check Configured Nodes:** Take a look at `/etc/aos-unit/unit_config.yaml` to check the supplied sample cluster layout. It defines 2 nodes that will run on `aos-vm-main-qemux86-64.qcow2` and `aos-vm-secondary-qemux86-64.qcow2` images you have just downloaded. Network is configured with static IP to the __main__ node and dynamic IP to the __secondary__ node. Note that static IP forthe main node is 10.0.0.100 - so let's configure system's DHCP pool to match it. Add `DHCP_CIDR=10.0.0.0/24` to `/etc/aos-unit/runtime.conf` using some `sed` regexp magic:
   ```
   sudo sed -i 's|^#\s*DHCP_CIDR=.*|DHCP_CIDR="10.0.0.0/24"|' /etc/aos-unit/runtime.conf
   ```
6. **Launch the Unit:**
   ```
   sudo systemctl start aos-unit
   ```
7. **Access Your Nodes:** Thanks to the DNS Sidecar, VMs are immediately accessible by their YAML names with SSH (use `Password1`). For the main node, use:
   ```
   ssh root@main.aos-unit
   ```
     and for accessing secondary unit, use:
   ```
   ssh root@secondary.aos-unit
   ```
Congratulations! You have successfully deployed your virtual AosEdge Unit. Now you need to provision it and it will be ready to run your edge services!  

## Architecture & Core Principles

The architecture favors determinism, explicit failure semantics, and controlled recovery over opaque background behavior.

* **Systemd-Native Lifecycle:** VMs are not mere background processes; each node runs as an instance of the packaged template unit aos-unit-node@\<name\>.service, allowing for native monitoring, auto-restarts, and dependency management. Because the unit names are static, restarting the manager is an ordinary systemd restart rather than a race to re-create unit names.
* **Security-Focused Execution:** Operates as a dedicated, unprivileged aos-unit system user. Elevated network operations are handled strictly within the systemd unit via minimal Linux capabilities and specific PolicyKit rules.  
* **Resource Management:** Per-VM CPU and memory limits from unit_config.yaml are enforced via cgroups v2, applied to each node instance at start time with systemctl set-property. All VMs are grouped under aos-unit.slice for hierarchical control over the emulation cluster's total footprint.  
* **Portable Virtualization:** Defaults to KVM hardware acceleration for near-native performance. If /dev/kvm is unavailable, it gracefully falls back to QEMU software emulation (TCG) to enable cross-architecture testing.

### **FHS-Aligned Layout**

The orchestrator strictly adheres to the Linux Filesystem Hierarchy Standard (FHS). Paths are not arbitrarily configured by the user; they are securely provisioned and injected directly by systemd:

* **Configuration:** /etc/aos-unit/  
* **Images:** /var/lib/aos-unit/  
* **Logs:** /var/log/aos-unit/  
* **Runtime:** /run/aos-unit/

## Service Hierarchy & Component Flow

AosEdge Unit is implemented as a layered set of systemd units and focused helper scripts.

```
systemd (host)
├── aos-unit.service                     # Main manager (entry point)
│   ├── ExecStartPre: service-startup    # Bridge + nftables base + resolved routing
│   ├── ExecStart: runner                # Parse YAML, start node instances, monitor routes
│   ├── (runtime): route-monitor         # Watches uplink/default-route changes via netlink
│   ├── (runtime): network-helper        # Applies NAT rules; updated on route changes
│   └── ExecStopPost: service-cleanup    # Teardown bridge, nftables
│
├── aos-unit-dnsmasq.service             # Transient unit (DHCP + DNS on alt port)
│   └── dnsmasq-lease-hook               # DHCP lease event hook
│
├── aos-unit-vm-failed@<name>.service    # Triggered on VM node <name> failure
│   └── vm-failed-handler                # Appends forensic data to persistent logs
│
└── aos-unit.slice                       # VM-only resource hierarchy
    ├── aos-unit-node@main.service       # Instance of the aos-unit-node@ template
    │   ├── ExecStartPre: network-helper  # Register tap + DHCP reservation
    │   ├── ExecStart: vm-launch          # Builds argv, execs qemu-system-* (KVM or TCG)
    │   ├── ExecStop: vm-shutdown         # Send ACPI power-down via QMP
    │   └── ExecStopPost: network-helper  # Unregister tap + DHCP reservation
    └── aos-unit-node@secondary.service  # Instance of the aos-unit-node@ template
```

## **Dynamic Networking & "Smart NAT"**

Networking is not static. The host's default route may change (e.g., Ethernet ↔ Wi-Fi). AosEdge Unit implements an adaptive, kernel-aware "Smart NAT" system to handle this seamlessly.

* **Atomic Handover:** A background monitor (`route-monitor`) listens for kernel routing events (e.g., RTM_NEWROUTE) via netlink. When the default route changes, NAT rules are dynamically recalculated with debouncing (waits for changes to settle), and nftables mappings are updated to preserve VM outbound connectivity.  
* **Deterministic Bridge Naming:** The virtual bridge name is automatically derived from the host's machine-id (`aosbr{machine-id:0:6}`) to ensure uniqueness without conflicts while remaining deterministic across reboots.
* **Automatic Network Derivation:** Network configuration (bridge IP, DHCP range, netmask) is automatically calculated from a single CIDR parameter (default: 10.200.1.0/24), eliminating manual subnet management and reducing configuration errors.
* **Host DNS integration:** Host-side resolution of VM names relies on systemd-resolved (resolvectl) being available and running.
* **DNS Sidecar Architecture:** To avoid conflicts with systemd-resolved, the internal dnsmasq server binds to an alternate port (default 5300). Local nftables rules transparently redirect VM bridge traffic from port 53 to 5300. 
* **Deterministic Networking:** Integrated DHCP supports both static reservations and dynamic pools. MAC addresses are hashed from node names to ensure persistent network identities.

## Fail-Fast Logic & Exit Semantics

AosEdge Unit deliberately separates VM-level failures from orchestrator-level failures and configuration errors.

* **VM-Level Faults:** If a guest OS crashes, systemd detects the unit failure. Restart behavior is governed strictly by the node unit's systemd policy, preventing cascading teardowns of the entire environment.  
* **Orchestrator Exit Codes (aos-unit.service):**  
  * 0: clean shutdown.  
  * 1: runtime failure (Transient errors, systemd will automatically restart).  
  * 2: configuration error (Permanent failures like missing YAML or invalid IPs. Systemd will **not** attempt restarts to prevent infinite loops).

### **Observability & Troubleshooting**

* **Check Entire Cluster Status:**  
  `systemctl list-units 'aos-unit-node@*'`
* **Follow Aggregated Logs:**  
  `journalctl -u 'aos-*' -f`
* **Live Serial Console:** Connect to the live virtual UART without interrupting persistent logging:  
  `sudo socat -,raw,echo=0,escape=0x1d UNIX-CONNECT:/run/aos-unit/main.serial`
* **DNS Resolution Issues:**
  * **Host-to-VM:** Check if the host correctly routes the `~aos-unit` domain to the bridge: run `resolvectl status` and look for the dynamically-named bridge.
  * **NFTables Redirect:** Verify the `nftables` rules are successfully catching traffic on port 53 and redirecting it to the `dnsmasq` sidecar: run `sudo nft list chain inet aos_unit prerouting`.
* **DHCP & IP Assignment:**
  * If a VM has no IP address, verify the active DHCP leases managed by the sidecar: run `cat /run/aos-unit/dnsmasq.leases`.
  * Inspect the sidecar logs for DHCP handshake errors: run `journalctl -u aos-unit-dnsmasq | grep DHCP`.
* **NAT & Internet Access:**
  * If a VM cannot reach the internet, verify the host's default route is correctly detected: run `ip route show default`.
  * Ensure the `Smart NAT` masquerade and forward rules are actively applied: run `sudo nft list chain inet aos_unit postrouting` and `sudo nft list chain inet aos_unit forward`.
* **Bridge Discovery:**
  * To find the automatically-created bridge name: run `ip link show | grep aosbr` or check the runtime config at `/run/aos-unit/network.conf`.

## **Build & Release System**

AosEdge Unit supports two distinct build modes driven by a strict "Git tag = release source of truth" philosophy.

| Aspect | Local Build | PPA Build |
| :---- | :---- | :---- |
| **Purpose** | Development / testing | Public distribution |
| **Signing** | No source upload signing | GPG-signed source upload (required by Launchpad) |
| **Version Scheme** | Build info-based {base}+git{short_sha}+{timestamp} | Tag-based {base}~{series} |
| **Changelog** | Generated automatically | Extracted from git tag (mandatory) |
| **Output** | Binary .deb | Signed source package (.dsc, .changes, tarballs) |

### **PPA CI Integration (GitHub Actions)**

Releases are heavily guarded. The CI workflow (ppa-publish.yml) triggers on versioned tags (v\*.\*.\*). It strictly verifies the GPG signature of the tag against an allowed list of public keys (TAG_SIGNING_PUBKEYS) before importing the Launchpad signing key to build and upload the source packages. This completely prevents unauthorized releases from entering the PPA. For more information check out [release documentation](RELEASE.md).

## **Documentation & Built-in Help**

AosEdge Unit is designed to be self-documenting. Instead of relying purely on this README, use the native system manuals and tool-level help for authoritative, up-to-date information:

* **The Configuration Manual:** `man 5 aos-unit.conf` contains the strict definitions, defaults, and exact syntax for every variable allowed in `/etc/aos-unit/runtime.conf` and the networking setup.  
* **The Operational Manual:** `man 8 aos-unit` covers the system integration architecture, FHS paths, live console access via socat, and the strict exit code semantics.  
* **Build System Reference:** if you are contributing to the project or compiling from source, the build script contains detailed usage instructions. Run `./build_package.sh --help` to see all available flags for local testing and PPA release modes.
