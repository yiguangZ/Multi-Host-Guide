# README: Simulating a Two‐Host QEMU/KVM Cluster on One Laptop

This guide walks you through all the commands and steps to mimic two separate hypervisor hosts (Host-VM-1 and Host-VM-2), each running their own nested guest (Nested-VM-A and Nested-VM-B), all on a single Ubuntu laptop. 

You will need a laptop, preferably one with good size RAM. You will also need a ethernet connection to the laptop.

---
### 1 On Your Ubuntu Laptop (Physical Host) Install KVM/QEMU, libvirt, virt-manager, etc.
```bash
sudo apt update
sudo apt install -y \
  qemu-kvm libvirt-daemon-system libvirt-clients \
  bridge-utils virt-manager cpu-checker
sudo usermod -aG libvirt,kvm $USER
# Log out & back in
```
### 2 Verify KVM Support
```
sudo kvm-ok
# → “Should print out KVM acceleration can be used”
```
### 3 Create a Linux Bridge on the Physical NIC
```
# identify the wired NIC name (mine was enx0826ae3a1b00)
ip link show

# 2. create & bring up br0
sudo ip link add br0 type bridge
sudo ip link set br0 up

# 3. enslave your NIC into br0
sudo ip link set enx0826ae3a1b00 master br0
sudo ip link set enx0826ae3a1b00 up


# 4. DHCP on br0sudo dhclient -v br
sudo dhclient -v br0
ip addr show br0
```
### 4 Create Two “Host” VMs (Level-1)
In virt-manager click New VM → Local install media → point at your Ubuntu ISO.

Name: Host-VM-1 | RAM: 1024 MiB | vCPUs: 1

Storage: 20 GiB qcow2 (default pool) (Make sure storage is >15GiB since Ubuntu is 15 GiB)

Network: Bridge br0, model virtio → Finish & install.

Repeat for Host-VM-2.
### 5 Enable nested virt in Host VMs

Back in the host’s virt-manager window, with Host-VM-1 or Host-Vm-2 selected, click the 🛈 (Show hardware details) icon.

Click the “XML” tab.

Locate the <cpu> element and change it from:
```
<cpu mode="host-passthrough" …/>
```
to:
```
<cpu mode="host-passthrough" …>
  <feature policy="require" name="vmx"/>
</cpu>
```
Click Apply, then Shut Down → restart Host-VM.

Inside Host-VM-2 or Host-VM-1, verify nested virt is enabled:
```
grep -E 'vmx|svm' /proc/cpuinfo
sudo kvm-ok
```
Repeat for the other VM.
### 5. Create (or Verify) a Bridge Inside Host‑VM‑1/2
You need a Linux bridge inside Host‑VM‑1 so your nested guest can share the external link:
```
# 1. Add and bring up the bridge
sudo ip link add name br0 type bridge
sudo ip link set dev br0 up

# 2. Enslave Host‑VM‑1’s interface into it
sudo ip link set dev enp1s0 master br0
sudo ip link set dev enp1s0 up

# 3. Obtain an IP via DHCP
sudo dhclient -v br0

# 4. Verify
ip addr show br0    # should list your real‐network IPv4 on br0
```
### 6. Install QEMU/KVM in Host‑VM-1/2
In the Host‑VM‑2 console:
```
sudo apt update
sudo apt install -y \
  qemu-kvm libvirt-daemon-system libvirt-clients \
  bridge-utils virt-manager cpu-checker
sudo usermod -aG libvirt,kvm $USER
# Log out & back in (or reboot) so your user is in the groups
```

### 7. Create Nested-VM-A in Host-VM-1 and Nested-VM-B in Host-VM-2
In Host-VM-1’s virt-manager → New VM → Ubuntu ISO.

Name: Nested-VM-A | RAM: 512 MiB | vCPU: 1

Storage: select nested-disks pool (or default) → 20 GiB.

Network: Bridge br0 → Finish & install.

Repeat to create two nested VMs.

### 8. Configure Nested-VM-A’s Network
```
# inside Nested-VM-A
sudo ip link set enp1s0 up
sudo ip addr add 10.0.0.10/24 dev enp1s0
ip addr show enp1s0
```
### 9. Configure Nested-VM-B’s Network
```
# inside Nested-VM-B
sudo ip link set enp1s0 up
sudo ip addr add 10.0.0.20/24 dev enp1s0
ip addr show enp1s0
```
### 10. Test Connectivity
From Nested-VM-A:
```
ping -c3 10.0.0.20
```
From Nested-VM-B:
```
ping -c3 10.0.0.10
```
### 11. Real Multi-Node Cluster (not nested)
To move from this single-laptop hack to a true multi-host cluster:

Physical hosts: use two or more servers or cloud VMs as true “hosts.”

Shared storage: NFS/Ceph/Gluster so every host mounts the same VM images.

Networking: configure each host’s br0 on the same VLAN or use overlay SDN.

Libvirt remote: in virt-manager → File → Add Connection → QEMU/KVM remote.

Create VMs on each host, storing disks on the shared pool.

Cluster management: use oVirt, Proxmox VE, or OpenStack for live-migration and HA.

This setup gives real CPU/memory isolation, shared-disk live migration, and proper cluster orchestration.
