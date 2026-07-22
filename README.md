# Proxmox Pentest Lab — Build Notes

A nested home lab for pentesting practice, built inside VMware Workstation. Goal: a Kali attacker box plus expandable victim machines, with room to grow toward enterprise-style segmentation later.

## Stack

- **Hypervisor:** Proxmox VE (installed as a VM inside VMware Workstation)
- **Gateway/Firewall:** pfSense
- **Attacker:** Kali Linux
- **Victims:** Metasploitable 2, Metasploitable 3

## Topology

```
                Internet (VMware NAT)
                       |
                    vmbr0 (WAN)
                       |
                  ┌─────────┐
                  │ pfSense │  LAN: 10.10.10.1
                  └─────────┘
                       |
                    vmbr1 (LAN, no physical NIC — pure L2 bridge)
                       |
        ┌──────────────┼──────────────┐
      Kali          Metasploitable2  Metasploitable3
   10.10.10.x         10.10.10.x       10.10.10.x
```

Proxmox host itself sits on `192.168.64.3`, with `192.168.64.2` reserved as the VMware NAT gateway.

## Setup Steps

### 1. Proxmox install
- Assigned the host `192.168.64.3`, leaving `.2` for the VMware NAT gateway.
- Swapped the enterprise repo for the no-subscription one so `apt` works without a license:
  - Removed `pve-enterprise.list`
  - Added `pve-no-subscription.list` → `deb http://download.proxmox.com/debian/pve trixie pve-no-subscription`
  - Ran `apt-get update && apt-get dist-upgrade`

### 2. Kali VM
- Hit an I/O error on install — root cause was an undersized disk. Fixed by growing it:
  ```bash
  growpart /dev/sda 3
  pvresize /dev/sda3
  vgs
  lvextend -l +100%FREE /dev/pve/data
  ```
- Locked down management access at the VM firewall level — a rule on Kali specifically rejecting outbound traffic to the Proxmox panel:
  ```
  Type: out | Action: REJECT | Proto: tcp
  Source: * | Source Port: *
  Destination: 192.168.64.3 | Destination Port: 8006
  ```
  Note: this only blocks Kali from reaching the panel — worth reconsidering as a REJECT-all-ports rule once more victim machines are added.

### 3. pfSense
- Two NICs: Kali on LAN, internet on WAN, deny-by-default (pfSense's out-of-the-box posture) — Kali can reach out, nothing can reach in.
- **IP conflict bug:** gave pfSense's LAN interface `10.10.10.1`, not realizing `vmbr1` already held that address from the original bridge setup. Two devices claiming the same IP broke hostname resolution (raw pings to `8.8.8.8` worked, DNS didn't).
  - **Fix:** stripped the IP from `vmbr1`, leaving it as a pure Layer 2 bridge; pfSense kept `10.10.10.1` as the sole LAN address.
- Reviewed firewall rules: WAN has no rules (implicit deny holds), LAN still has the default "allow LAN net to any" (left permissive — nothing sensitive behind it yet), outbound NAT set to Automatic.
- **Known gap:** Proxmox's management panel and pfSense's WAN currently share `vmbr0`. Not an active risk today (nothing untrusted lives there), but flagged for fixing via out-of-band WireGuard access before the lab gets busier.

### 4. TLS trust for the Proxmox panel
- Exported Proxmox's self-signed cert and imported it into:
  - Windows' Trusted Root Certification Authorities store
  - Firefox's own cert store separately (Firefox doesn't use the Windows trust store)
- A full browser restart was required — a page reload alone didn't pick up the new trust.

### 5. Snapshot
- Took a baseline snapshot once pfSense + Kali were confirmed stable, before bringing in victim machines.

### 6. Victim machines — Metasploitable 2 & 3
- Imported as pre-built `.vmdk` disks rather than fresh installs.
- **Boot failure on first attempt** (`Boot failed: not a bootable disk`): the VM creation wizard auto-creates a blank disk, which gets set as the boot device while the real imported disk sits unattached as "Unused Disk 0."
  - **Fix:** remove the blank auto-created disk, attach the imported one as **IDE** (not SCSI/VirtIO — these images predate VirtIO support), confirm boot order under Options.
  - **Takeaway:** delete the wizard's auto-created disk *before* importing, every time.

## Current State

- Kali + both Metasploitable boxes sit flat on `vmbr1` — same subnet, same broadcast domain.
- Because it's all Layer 2, pfSense never sees attacker↔victim traffic — nothing to route. Visibility for now is a direct `tcpdump` on `vmbr1` from the Proxmox host as needed.

## Next Steps

- Split Kali onto its own VLAN, separate from the victims (deferred to the segmentation phase).
- Once segmented, run Suricata/Zeek on pfSense for proper IDS-style logging of attacker traffic.
- Close the Proxmox-panel/pfSense-WAN shared-bridge gap with out-of-band WireGuard access.
- Add a Windows victim machine.
