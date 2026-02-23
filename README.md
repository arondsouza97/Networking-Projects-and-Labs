# 🖧 VMware 3-VM Network Lab

> A hands-on virtual networking lab with a **Client**, **DNS/DHCP Server**, and **Web Server** running on two isolated networks in VMware Workstation Pro — plus a bridged adapter on each VM for internet access.

---

## 📋 What You'll Build

You'll set up three virtual machines that talk to each other across two private networks. The DNS/DHCP server sits in the middle, routing traffic between the client and the web server. Every VM also has a bridged adapter so you can download packages from the internet during setup.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     VMware Workstation Pro (Host)                    │
│                                                                     │
│   VMnet2 ─ Client LAN (192.168.10.0/24)                            │
│   ┌─────────────┐              ┌────────────────────┐               │
│   │  Client VM   │◄────────────►  DNS/DHCP Server   │               │
│   │  (Ubuntu     │   VMnet2     │  (Ubuntu Server)   │              │
│   │   Desktop)   │              │                    │               │
│   │              │              │  NIC1: 192.168.10.1│               │
│   │  NIC1: DHCP  │              │  NIC2: 172.16.0.1  │              │
│   │  (192.168.   │              │  NIC3: Bridged      │              │
│   │   10.100-200)│              │       (Internet)    │              │
│   │  NIC2:Bridged│              └─────────┬──────────┘               │
│   │   (Internet) │                        │                         │
│   └─────────────┘                        │ VMnet3                   │
│                                           │                         │
│   VMnet3 ─ Server Network (172.16.0.0/24) │                         │
│                                           │                         │
│                               ┌───────────┴──────────┐              │
│                               │    Web Server         │              │
│                               │    (Ubuntu Server)    │              │
│                               │                       │              │
│                               │    NIC1: 172.16.0.10  │              │
│                               │    NIC2: Bridged      │              │
│                               │         (Internet)    │              │
│                               └───────────────────────┘              │
│                                                                     │
│   ── ── ── ── ── ── ── ── ── ── ── ── ── ── ── ── ── ── ── ──     │
│   Bridged Adapter (all VMs) ──► Your Home/Office Router ──► Internet│
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Prerequisites

Before you start, make sure you have:

- **VMware Workstation Pro** (version 16 or later)
- **Ubuntu Server 22.04 LTS ISO** — for the DNS/DHCP server and web server
- **Ubuntu Desktop 22.04 ISO** (or Windows 10/11 ISO) — for the client
- **8 GB+ RAM** on your host machine (each VM gets 2 GB)
- **60 GB+ free disk space** (20 GB per VM)
- **A working internet connection** — the bridged adapter shares this with your VMs

---

## 🌐 Network Summary

| Network | Subnet | Type | Purpose |
|---------|--------|------|---------|
| **VMnet2** | `192.168.10.0/24` | Host-only (no DHCP) | Client ↔ DNS/DHCP Server |
| **VMnet3** | `172.16.0.0/24` | Host-only (no DHCP) | DNS/DHCP Server ↔ Web Server |
| **Bridged** | Your home network | Bridged (Automatic) | Internet access on all VMs |

### VM NIC Assignments

| VM | NIC 1 | NIC 2 | NIC 3 |
|----|-------|-------|-------|
| **Client** | VMnet2 (DHCP: `192.168.10.100–200`) | Bridged (Internet) | — |
| **DNS/DHCP Server** | VMnet2 (`192.168.10.1`) | VMnet3 (`172.16.0.1`) | Bridged (Internet) |
| **Web Server** | VMnet3 (`172.16.0.10`) | Bridged (Internet) | — |

---

## 🚀 Setup Steps

### Step 1 — Create the Virtual Networks

Open VMware Workstation Pro and go to **Edit → Virtual Network Editor**.

1. Click **Change Settings** (accept the admin prompt).
2. **Add VMnet2:**
   - Click **Add Network...** → select **VMnet2** → OK.
   - Set type to **Host-only**.
   - Subnet IP: `192.168.10.0` / Mask: `255.255.255.0`.
   - **Uncheck** "Use local DHCP service to distribute IP addresses to VMs".
3. **Add VMnet3:**
   - Click **Add Network...** → select **VMnet3** → OK.
   - Set type to **Host-only**.
   - Subnet IP: `172.16.0.0` / Mask: `255.255.255.0`.
   - **Uncheck** "Use local DHCP service to distribute IP addresses to VMs".
4. Click **Apply** → **OK**.

> ⚠️ **Important:** You must disable VMware's built-in DHCP on both networks. If you leave it on, it will conflict with the DHCP server you're about to build.

---

### Step 2 — Create the Client VM

#### 2a. Create the VM

1. **File → New Virtual Machine...** → Typical → Next.
2. Select your **Ubuntu Desktop 22.04 ISO** → Next.
3. Name it `Client-VM` → Next.
4. Disk: **20 GB** → Next.
5. Click **Customize Hardware...** → set **Memory to 2048 MB**, **Processors to 2** → Close → Finish.

#### 2b. Configure NICs

Right-click `Client-VM` → **Settings...**

| Adapter | Setting |
|---------|---------|
| **Network Adapter** (NIC 1) | Custom: Specific virtual network → **VMnet2** |
| **Add... → Network Adapter** (NIC 2) | **Bridged: Connected directly to the physical network** |

Leave "Replicate physical network connection state" **unchecked** on the bridged adapter.

#### 2c. Install Ubuntu Desktop

Power on the VM and install Ubuntu Desktop normally. The bridged adapter gives you internet during installation.

#### 2d. Configure Networking

After installation, edit the Netplan config:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens33:                        # NIC 1 - Internal (VMnet2)
      dhcp4: true
    ens36:                        # NIC 2 - Bridged (Internet)
      dhcp4: true
```

```bash
sudo netplan apply
```

> 💡 The internal NIC (`ens33`) won't get an IP until the DHCP server is running. The bridged NIC (`ens36`) works immediately for internet.

---

### Step 3 — Create the DNS/DHCP Server

#### 3a. Create the VM

1. **File → New Virtual Machine...** → Typical → Next.
2. Select your **Ubuntu Server 22.04 ISO** → Next.
3. Name it `DNS-DHCP-Server` → Next.
4. Disk: **20 GB** → Next.
5. Click **Customize Hardware...** → **Memory: 2048 MB**, **Processors: 2** → Close → Finish.

#### 3b. Configure NICs (3 adapters)

Right-click `DNS-DHCP-Server` → **Settings...**

| Adapter | Setting |
|---------|---------|
| **Network Adapter** (NIC 1) | Custom → **VMnet2** |
| **Add... → Network Adapter** (NIC 2) | Custom → **VMnet3** |
| **Add... → Network Adapter** (NIC 3) | **Bridged** |

Leave "Replicate physical network connection state" **unchecked**.

#### 3c. Install Ubuntu Server

Power on and install Ubuntu Server. Install **OpenSSH server** when prompted.

#### 3d. Configure Static IPs

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens33:                        # NIC 1 - Client LAN (VMnet2)
      addresses:
        - 192.168.10.1/24
    ens36:                        # NIC 2 - Server Network (VMnet3)
      addresses:
        - 172.16.0.1/24
    ens37:                        # NIC 3 - Bridged (Internet)
      dhcp4: true
```

```bash
sudo netplan apply
```

#### 3e. Enable IP Forwarding

This lets the server route traffic between the two internal networks:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Verify:

```bash
cat /proc/sys/net/ipv4/ip_forward
# Should output: 1
```

#### 3f. Install and Configure DHCP

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

Edit `/etc/dhcp/dhcpd.conf`:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

```conf
option domain-name "lab.local";
option domain-name-servers 192.168.10.1;

default-lease-time 600;
max-lease-time 7200;
authoritative;

# Client LAN - serves DHCP addresses
subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.100 192.168.10.200;
    option routers 192.168.10.1;
    option domain-name-servers 192.168.10.1;
}

# Server Network - declared but no pool (static only)
subnet 172.16.0.0 netmask 255.255.255.0 {
}
```

Set the listening interface:

```bash
sudo nano /etc/default/isc-dhcp-server
```

```
INTERFACESv4="ens33"
```

Start the service:

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

> ⚠️ The empty `subnet 172.16.0.0` block is required. Without it, the DHCP server refuses to start because it sees an interface on an undeclared subnet.

#### 3g. Install and Configure DNS (BIND9)

```bash
sudo apt install bind9 bind9-utils -y
```

Edit `/etc/bind/named.conf.options`:

```bash
sudo nano /etc/bind/named.conf.options
```

```conf
options {
    directory "/var/cache/bind";
    listen-on { 192.168.10.1; 172.16.0.1; 127.0.0.1; };
    allow-query { any; };
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
    recursion yes;
};
```

Add the zone to `/etc/bind/named.conf.local`:

```bash
sudo nano /etc/bind/named.conf.local
```

```conf
zone "lab.local" {
    type master;
    file "/etc/bind/zones/db.lab.local";
};
```

Create the zone file:

```bash
sudo mkdir -p /etc/bind/zones
sudo nano /etc/bind/zones/db.lab.local
```

```conf
$TTL    604800
@       IN      SOA     ns1.lab.local. admin.lab.local. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; Name server
@       IN      NS      ns1.lab.local.

; A records
ns1         IN      A       192.168.10.1
www         IN      A       172.16.0.10
webserver   IN      A       172.16.0.10
```

Validate and start:

```bash
sudo named-checkconf
sudo named-checkzone lab.local /etc/bind/zones/db.lab.local
sudo systemctl restart bind9
sudo systemctl enable bind9
```

---

### Step 4 — Create the Web Server

#### 4a. Create the VM

1. **File → New Virtual Machine...** → Typical → Next.
2. Select your **Ubuntu Server 22.04 ISO** → Next.
3. Name it `Web-Server` → Next.
4. Disk: **20 GB** → Next.
5. Click **Customize Hardware...** → **Memory: 2048 MB**, **Processors: 2** → Close → Finish.

#### 4b. Configure NICs

Right-click `Web-Server` → **Settings...**

| Adapter | Setting |
|---------|---------|
| **Network Adapter** (NIC 1) | Custom → **VMnet3** |
| **Add... → Network Adapter** (NIC 2) | **Bridged** |

Leave "Replicate physical network connection state" **unchecked**.

#### 4c. Install Ubuntu Server

Power on and install. The bridged adapter provides internet during installation.

#### 4d. Configure Network Interfaces

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens33:                        # NIC 1 - Server Network (VMnet3)
      addresses:
        - 172.16.0.10/24
      routes:
        - to: 192.168.10.0/24
          via: 172.16.0.1
      nameservers:
        addresses:
          - 172.16.0.1
    ens36:                        # NIC 2 - Bridged (Internet)
      dhcp4: true
```

```bash
sudo netplan apply
```

> 💡 The route to `192.168.10.0/24` via `172.16.0.1` is critical — without it, the web server can't send responses back to the client.

#### 4e. Install Apache

```bash
sudo apt update
sudo apt install apache2 -y
echo "<h1>Hello from Web Server - lab.local</h1>" | sudo tee /var/www/html/index.html
sudo systemctl enable apache2
sudo systemctl start apache2
```

#### 4f. Configure Firewall (Optional)

```bash
sudo ufw allow 80/tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

---

## ✅ Verification

Once all three VMs are running, hop onto the **Client VM** and run these tests:

```bash
# 1. Check that DHCP gave you an IP
ip addr show ens33
# → You should see an IP in 192.168.10.100–200

# 2. Test DNS resolution
nslookup www.lab.local
# → Should resolve to 172.16.0.10

# 3. Ping the DNS/DHCP server (same network)
ping -c 4 192.168.10.1

# 4. Ping the web server (routed through DNS/DHCP server)
ping -c 4 172.16.0.10

# 5. Access the web page
curl http://www.lab.local
# → Should return: <h1>Hello from Web Server - lab.local</h1>

# 6. Test internet (via bridged adapter)
ping -c 4 google.com
```

### Expected Results

| Test | Expected Result |
|------|----------------|
| `ip addr show ens33` | IP in `192.168.10.100–200` range |
| `nslookup www.lab.local` | Resolves to `172.16.0.10` |
| `ping 192.168.10.1` | Reply from DNS/DHCP server |
| `ping 172.16.0.10` | Reply from Web Server (routed) |
| `curl http://www.lab.local` | HTML page from Apache |
| `ping google.com` | Reply via bridged adapter |

---

## 🔥 Troubleshooting

### Client not getting an IP from DHCP

- Check DHCP server status: `sudo systemctl status isc-dhcp-server`
- View DHCP logs: `sudo journalctl -u isc-dhcp-server -f`
- Confirm the Client's NIC 1 is on **VMnet2** in VM Settings
- Make sure VMware's built-in DHCP is **disabled** for VMnet2
- Try renewing: `sudo dhclient -r ens33 && sudo dhclient ens33`

### DNS not resolving

- Check BIND9 status: `sudo systemctl status bind9`
- Validate config: `sudo named-checkconf`
- Validate zone: `sudo named-checkzone lab.local /etc/bind/zones/db.lab.local`
- Test directly: `dig @192.168.10.1 www.lab.local`

### Client can't ping the web server

- Verify IP forwarding: `cat /proc/sys/net/ipv4/ip_forward` (should be `1`)
- Check routes on DNS/DHCP server: `ip route`
- Check routes on web server: `ip route show` (must have `192.168.10.0/24 via 172.16.0.1`)
- Temporarily disable firewalls: `sudo ufw disable`

### Web page not loading

- Check Apache: `sudo systemctl status apache2`
- Test locally on web server: `curl http://localhost`
- Check port 80: `sudo ss -tlnp | grep :80`
- Check firewall: `sudo ufw status`

### NIC names don't match (not ens33/ens36/ens37)

Your interfaces might have different names. Identify them:

```bash
ip link show
# Match MAC addresses with what VMware shows in VM Settings
# for each Network Adapter
```

Update all Netplan configs with the correct names.

### Bridged adapter not getting internet

- Make sure your host machine has internet
- In VM Settings, try changing Bridged from "Automatic" to your specific host adapter (e.g., "Intel Wi-Fi" or "Realtek Ethernet")
- Check: `ip addr show ens36` — you should see an IP from your home router

---

## 🔒 Optional: Disable Bridged Adapters After Setup

Once all packages are installed, you can fully isolate the lab by removing the bridged adapters.

**In VMware Workstation Pro:**

1. Power off the VM.
2. Right-click → **Settings...**
3. Select the Bridged adapter → click **Remove**.
4. Click **OK**.

**Or disable in the OS:**

```bash
# Bring the bridged interface down
sudo ip link set ens36 down

# Or remove it from Netplan and apply
sudo nano /etc/netplan/00-installer-config.yaml
# Delete the ens36/ens37 (bridged) section
sudo netplan apply
```

> ⚠️ After this, the VMs won't have internet. Make sure everything is installed first.

---

## 📁 Project Structure

```
vmware-3vm-lab/
├── README.md                  ← You are here
└── VMware_Lab_Setup_Guide.docx ← Detailed Word document with full instructions
```

---

## 📜 License

This project is provided as-is for educational and lab purposes. Feel free to use, modify, and share.
