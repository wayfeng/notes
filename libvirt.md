# Managing virtual devices on Linux with libvirt

## Install tools

Install necessary tools for the demo.
```bash
sudo apt install qemu-kvm qemu-system-x86 libvirt-clients \
                   libvirt-daemon-system virtinst cloud-utils -y
```
Then grant access right of KVM and libvirt to current user.
```bash
sudo usermod -aG kvm $(id -u)
sudo usermod -aG libvirt $(id -u)
```

Check if everything is OK.
```bash
$ kvm-ok
  INFO: /dev/kvm exists
  KVM acceleration can be used

$ virsh list
 Id   Name   State
--------------------
```

## Setup VM with libvirt

### Setup bridged network (optional)

Create a bridge `br0` and set NIC `eno1` as it's interface.
```bash
sudo apt install bridge-utils -y
sudo brctl addbr br0
sudo brctl addif br0 eno1
```

Then bring up `br0` with `netplan`.
```bash
cat <<EOF | sudo tee /etc/netplan/00-installer-config.yaml
network:
    ethernets:
        eno1:
            dhcp4: no
    bridges:
        br0:
            dhcp4: yes
            interfaces:
                - eno1
    version: 2'
EOF
sudo netplan apply
```

Apply bridged network to `libvirt`
```bash
cat <<EOF > /tmp/bridged-network.xml
<network>
    <name>bridged-network</name>
    <forward mode="bridge" />
    <bridge name="br0" />
</network>
EOF
virsh net-define /tmp/bridged-network.xml
virsh net-start bridged-network
virsh net-autostart bridged-network
```

### Create root image

You can choose whatever version you need, for simplicity,
I'll use Ubuntu 20.04 minimal cloud image in this demo.
```bash
wget http://cloud-images.ubuntu.com/minimal/releases/focal/release-20211130/ubuntu-20.04-minimal-cloudimg-amd64.img
```

Make sure that file type of downloaded image is *qcow*.
```console
$ qemu-img info ubuntu-20.04-minimal-cloudimg-amd64.img

image: ubuntu-20.04-minimal-cloudimg-amd64.img
file format: qcow2
virtual size: 2.2 GiB (2361393152 bytes)
disk size: 245 MiB
cluster_size: 65536
Format specific information:
    compat: 0.10
    refcount bits: 16
```
For less confusion, rename base image.
```bash
mv ubuntu-20.04-minimal-cloudimg-amd64.img ubuntu-20.04-minimal-cloudimg-amd64.qcow
```

Create root image of VM and enlarge disk size.
```bash
$ qemu-img convert -f qcow2 -O qcow2 ubuntu-20.04-minimal-cloudimg-amd64.qcow2 disk.qcow2

$ qemu-img  resize disk.qcow2 128G

$ qemu-img info disk.qcow2

image: disk.qcow2
file format: qcow2
virtual size: 128 GiB (137438953472 bytes)
disk size: 33.9 GiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

### Create user data

Generate password which you are going to use in the virtual machine.
```bash
$ mkpasswd --method=SHA-512 --rounds=4096
Password:
$6$rounds=4096$JD68ijyq6bBiLCcP$PtUuzIUCNPnXWiBK/w7lkZnjg4PmIVeAufcKqFgVwWpCjujybiubO/xkt12o8qHCgi7wx4.nCQPhAPMoc1adb.
```

Generate SSH key
```bash
$ ssh-keygen -C "eii-virtual-machine"
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3... eii-virtual-machine
```

Create user data file for [cloud-init](https://cloudinit.readthedocs.io/en/latest/).
```bash
cat user-data
```
```yaml
#cloud-config
hostname: vm-eii
manage_etc_hosts: true
users:
  - default
  - name: eii
    gecos: John Doe
    lock_passwd: false
    passwd: $6$rounds=4096$JD68ijy...adb.
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    home: /home/eii
    shell: /bin/bash
    ssh-authorized-keys:
      - ssh-rsa AAAAB3...eii-virtual-machine

apt:
  primary:
    - arches: [default]
      uri: http://us.archive.ubuntu.com/ubuntu/
  http_proxy: http://child-prc.intel.com:913/
  https_proxy: http://child-prc.intel.com:913/

packages:
  - vim
  - git

package_update: true
package_upgrade: true
```

Then create image of user data.
```bash
cloud-localds user-data.img user-data
```

### Start virtual machine with libvirt CLI

The parameters of libvirt CLI is a bit messy. For simplicity, I created a shell script to create the virtual machine.

```bash
cat <<EOF > install-eii-vm.sh
#!/usr/bin/env bash
VM_NAME=eii
VCPUS="10"
RAM_MB="8192"

virt-install \
    --connect="qemu:///system" \
    --name="${VM_NAME}" \
    --vcpus="${VCPUS}" \
    --memory="${RAM_MB}" \
    --os-variant="ubuntu20.04" \
    --disk $HOME/eii/disk.qcow2,device=disk,bus=virtio \
    --disk $HOME/eii/user-data.img,format=raw \
    --virt-type kvm \
    --graphics none \
    --import \
    --network network=bridged-network
EOF
chmod +x install-eii-vm.sh
```
Then run this script to create the virtual machine named *eii* in background.
```bash
./install-eii-vm.sh
```

Now you can check the state of *eii* you just created.
```bash
$ virsh list
 Id   Name   State
----------------------
 1    eii    running
```

There are two ways to connect to the VM: using `virsh console` command or `ssh`.

- With `virsh console`
```bash
$ virsh console eii
Connected to domain eii
Escape character is ^] <press Enter>

eii@vm-eii:~$ ip addr show dev enp1s0 scope global
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:b8:8f:cb brd ff:ff:ff:ff:ff:ff
    inet 10.238.156.118/23 brd 10.238.157.255 scope global dynamic enp1s0
       valid_lft 2884sec preferred_lft 2884sec
```

- With `ssh`

```bash
$ ssh eii@<vm-ip-addr>
```

To get ip address of VM if bridged network is used:
```bash
$ arp -an | grep $(virsh dumpxml eii | grep "mac address" | awk -F\' '{ print $2}')
? (10.238.156.118) at 52:54:00:b8:8f:cb [ether] on br0
```
Otherwise use virsh command:
```bash
virsh net-dhcp-leases default
```
