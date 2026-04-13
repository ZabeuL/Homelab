# Proxmox VE

**IP:** `192.168.2.2` (static, set during install)  
**Domain:** `pve.local`  
**Access:** `https://proxmox.zabeu.dev` (via NPM) or `https://192.168.2.2:8006`

## Post-Install Steps

1. Switched to the no-subscription repository
2. Removed `local-lvm` and expanded `local` to use the full disk — see [Architecture](architecture.md)
3. Enabled IOMMU in BIOS

## NAT Configuration

Proxmox acts as a NAT gateway for the Ubuntu VM. The Bell Gigahub only responds to ARP from MACs it assigned via DHCP — the VM's static IP was never issued by the router, so its MAC was never in the ARP table and the router ignored it entirely. Proxmox masquerades the VM's outbound traffic as its own IP so the router only ever sees a known MAC.

```bash
# Enable IP forwarding — persisted in /etc/sysctl.conf
net.ipv4.ip_forward=1

# NAT masquerade rule — VM traffic exits as 192.168.2.2
iptables -t nat -A POSTROUTING -s 192.168.2.3/32 -o vmbr0 -j MASQUERADE
```

Rules persisted via `iptables-persistent` / `netfilter-persistent save`.

Any additional VMs added to this lab should also use `192.168.2.2` as their default gateway for the same reason.

## VT-x / KVM Note

After enabling Virtualization in BIOS, a warm reboot was not sufficient — KVM flags did not appear in `/proc/cpuinfo`. The BIOS was not retaining settings on a warm reboot (XMP, boot order, and VT-x all reset to defaults). A full cold boot with the power cable unplugged forced the CPU to reinitialize and pick up the BIOS settings correctly.

Verify KVM is active before creating VMs:

```bash
kvm-ok
# or
grep -E 'vmx|svm' /proc/cpuinfo
```
