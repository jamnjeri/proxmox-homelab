# Proxmox Home Lab: Turning an old HP Envy Laptop into a Proxmox Server

## Project Goal: 
- Deploy a stable Proxmox VE 9.x environment on an HP Envy (i7-7500U) using a Realtek RTL8822BE Wi-Fi adapter as the primary management interface 

## Hardware & Requirements
- HP Envy Laptop (or similar to be used as the server)
- USB Flash drive (8GB or larger recommended)
- Another computer to prepare the installer USB
- Internet connection
- Proxmox VE ISO installer
- balenaEtcher


## Installation Steps:
1. Download the Proxmox VE ISO installer from the official Proxmox website.
2. Download and Install balenaEtcher on the computer you will use to create the installer USB.
3. Insert the USB flash drive into the computer.
4. Open balenaEtcher and flash the Proxmox VE ISO onto the USB drive.
5. Once the flashing process is complete, safely eject the USB drive.
6. Insert the USB drive into the HP Envy laptop aka server laptop aka the laptop that will become the Proxmox server.
7. Power on the laptop and enter the BIOS/boot menu (usually by pressing `F9`, `F10` OR `ESC` during startup).
8. Select the USB drive as the boot device.
9. Follow the on-screen Proxmox installer instructions:
    - Accept the license agreement
    - Select the target disk
    - Configure country, time zone, and keyboard layout
    - Set a root password and email address
    - Configure network settings
10. After the installation completes, remove the USB drive and reboot the system.
11. Access the Proxmox web interface from another computer by navigating to:
```
https://<server-ip>:8006
```


## Post-Installation Notes 
1. The "Bootstrap" Problem (No Ethernet)
- Since Proxmox is built for servers, it expects a physical Ethernet connection. The HP Envy lacks this, and the installer does not enable Wi-Fi by default.
-**The solution:** I used an Android device via USB Tethering.
-**Mechanism** Linux treats the phone as a standard USB-Ethernet interface. This provided the initial "bridge" to the internet required to update the system and install necessary drivers.

2. Enabling Virtualization (BIOS)
- The server initially threw errors regarding VMX being disabled.
-**Action:** Entered BIOS (F10) and enabled Intel Virtualization Technology (VT-x).
-**Verification:** Confirmed via the terminal command ```lscpu```, hich showed *Virtualization: VT-x*.

## Major Obstacles & Debugging
1. The "Kernel Panic" (Hardware Lockup)
-**Issue:** The system would freeze (hang) at the LVM/Journal screen during boot.
-**Root Cause:** The Realtek RTL8822BE driver has unstable power-management features (ASPM) that conflict with the Proxmox kernel during the startup sequence.
-**Solution:** Created a kernel module configuration to disable power and stabilize the hardware

```
# File: /etc/modprobe.d/rtw88.conf
options rtw88_pci disable_aspm=Y
options rtw88_8822be disable_lps_deep=Y
```
*By creating this file, the system knows to apply these "High Stability" settings every time the driver loads*

2. "Operation Not Supported" (The Bridge Limitation)
-**Issue:** Standard Proxmox setups use a Linux Bridge (vmbr0) to connect the physical network to Virtual Machines. Most Wi-Fi drivers do not support this Layer-2 bridging.
-**Solution:** Moved to a Routed Architecture.
- -**The Strategy:** The Wi-Fi card (wlo1) remains independent, and the system uses *IP Forwarding* to route traffic to an internal private network for future VMs/Containers.

## Final Network Configuration
- To prevent the "Moving Target" problem (where the server IP changes every reboot), I implemented *DHCP with a Static Lease* at the router level.

1. Router Management
- **Device:** Safaricom Router
- **Process:** Logged into the router, and bound the server's (aka server laptop's) unique MAC address to a reserved IP.
- **Result:** The server always receives it's ip address via DHCP, ensuring a permanent management address without the risk of driver-crashing local static configs.

2. Wi-Fi Credentials Configuration (```/etc/wpa_supplicant/wpa_supplicant.conf```)
- This file essentially tells the Wi-Fi card which network to join and what the password is. Without this, the server knows *how to* talk to the router butdoesn't have the *credentials* to get in. So I manually configured *wpa_supplicant* service. This acts as the "handshake" between the Realtek hardware and the router.
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=KE

network={
    ssid="Your_WiFi_Name"
    psk="Your_WiFi_Password"
}
```


3. Network File (```/etc/network/interfaces```)
```
auto lo
iface lo inet loopback

# Primary Wi-Fi Interface
auto wlo1
iface wlo1 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

# Virtual Bridge for Lab Environment
auto vmbr0
iface vmbr0 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

4. Headless Operation (Lid Close Fix) (```/etc/systemd/logind.conf```)
- To operate the laptop as a dedicated server, I configured `systemd-logind` to ignore the lid switch. This prevents the server from entering "Suspend" mode when the lid is closed, allowing for a compact, headless deployment.
*Edit these lines of code*
```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```
- To make the system recognize the new "Ignore" rule without rebooting, restart the service ```systemctl restart systemd-logind```


## Summary Accomplishments
- [x]**Independent Boot:** No longer requires a phone tether or physical cable.
- [x]**Stable Drivers:** Kernel-level fixes applied to prevent hardware hangs.
- [x]**Static Identity:** Fixed IP at the router level for reliable remote access.
- [x]**Virtualization Ready** BIOS and Kernel verified for KVM/LXC deployment.

## Key Learning Outcomes
- **Linux Networking Fundamentals:** Gained hands-on experience with Layer 2 (Bridging) vs Layer 3 (Routing) and why consumer Wi-Fi drivers require a routed approach. Since my Wi-Fi card refused to share it's desk (Bridging), I turned the server into a Mailroom (Routing) to handle everyone's messages personally.

- **Kernel Tuning:** Learned how to stabilize buggy hardware by passing specific parameters to kernel modules via *modprobe.d*.  I added a 'Stay Awake' sticky note to the Wi-Fi manual so the Engine wouldn't stall trying to save power.

- **System Reliability Engineering (SRE):** Developed a "Bootstrap" methodology using mobile tethering to recover a system without native Ethernet support.

- **Network Persistence:** Implemented DHCP Static Leases to balance driver stability with the need for a fixed server identity.

