# Complete Guide: pfSense & Tailscale on Proxmox

This guide provides a linear, step-by-step process to deploy pfSense as a Virtual Machine on Proxmox VE, and configure Tailscale for secure remote access using a single LAN subnet.

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

---

## Step 3: Install pfSense
1. Start the VM and open the **Console**.
2. Boot the installer and select **Auto (ZFS)** for partitioning. 
3. Proceed with the default settings and reboot once finished.

---

## Step 4: Interface Assignment (Console)
Upon reboot, pfSense will prompt you to assign network interfaces.

1. *VLANs set up?* Enter `n`.
2. *Enter the WAN interface name:* Type `vtnet0` and press `Enter`.
   ![Assign WAN](Pasted%20image%2020260807235645.png)

3. *Enter the LAN interface name:* Type `vtnet1` and press `Enter`.
   ![Assign LAN](Pasted%20image%2020260807235714.png)

4. *Enter the Optional 1 interface name:* Press `Enter` to skip. We are building a single-subnet environment, so no optional interfaces are needed.
   ![Skip OPT1](Pasted%20image%2020260807235818.png)

5. *Enter the Optional 2 interface name:* Press `Enter` to skip again.
   ![Skip OPT2](Pasted%20image%2020260807235859.png)

6. The system will display the assigned interfaces. Verify WAN is mapped to `vtnet0` and LAN is mapped to `vtnet1`. *Proceed?* Type `y`.
   
   > **Note:** Depending on your Proxmox VM configuration, you might see extra interfaces like `OPT1` or `OPT2`. If you are only configuring a WAN and a single LAN, you can safely ignore them.
   ![Verify Interfaces](Pasted%20image%2020260807235919.png)

---

## Step 5: Configure LAN IP Address & DHCP (Console)
We will now configure a custom subnet (e.g., `192.168.20.1/24`) and enable the DHCP server for the LAN.

1. From the main pfSense menu, enter option `2` (Set interface(s) IP address).
   ![Main Menu](Pasted%20image%2020260808000032.png)

2. Enter the number of the LAN interface: `2`.
   ![Select LAN Interface](Pasted%20image%2020260808000314.png)

3. *Configure IPv4 address LAN interface via DHCP?* Type `n`.
   ![Disable DHCP on LAN](Pasted%20image%2020260808000330.png)

4. *Enter the new LAN IPv4 address:* Type your desired gateway IP, for example `192.168.20.1`.
   ![Enter LAN IP](Pasted%20image%2020260808000519.png)

5. *Enter the new LAN IPv4 subnet bit count:* Type `24`.
   ![Enter Subnet Mask](Pasted%20image%2020260808000534.png)

6. *For a WAN, enter the new LAN IPv4 upstream gateway address:* Press `Enter` for none.
   ![Upstream Gateway](Pasted%20image%2020260808000554.png)

7. *Configure IPv6 address LAN interface via DHCP6?* Type `n`.
   ![Disable DHCP6](Pasted%20image%2020260808000629.png)

8. *Enter the new LAN IPv6 address:* Press `Enter` for none.
   ![Skip IPv6 Address](Pasted%20image%2020260808000706.png)

9. *Do you want to enable the DHCP server on LAN?* Type `y`.
   ![Enable DHCP Server](Pasted%20image%2020260808000809.png)

10. *Enter the start address of the IPv4 client address range:* Type `192.168.20.100`.
    ![DHCP Start Range](Pasted%20image%2020260808000949.png)

11. *Enter the end address of the IPv4 client address range:* Type `192.168.20.200`.
    ![DHCP End Range](Pasted%20image%2020260808001043.png)

12. *Do you want to revert to HTTP as the webConfigurator protocol?* Type `y`.
    ![Revert to HTTP](Pasted%20image%2020260808001124.png)

---

## Step 6: WebGUI Initial Setup

> **[WARNING] Web Interface Access Limitation**
> By default, pfSense blocks all incoming traffic on the WAN interface for security reasons. You **cannot** access the WebGUI directly from your physical host network or the Internet.
> 
> To access the configuration page:
> 1. You must create a client Virtual Machine (e.g., Windows 10/11, Ubuntu Desktop) in Proxmox.
> 2. Assign this client VM's network adapter to the **`vmbr1` (LAN)** bridge.
> 3. Open a web browser inside this client VM and navigate to the LAN IP.

1. Using your client VM on `vmbr1`, log into the pfSense WebGUI at `http://192.168.20.1`. (Default credentials: `admin` / `pfsense`).
2. Follow the Initial Setup Wizard (DNS, Timezone, Admin Password).
3. Navigate to **Firewall > Rules > LAN** to ensure the default allow rule is present.

---

## Step 7: Install & Configure Tailscale

### 7.1 Install the Package
1. Navigate to **System > Package Manager > Available Packages**.
2. Search for `Tailscale` and click **Install**.

### 7.2 Authenticate Node
1. Go to your **Tailscale Admin Console** > **Settings > Keys**.
2. Generate an **Auth key** (Reusable: *No*, Ephemeral: *No*, Pre-authorized: *Yes*).
3. In pfSense, navigate to **VPN > Tailscale > Authentication**.
4. Paste the Key and click **Save**.

### 7.3 Subnet Routing Configuration
To access your isolated network remotely:

1. In pfSense, go to **VPN > Tailscale > Settings**.
2. Check **Enable Tailscale**.
3. Under *Advertised Routes*, enter your specific LAN subnet: `192.168.20.0/24`.
4. Click **Save**.
5. Return to the **Tailscale Admin Console** > **Machines**.
6. Click the `...` menu on your pfSense node > **Edit route settings**.
7. Toggle on `192.168.20.0/24` under *Subnet routes*.

### 7.4 Firewall Rules for Tailscale
1. In pfSense, go to **Firewall > Rules > Tailscale**.
2. Click **Add**. Set Action to `Pass`, Protocol to `Any`, Source to `Any`, Destination to `LAN net`.
3. Click **Save** and **Apply Changes**.

---

## Step 8: Validation
1. Disconnect a personal device from your local network.
2. Connect to Tailscale on that device.
3. Access the pfSense WebGUI via `http://192.168.20.1` to confirm your setup is fully operational.
