# README: Simulating a Two‐Host QEMU/KVM Cluster on One Laptop

This guide walks you through all the commands and steps to mimic two separate hypervisor hosts (Host-VM-1 and Host-VM-2), each running their own nested guest (Nested-VM-A and Nested-VM-B), all on a single Ubuntu laptop.

---

## 1. On Your Ubuntu Laptop (Physical Host)

### 1.1 Install KVM/QEMU, libvirt, virt-manager, etc.
```bash
sudo apt update
sudo apt install -y \
  qemu-kvm libvirt-daemon-system libvirt-clients \
  bridge-utils virt-manager cpu-checker
sudo usermod -aG libvirt,kvm $USER
# Log out & back in
1.2 Verify KVM Support
bash
Copy
Edit
sudo kvm-ok
# → “KVM acceleration can be used”
1.3 Create a Linux Bridge on the Physical NIC
bash
Copy
Edit
# 1. identify your wired NIC name (e.g. enx0826ae3a1b00)
ip link show

# 2. create & bring up br0
sudo ip link add br0 type bridge
sudo ip link set br0 up

# 3. enslave your NIC into br0
sudo ip link set enx0826ae3a1b00 master br0
sudo ip link set enx0826ae3a1b00 up

# 4. get an IP on the bridge
sudo dhclient -v br0
ip addr show br0
2. Create Two “Host” VMs (Level-1)
In virt-manager click New VM → Local install media → point at your Ubuntu ISO.

Name: Host-VM-1 | RAM: 1024 MiB | vCPUs: 1

Storage: 8 GiB qcow2 (default pool)

Network: Bridge br0, model virtio → Finish & install.

Repeat for Host-VM-2.

2.1 Enable Nested Virt
For each Host-VM:

bash
Copy
Edit
# In virt-manager UI:
#  - Right-click VM → Show Hardware Details → CPU
#  - Set Mode: host-passthrough
#  - (Optional) Add <feature name='vmx' policy='require'/> under <cpu> in XML
#  - Apply & reboot the VM
3. Inside Each Host-VM: Prep for Nested VMs
Log into Host-VM-1 (then repeat in Host-VM-2).

3.1 Install KVM/QEMU & virt-manager
bash
Copy
Edit
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
3.2 Create a Bridge in the VM
bash
Copy
Edit
# 1. identify interface (likely enp1s0)
ip link show

# 2. create & bring up VM-bridge
sudo ip link add br0 type bridge
sudo ip link set br0 up

# 3. enslave VM NIC into br0
sudo ip link set enp1s0 master br0
sudo ip link set enp1s0 up

# 4. DHCP on br0
sudo dhclient -v br0
ip addr show br0
3.3 (Optional) Add Extra Disk for Nested-VMs
On the host virt-manager → Host-VM-1 → Add Hardware → Storage → Create 20 GiB qcow2 → Finish.
Then inside Host-VM-1:

bash
Copy
Edit
sudo partprobe
lsblk         # see /dev/vdb
sudo mkfs.ext4 /dev/vdb
sudo mkdir /mnt/vm-disks
sudo mount /dev/vdb /mnt/vm-disks
echo '/dev/vdb /mnt/vm-disks ext4 defaults 0 2' | sudo tee -a /etc/fstab
In the host’s connection details → Storage → + → Directory → /mnt/vm-disks → Build.

4. Create Nested-VM-A in Host-VM-1
In Host-VM-1’s virt-manager → New VM → Ubuntu ISO.

Name: Nested-VM-A | RAM: 512 MiB | vCPU: 1

Storage: select nested-disks pool (or default) → 8 GiB.

Network: Bridge br0 → Finish & install.

4.1 Configure Nested-VM-A’s Network
bash
Copy
Edit
# inside Nested-VM-A
sudo ip link set enp1s0 up
sudo ip addr add 10.0.0.10/24 dev enp1s0
ip addr show enp1s0
5. Create Nested-VM-B in Host-VM-2
Repeat Section 4 in Host-VM-2:

bash
Copy
Edit
# inside Nested-VM-B
sudo ip link set enp1s0 up
sudo ip addr add 10.0.0.20/24 dev enp1s0
ip addr show enp1s0
6. Test Connectivity
From Nested-VM-A:

bash
Copy
Edit
ping -c3 10.0.0.20
From Nested-VM-B:

bash
Copy
Edit
ping -c3 10.0.0.10
Optionally, install iperf3 on both:

bash
Copy
Edit
# VM-B:
sudo apt install -y iperf3
iperf3 -s

# VM-A:
sudo apt install -y iperf3
iperf3 -c 10.0.0.20
7. Real Multi-Node Cluster (ⓘ not nested)
To move from this single-laptop hack to a true multi-host cluster:

Use two (or more) physical servers or cloud VMs as true “hosts.”

Share storage via NFS/Ceph/Gluster so every host can mount the same VM images.

Network setup: configure each host’s br0 on the same VLAN (or overlay SDN).

Configure libvirt remote: add each host into virt-manager under File → Add Connection → QEMU/KVM remote.

Create VMs directly on each host, attaching their disks to the shared storage pool.

Leverage HA & migration: use oVirt, Proxmox VE, or OpenStack Nova+Neutron to manage live-migration and failover across hosts.

This gives you real isolation (CPU/memory), true shared-disk live migration, and cluster management—rather than nested-within-a-single-kernel.
