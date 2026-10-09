# Local SOC Sandbox Environment Setup

> **AI disclosure:** AI assistance was used for **documentation only**: structuring and writing this README from the author's screenshots and notes. The lab design, configuration, installation, and all testing were performed manually by the author.

### Overview

This is a home Security Operations Center (SOC) lab built on a single Proxmox VE host. The lab contains a **pfSense** firewall/router VM (VM ID 100) that sits between the host's upstream network and the lab. The goal is hands-on practice with network segmentation, log collection, and alert triage.

### Lab Architecture

| Component | Details |
|-----------|---------|
| Hypervisor | Proxmox Virtual Environment **9.2.2**, node name `admin` |
| Firewall VM | **pfSense**, VM ID `100`, name `PFSense` |
| Host management network | Linux bridge `vmbr0` on physical NIC `nic0` (Active, Autostart) |
| Storage | `local` (ISO images and templates), `local-lvm` (VM disks) |

### Virtual Machines

#### VM 100: PFSense

| Setting | Value |
|---------|-------|
| VM ID / Name | 100 / PFSense |
| Node | admin |
| Memory | 1024 MiB (1.00 GiB) |
| Processors | 1 socket, 1 core, type `x86-64-v2-AES` |
| BIOS / Machine | SeaBIOS (Default) / i440fx (Default) |
| SCSI controller | VirtIO SCSI single |
| Disk | `local-lvm:vm-100-disk-0`, 32 GB |
| CD/DVD | `local:iso/netgate-installer-v1.2-RELEASE-amd64.iso` |
| Guest OS type | Other |
| Network device | Intel E1000, bridge `vmbr0`, VLAN tag `1`, firewall flag enabled |

### Software Used

| Software | Version | Purpose |
|----------|---------|---------|
| Proxmox Virtual Environment | 9.2.2 | Hypervisor and VM management |
| pfSense (Netgate Installer) | v1.2-RELEASE (amd64 ISO, about 1010 MiB) | Firewall/router VM |

### Setup Steps

#### 1. Download the pfSense installer ISO

- In Proxmox, open **admin > local > ISO Images** and choose **Download from URL**.
- The ISO was downloaded from the Netgate download server (`shop.netgate.com`). The task log records a file of 342,781,760 bytes, decompressed to `netgate-installer-v1.2-RELEASE-amd64.iso`.
- The task completed successfully (**TASK OK**, 19.4 MB/s average, 2026-09-15 18:03:43).
- The ISO appears in the `local` ISO list.

#### 2. Review the host bridge (node network)

- Go to **admin > System > Network**. The **Create** menu offers Linux Bridge, Linux Bond, Linux VLAN, OVS Bridge, OVS Bond, and OVS IntPort.
- The host's existing network is a **Linux Bridge** (Active: Yes, Autostart: Yes) with port `nic0`. The VM's NIC is attached to `vmbr0`.

#### 3. Create the VM

1. **Create VM** > **General**: Node `admin`, VM ID `100`, Name `PFSense`. "Add to HA" left unchecked.
2. **OS**: Use CD/DVD disc image (ISO) from storage `local`, image `netgate-installer-v1.2-RELEASE-amd64.iso`. Guest OS Type **Other**, Version `-`.
3. **System**: Default settings (SeaBIOS, i440fx, VirtIO SCSI single).
4. **Disks**: 32 GB disk on `local-lvm`.
5. **CPU**: 1 socket, 1 core, `x86-64-v2-AES`.
6. **Memory**: **1024 MiB**.
7. **Network**: Intel E1000 network device on bridge **Internal** (as shown in the Add Network Device dialog), VLAN Tag **no VLAN**, Firewall unchecked, MAC auto.
8. **Confirm**, then create the VM.

After creation, the **Hardware** tab shows the NIC as `e1000=<MAC>,bridge=vmbr0,firewall=1,tag=1`. The saved configuration uses VLAN tag `1` on `vmbr0`, which differs from the "no VLAN" option in the wizard. The Hardware tab is the authoritative record of the NIC settings.

#### 4. Install pfSense

1. Start VM 100 and open **Console**.
2. The Netgate Installer **Welcome** screen offers **Install pfSense**, **Rescue Shell**, **Advanced Options**, and **Cancel**. Choose **Install pfSense**.
3. On the **WAN (em0) Network Mode Setup** screen, the following settings were used:
   - Interface Mode: **DHCP (client)**
   - VLAN Settings: **VLAN Tagging disabled**
   - Use local resolver: **false**
   - Selected: **Continue**

The WAN interface is `em0`, the FreeBSD driver name for the E1000 virtual NIC.

### Screenshots

| # | Screenshot | What it shows |
|---|------------|---------------|
| 1 | Create VM, Memory tab | Memory set to 1024 MiB. Background shows the `local` ISO list with `netgate-installer-v1.2-RELEASE-amd64.iso` (1010.17 MiB). |
| 2 | Add Network Device dialog | Bridge "Internal", model Intel E1000, VLAN Tag "no VLAN", MAC "auto", Firewall unchecked. |
| 3 | VM 100 Hardware tab, Add menu open | Hardware list with 1 GiB memory, 1 core CPU, 32 GB disk, ISO on the CD/DVD drive, and the E1000 NIC on `vmbr0` with tag 1. The Add menu offers Hard Disk, Network Device, CD/DVD, and others. |
| 4 | Node `admin` > System > Network | Network list with the Linux Bridge (Active, Autostart, port `nic0`) and the Create menu open. |
| 5 | Create VM, General tab | Node `admin`, VM ID `100`, Name field, HA unchecked. |
| 6 | Create VM, OS tab | ISO selected from storage `local`, Guest OS type Other. |
| 7 | pfSense installer, Welcome screen | Netgate Installer v1.2-RELEASE welcome dialog with "Install pfSense" highlighted. |
| 8 | pfSense installer, WAN network mode | WAN (em0) set to DHCP (client), VLAN tagging disabled, Continue highlighted. |
| 9 | Proxmox task viewer | ISO download log from `shop.netgate.com`, 342,781,760 bytes, **TASK OK**. |
| 10 | Storage `local` > ISO Images | ISO `netgate-installer-v1.2-RELEASE-amd64.iso` listed (1010.17 MiB, format `iso`). |

### Issues Encountered

- **Failed "Update package database" tasks on the Proxmox node.** Two `apt-get update` tasks failed, on 2026-09-14 at 21:54 and 2026-09-15 at 03:55.

### Lessons Learned

- **Check the NIC tag after creating a VM.** The wizard offered "no VLAN", but the saved NIC has `tag=1`. The Hardware tab shows the final configuration.
- **Verify ISO checksums.** The download task log records the transfer. Compare the ISO against Netgate's published hash before installing.

### Next Steps

- Finish the pfSense installation and assign the LAN interface (second E1000 NIC or VLAN).
- Define lab network segments and firewall rules in pfSense.
- Deploy the SIEM/log collection stack and forward logs from the lab VMs.
