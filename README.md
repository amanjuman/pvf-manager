# pvf-manager

> Browser-based port forwarding manager for Proxmox VE — generate, manage and export iptables DNAT rules for SDN NAT zones.

**Live demo:** https://amanjuman.github.io/pvf-manager/

---

## What is this?

When your Proxmox VMs or LXC containers sit behind a [Proxmox SDN Simple zone](https://pve.proxmox.com/wiki/Setup_Simple_Zone_With_SNAT_and_DHCP) with NAT, they all share the host's public IP. To expose services running inside those VMs/LXCs, you need iptables DNAT (port forwarding) rules on the host.

`pvf-manager` gives you a clean UI to:

- Define port forwarding rules (protocol, host port → VM IP:port)
- Validate against reserved host ports (SSH, Proxmox Web UI, etc.)
- Export a ready-to-use shell script
- Install it as a persistent systemd service

No backend, no dependencies. Everything runs in your browser and is stored in `localStorage`.

---

## Features

- **Setup wizard** — enter your host details once (WAN interface, public IP, NAT subnet)
- **Rule manager** — add, edit, delete forwarding rules
- **Reserved port protection** — blocks `22`, `8006`, `8007`, `5900`, `3128`, `111`
- **Script export** — generates a clean `/etc/iptables-portforward.sh`
- **Download** — saves the script directly to your machine
- **Install guide** — step-by-step systemd setup built into the UI
- **Persistent config** — your setup is remembered across visits via localStorage

---

## Requirements

- Proxmox VE with SDN configured as a **Simple zone** with `snat 1`
- A WAN-facing bridge (e.g. `vmbr0`) with your public IP
- VMs or LXCs on the NAT subnet (e.g. `192.168.101.0/24`)

---

## Usage

### Option A — Use the hosted page

Visit **https://amanjuman.github.io/pvf-manager/** in your browser.

### Option B — Self-host or run locally

Just open `index.html` — it's a single self-contained file with no dependencies.

```bash
git clone https://github.com/amanjuman/pvf-manager
cd pvf-manager
open index.html   # or just drag it into your browser
```

---

## Installing the generated script on your Proxmox host

Once you've defined your rules and exported the script:

**1. Upload to host**
```bash
scp iptables-portforward.sh root@YOUR_HOST_IP:/etc/iptables-portforward.sh
```

**2. Make executable**
```bash
chmod +x /etc/iptables-portforward.sh
```

**3. Create systemd service**
```bash
cat > /etc/systemd/system/iptables-portforward.service << 'EOF'
[Unit]
Description=IPTables Port Forwarding Rules for SDN NAT VMs
After=network.target pve-firewall.service

[Service]
Type=oneshot
ExecStart=/etc/iptables-portforward.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

**4. Enable and start**
```bash
systemctl daemon-reload
systemctl enable --now iptables-portforward.service
```

**5. Verify**
```bash
systemctl status iptables-portforward.service
iptables -t nat -L PREROUTING -n -v --line-numbers
```

**After adding or removing rules**, regenerate the script and reapply:
```bash
systemctl restart iptables-portforward.service
```

> **Note:** After running **SDN Apply** in the Proxmox UI, iptables rules may be flushed. Run the above restart command to reapply.

---

## Example generated script

```bash
#!/bin/bash
# Proxmox NAT port forwarding rules
# Host: pve-myhost (203.0.113.10) | NAT: 192.168.101.0/24

# Flush existing custom rules before reapplying
iptables -t nat -F PREROUTING
iptables -F FORWARD

PUB_IF="vmbr0"

# web-01 nginx
iptables -t nat -A PREROUTING -i $PUB_IF -p tcp --dport 8080 -j DNAT --to-destination 192.168.101.10:80
iptables -A FORWARD -i $PUB_IF -d 192.168.101.10 -p tcp --dport 80 -j ACCEPT

# db-01 SSH
iptables -t nat -A PREROUTING -i $PUB_IF -p tcp --dport 2222 -j DNAT --to-destination 192.168.101.11:22
iptables -A FORWARD -i $PUB_IF -d 192.168.101.11 -p tcp --dport 22 -j ACCEPT
```

---

## Reserved ports

The following host ports cannot be used as forwarding targets to protect host services:

| Port | Service |
|------|---------|
| `22` | Host SSH |
| `8006` | Proxmox Web UI |
| `8007` | Proxmox SPICE proxy |
| `3128` | Proxmox SPICE (alt) |
| `5900` | VNC |
| `111` | rpcbind |

---

## Privacy

All data (host config and rules) is stored in your browser's `localStorage`. Nothing is sent to any server.

---

## License

MIT
