# Complete Guide: pfSense & Tailscale on Proxmox

This guide provides a linear, step-by-step process to deploy pfSense as a Virtual Machine on Proxmox VE, and configure Tailscale for secure remote access. 

**Validated for:** Proxmox VE 8.x | pfSense CE 2.7.x / Plus 23.x | Tailscale 1.5x+

## Prerequisites
* **Proxmox VE** installed and accessible.
* **pfSense ISO** downloaded and uploaded to Proxmox (`local` storage > ISO Images).
* **Tailscale Account** created (Admin Console accessible).

---

## Step 1: Proxmox Network Configuration
Before creating the VM, you need virtual switches (Linux Bridges) to route traffic.

1. In Proxmox, select your Node > **Network**.
2. Identify your existing WAN bridge (usually `vmbr0` connected to your physical NIC).
3. Click **Create > Linux Bridge** to create the isolated LAN for your VMs:
   * **Name:** `vmbr1`
   * **IPv4/CIDR:** Leave blank.
   * **Bridge ports:** Leave blank.
   * **Comment:** `Isolated-LAN`
4. Click **Apply Configuration**.

![Proxmox Network Interfaces showing vmbr0 and vmbr1](assets/01-proxmox-network-bridges.png)
*(Screenshot: Show the Proxmox Network tab with both vmbr0 and vmbr1 visible)*

> **[WARNING]** 
> Never assign your Proxmox management IP (GUI) to a bridge directly exposed to the public Internet without an upstream hardware firewall.

---

## Step 2: Create the pfSense Virtual Machine
1. Click **Create VM** in Proxmox. Name it `pfSense`.
2. **OS:** Select the pfSense ISO. Type: `Other`.
3. **System:** Machine: `q35`, BIOS: `OVMF (UEFI)`.
4. **Disks:** Bus/Device: `VirtIO Block`. Size: `32 GB`. Cache: `Write through`.
5. **CPU / Memory:** `2 Cores` (Type: Host) / `2048 MB`.
6. **Network:** Bridge: `vmbr0` (WAN). Model: `VirtIO`. Uncheck *Firewall*.
7. Click **Finish**.
8. Select the VM > **Hardware** > **Add** > **Network Device**.
    * Bridge: `vmbr1` (LAN).
    * Model: `VirtIO`.
    * Uncheck *Firewall*.

![Proxmox VM Hardware Configuration](assets/02-proxmox-vm-hardware.png)
*(Screenshot: Show the VM Hardware tab highlighting the two Network Devices)*

---

## Step 3: Install pfSense
1. Start the VM and open the **Console**.
2. Boot the installer and select **Auto (ZFS)** for partitioning. 
3. Proceed with the default settings and reboot once finished.

![pfSense ZFS Installation](assets/03-pfsense-zfs-install.png)
*(Screenshot: Show the pfSense installer screen selecting Auto ZFS)*

---

## Step 4: Interface Assignment (Console)
Upon reboot, pfSense will prompt you to assign network interfaces.

1. *VLANs set up?* Enter `n`.
2. *Enter the WAN interface name:* Type `vtnet0`.
3. *Enter the LAN interface name:* Type `vtnet1`.
4. *Proceed?* Type `y`.

![pfSense Console Interface Assignment](assets/04-pfsense-console-interfaces.png)
*(Screenshot: Show the console screen successfully displaying the assigned WAN and LAN IP addresses)*

---

## Step 5: WebGUI Initial Setup
1. Log into the pfSense WebGUI at `https://192.168.1.1` (Default: `admin` / `pfsense`).
2. Follow the Initial Setup Wizard (DNS, Timezone, Admin Password).
3. Navigate to **Firewall > Rules > LAN**.
4. Ensure the `Default allow LAN to any rule` exists.

![pfSense LAN Firewall Rules](assets/05-pfsense-lan-rules.png)
*(Screenshot: Show the Firewall LAN rules page with the default allow rule)*

---

## Step 6: Install & Configure Tailscale

### 6.1 Install the Package
1. Navigate to **System > Package Manager > Available Packages**.
2. Search for `Tailscale` and click **Install**.

![pfSense Package Manager Tailscale](assets/06-pfsense-package-tailscale.png)
*(Screenshot: Show Tailscale successfully installed in the Package Manager)*

### 6.2 Authenticate Node
1. Go to your **Tailscale Admin Console** > **Settings > Keys**.
2. Generate an **Auth key** (Reusable: *No*, Ephemeral: *No*, Pre-authorized: *Yes*).
3. In pfSense, navigate to **VPN > Tailscale > Authentication**.
4. Paste the Key and click **Save**.

![Tailscale Authentication in pfSense](assets/07-tailscale-auth-key.png)
*(Screenshot: Show the pfSense Tailscale authentication page)*

### 6.3 Subnet Routing Configuration
1. In pfSense, go to **VPN > Tailscale > Settings**.
2. Check **Enable Tailscale**.
3. Under *Advertised Routes*, enter your LAN subnet: `192.168.1.0/24`. Click **Save**.

![pfSense Tailscale Advertised Routes](assets/08-pfsense-tailscale-routes.png)
*(Screenshot: Show the Advertised Routes field filled in pfSense)*

4. Return to the **Tailscale Admin Console** > **Machines**.
5. Click the `...` menu on your pfSense node > **Edit route settings**.
6. Toggle on `192.168.1.0/24` to approve the route.

![Tailscale Admin Console Subnet Approval](assets/09-tailscale-console-approve.png)
*(Screenshot: Show the Tailscale web console where you toggle/approve the subnet)*

### 6.4 Firewall Rules for Tailscale
1. In pfSense, go to **Firewall > Rules > Tailscale**.
2. Click **Add**. Set Action to `Pass`, Protocol to `Any`, Source to `Any`, Destination to `LAN net`.
3. Click **Save** and **Apply Changes**.

![Tailscale Firewall Rules](assets/10-tailscale-firewall-rule.png)
*(Screenshot: Show the pass rule on the Tailscale firewall tab)*

---

## Step 7: Validation
1. Disconnect a personal device from your local network.
2. Connect to Tailscale on that device.
3. Access the pfSense WebGUI via `https://192.168.1.1` to confirm your setup is fully operational.
