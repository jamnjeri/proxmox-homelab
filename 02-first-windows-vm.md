# Phase 2: Virtual Machine Deployment & Advanced Routed Networking

## Project Goal:
Deploy a high-performance **Windows 10 Virtual Machine** for Power BI development on a Proxmox host constrained by Wi-Fi hardware (Realtek RTL8822BE), using a **Layer 3 Routed NAT architecture** instead of traditional bridging.


## Architecture Overview - Why I couldn't Bridge
A standard Proxmox setup uses a Linux bridge (`vmbr0`) to allow VMs to appear directly on the physical network.
**The Problem:** Most Wi-Fi drivers (including the Realtek RTL series) refuse to forward traffic for *foreign MAC addresses* (i.e., VM network identities). Therefore the VM cannot communicate properly when attached to a bridged Wi-Fi interface.
**The Solution:** I implemented a **Layer 3 Routed NAT architecture**:
- The Proxmox host becomes a gateway/router
- The VM lives on an internal private subnet (`10.10.10.0/24`)
- The host performs NAT (masquerading) over `wlo1`
- The outside network only sees the host


## Step 1: Preparing the ISOs
Before creating the VM, you must provide Proxmox with the operating system and the drivers that allow Windows to talk to virtual hardware efficiently.
1. **Windows 10 ISO:** Download from the Official Microsoft Media Creation Tool.
2. **VirtIO Drivers ISO:** Since Windows doesn't natively support KVM hardware, you need these drivers from the Fedora project.
- Source: VirtIO-Win ISO (Stable)

3. **Upload to Proxmox:**
- Navigate to your Proxmox Web GUI → local storage → ISO Images → Upload.
- Upload both the Windows 10 and VirtIO ISOs.

## Step 2: Host-Side Network Configuration
I transformed the Proxmox host into a functional router for the internal VM network. 

### 1. Configure the Internal Bridge
Edit (`/etc/network/interfaces`) to define vmbr0 as a static internal gateway.

```bash
auto vmbr0
iface vmbr0 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0

```
**Note:** `bridge-ports none` is critical. It tells Proxmox not to look for a physical Ethernet cable.


### 2. Enable IP Forwarding
The Linux kernel must be told to allow packets to travel between interfaces.

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

### 3. Implement NAT (Masquerading) via iptables
I allowed the VM to share the host’s Wi-Fi connection:

```
# Apply NAT
iptables -t nat -A POSTROUTING -o wlo1 -j MASQUERADE

# Allow forwarding
iptables -A FORWARD -i vmbr0 -o wlo1 -j ACCEPT
iptables -A FORWARD -i wlo1 -o vmbr0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Make rules persistent (survive reboots)
apt install iptables-persistent
netfilter-persistent save
```

## Step 3: Windows 10 VM Deployment & Optimization
I created a Windows 10 VM (ID 101) with the following configurations and optimizations:

### 1. VM Hardware Configuration
- **CPU**: 2-4 Cores (Type: host for maximum instruction set passthrough)
- **RAM**: 4GB+ (Ballooning enabled)
- **Disk**: VirtIO Block (For superior I/O speed)
- **Network**: VirtIO (Paravirtualized)

### 2. The "Code 43" Display Issue
Windows often fails to recognize virtual GPUs, defaulting to *Microsoft Basic Display Adapter*, which results in poor resolution and lag.
- **The Fix:** Manual driver injection aka mount the VirtIO Guest Tools ISO 
- **Path:** Install driver from: ```D:\viogpu\w10\amd64``` (on the VirtIO Guest Tools ISO).
- **The Proxmox Tweak:** Set the Hardware Display to VirtIO-GPU with at least 512MB memory.

### 3. Windows OS Optimization
To ensure Power BI runs smoothly on a 2-core VM
- **Visuals:** Adjusted for "Best Performance" (Disables animations).
- **Background Apps:** Disabled all background telemetry.
- **Power Plan:** Set to "High Performance."


## Major Obstacles & Debugging
### 1. The "Firewall Ghost" (Silent Dropped Packets)
- **Issue:** Even with correct NAT rules, the VM could not ping the gateway (10.10.10.1).
- **Discovery:** Proxmox creates a firewall bridge chain (fwbr, fwln) by default.
- **Solution:** Uncheck the "Firewall" box in the VM's Network Device settings. This bypassed the virtual security gate and restored connectivity immediately.

### 2. Realtek "Deep Sleep"
- **Issue:** VM internet would drop intermittently while the Host stayed online.
- **Root Cause:** Realtek aggressive power management on the Wi-Fi card.
- **Fix** Run: ```iw dev wlo1 set power_save off```

## Final VM Network Configuration
Inside the Windows VM, I bypassed DHCP (as no DHCP server exists on vmbr0) and used a static assignment:
- **IP Address:** ```10.10.10.100```
- **Subnet Mask:** ```255.255.255.0```
- **Default Gateway:** ```10.10.10.1``` (The Host)
- **DNS:** ```8.8.8.8``` | ```1.1.1.1```

## Summary Accomplishments
- [x]**Routed Architecture:** Bypassed Layer-2 Wi-Fi limitations.
- [x]**Persistence:** NAT and IP Forwarding survive system reboots.
- [x]**High Resolution:** Forced VirtIO drivers for 1920x1080 desktop experience.
- [x]**App Ready:** Power BI installed and verified on paravirtualized hardware.

## Key Learning Outcomes
- **The OSI Model in Practice:** I learned that just because a *Bridge* (Layer 2) exists doesn't mean the hardware supports it. Routing (Layer 3) is the professional workaround for hardware constraints.
- **Proxmox Virtualization Stack:** I gained introductory experience with qemu-server, virtio drivers, and the noVNC console environment.


#### Check out [How to make a Windows 10 VM in Proxmox in under 10 Minutes](https://www.youtube.com/watch?v=-KTTmrA3DX8) for a great visual walkthrough of the ISO mounting and "Load Driver" process
