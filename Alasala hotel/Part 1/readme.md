---

## 🏗️ Architecture & Specifications

### On-Premises Simulated Environment

| Machine Name | Role / Tier | Operating System | IP Address | Installed Services / Software |
| :--- | :--- | :--- | :--- | :--- |
| **SmartHotelHost** | Parent Hyper-V Host | Windows Server 2019/2022 | `10.0.0.4` (Azure VNet) | Hyper-V Role, NAT Gateway (`192.168.10.1`) |
| **UbuntuWAF** | Reverse Proxy / WAF | Ubuntu 18.04/20.04 LTS | `192.168.10.10` | NGINX Reverse Proxy & Load Balancer (Port 80) |
| **SmartHotelWeb1** | Web Tier (Frontend) | Windows Server 2012 R2 | `192.168.10.11` | IIS, ASP.NET Frontend Application UI |
| **SmartHotelWeb2** | App Tier (Backend) | Windows Server 2012 R2 | `192.168.10.12` | IIS, ASP.NET Web API / Services |
| **SmartHotelSQL1** | Database Tier | Windows Server 2016 | `192.168.10.20` | SQL Server 2017 (Dev/Express), Port 1433 |

---

### Azure Target Landing Zone (`SmartHotel-RGRG`)

| Resource Name | Resource Type | Architectural Purpose |
| :--- | :--- | :--- |
| `SmartHotelVNet` | Virtual Network | Target Cloud Network (`10.1.0.0/16`) for App, WAF & Bastion subnets |
| `SmartHotel-WAF` | Application Gateway (WAF v2) | Cloud-native Layer-7 load balancing & security (replaces UbuntuWAF) |
| `SmartHotel-WAF-ip` | Public IP Address | External entry point for Application Gateway |
| `smarthoteldb` | Azure SQL Database | Target PaaS database (replaces SmartHotelSQL1) |
| `smarthotelsqlggjxc` | SQL Logical Server | Host for the `smarthoteldb` PaaS instance in East US |
| `SmartHotelBastion` | Azure Bastion | Secure RDP/SSH portal access to migrated VMs without public IPs |
| `SmartHotelBastion-ip` | Public IP Address | Public IP dedicated to Azure Bastion |
| `SmartHotel LA` | Log Analytics Workspace | Centralized logging & monitoring for Azure Migrate & target workloads |

---

## 🚀 Step-by-Step Implementation Guide

### Step 1: Provision Parent Hyper-V Host (`SmartHotelHost`)

1. Create a Virtual Machine in Azure:
   * **Resource Group:** `SmartHotel-RGRG` (or `SmartHotel-RGHostRG`)
   * **VM Name:** `SmartHotelHost` | **Region:** East US
   * **Image:** Windows Server 2019/2022 Datacenter
   * **Size:** `Standard_D4s_v3` (4 vCPUs, 16 GB RAM) *(Must support Nested Virtualization)*
   * **Security Type:** **Standard** *(Do NOT use Trusted Launch, as it blocks nested virtualization extensions)*
2. Disks:
   * **OS Disk:** `SmartHotelHost_OsDisk`
   * **Data Disk:** 256 GB Data Disk (`SmartHotelHost_DataDisk`) formatted as Drive `E:` for nested VM storage.

---

### Step 2: Configure Hyper-V & NAT Gateway

RDP into `SmartHotelHost` using its Azure Public IP, open **PowerShell as Administrator**, and execute the following configuration scripts:

```powershell
# Step A: Install Hyper-V Feature and Management Tools
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart

# --- REBOOT HOST AND RE-LOG IN VIA RDP ---

# Step B: Create Internal Hyper-V Virtual Switch
New-VMSwitch -Name "SmartHotelSwitch" -SwitchType Internal

# Step C: Assign Default Gateway IP to host virtual adapter
New-NetIPAddress -IPAddress 192.168.10.1 -PrefixLength 24 -InterfaceAlias "vEthernet (SmartHotelSwitch)"

# Step D: Enable NAT Gateway for outbound internet access from nested VMs
New-NetNat -Name "SmartHotelNAT" -InternalIPInterfaceAddressPrefix "192.168.10.0/24"
