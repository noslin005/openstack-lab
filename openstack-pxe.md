
# OpenStack PXE 

## Port Setup

- OPTION A: Neutron DHCP Agent as DHCP Server

```bash
# This was previously created
MGMT_NET=mgmt-net

openstack network create pxe-net
openstack subnet create pxe-subnet --network pxe-net --subnet-range 10.77.0.0/24 --gateway none

# Security group
openstack security group create pxe-sg
for r in "tcp 80" "tcp 22"; do set -- $r; openstack security group rule create --proto $1 --dst-port $2 pxe-sg; done
openstack security group rule create --proto icmp pxe-sg

# Server port: fixed IP
openstack port create pxe-server-port --network pxe-net --fixed-ip subnet=pxe-subnet,ip-address=10.77.0.10 --security-group pxe-sg

# Client port: Neutron's DHCP hands out the boot URL (DHCP option 67)
openstack port create pxe-client-port --network pxe-net --extra-dhcp-option name=bootfile-name,value=http://10.77.0.10/netboot/boot.ipxe,ip-version=4 --security-group pxe-sg

# TODO: Need to figure out how to security group properly
openstack port set pxe-server-port --no-security-group --disable-port-security
openstack port set pxe-client-port --no-security-group --disable-port-security
```

- OPTION B: DHCP Server In PXE-SERVER

```bash
# Create the ports as above but disable security group and port-security
openstack network create pxe-net
openstack subnet create pxe-subnet --network pxe-net --subnet-range 10.77.0.0/24 --gateway none --no-security-group --disable-port-security

# Server port: fixed IP
openstack port create pxe-server-port --network pxe-net --fixed-ip subnet=pxe-subnet,ip-address=10.77.0.10  --no-security-group --disable-port-security

# OR
openstack port set pxe-server-port --no-security-group --disable-port-security
openstack port set pxe-client-port --no-security-group --disable-port-security

# Server handles DHCP
openstack subnet set pxe-subnet --no-dhcp
```

## PXE Server Setup

Create the VM and wait for the VM to start:

```bash
SRV_PORT=$(openstack port show pxe-server-port -f value -c id)
openstack server create pxe-server \
--image almalinux-10 \
--flavor m1.small \
--key-name mykey \
--nic net-id=mgmt-net \
--nic port-id=$SRV_PORT \
--security-group pxe-sg \
--wait
```

Assign a floating IP to the VM so you SSH into it:

```bash
#I'm using external_net here, change it you public facing network
FIP=$(openstack floating ip create external_net -f value -c name)
openstack server add floating ip pxe-server "$FIP"
```

Now SSH into the VM, using the floating IP:

```bash
# Example
ssh almalinux@<floating-ip>
```

If using neutron dhcp agent, eth1 should have an IP already assigned:

```bash
[root@pxe-server ~]# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
eth0             UP             10.128.2.56/16 fe80::f816:3eff:fe54:4d3f/64 
eth1             UP             10.77.0.10/24 fe80::f816:3eff:fe49:7405/64 
```

Configure the second Interface For PXE:

> THis Step is used when using this server as DHCP - OPTION B

```bash
nmcli con mod eth1 ipv4.method manual ip4 10.77.0.10/24
nmcli con up eth1

# stop cloud-init from re-rendering network config at boot
echo 'network: {config: disabled}' | tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

dnf -y install dnsmasq
tee /etc/dnsmasq.d/pxe.conf >/dev/null <<'EOF'
port=0                        # DHCP only, no DNS
interface=eth1
bind-interfaces
dhcp-authoritative
dhcp-range=10.77.0.100,10.77.0.200,255.255.255.0,12h
dhcp-option=option:router     # empty = send no gateway (dnsmasq would otherwise advertise itself)
dhcp-userclass=set:ipxe,iPXE
dhcp-boot=tag:ipxe,http://10.77.0.10/netboot/boot.ipxe
log-dhcp
EOF

dnsmasq --test && systemctl enable --now dnsmasq
# if firewalld is running: firewall-cmd --permanent --add-service=dhcp && firewall-cmd --reload
```

```bash
dnf -y install nginx
systemctl enable --now nginx
# If firewalld is running: firewall-cmd --add-service=http --permanent && firewall-cmd --reload

# Install tree = the ISO, loop-mounted with a label nginx can read under SELinux
# curl -fL -o /var/tmp/alma10-minimal.iso \
#   https://repo.almalinux.org/almalinux/10/isos/x86_64/AlmaLinux-10-latest-x86_64-minimal.iso
# mkdir -p /usr/share/nginx/html/alma10
# mount -o loop,ro,context=system_u:object_r:httpd_sys_content_t:s0 \
#   /var/tmp/alma10-minimal.iso /usr/share/nginx/html/alma10
# ls /usr/share/nginx/html/alma10/images/pxeboot/   # expect vmlinuz and initrd.img
```

See my other script on how to create minimal testing netboot image <https://github.com/noslin005/netboot-luks/blob/main/build-el10-netboot.sh>

```bash
# Test 
curl -sI http://10.77.0.10/netboot/boot.ipxe | head -1
curl -sI http://10.77.0.10/netboot/vmlinuz | head -1
```

## Create PXE Client Image

- OPTION 1: RAW disk image

Run this on the deployment node:

```bash
# 1. Empty disk with GPT + EFI System Partition
dd if=/dev/zero of=pxeboot-uefi.img bs=1M count=64
sgdisk -o -n 1:2048:0 -t 1:EF00 -c 1:"EFI System" pxeboot-uefi.img

# 2. Attach, format, mount
LOOP=$(losetup --find --show --partscan pxeboot-uefi.img)
mkfs.vfat -F 32 -n EFI ${LOOP}p1
mount ${LOOP}p1 /mnt

# 3. Install iPXE as the removable-media boot loader
mkdir -p /mnt/EFI/BOOT
wget -O /mnt/EFI/BOOT/BOOTX64.EFI https://boot.ipxe.org/x86_64-efi/ipxe.efi 
# empty autoextec.ipxe
: > /mnt/EFI/BOOT/autoexec.ipxe
umount /mnt
losetup -d ${LOOP}

# 4. Upload the image to glance
openstack image create ipxe-uefi \
  --file pxeboot-uefi.img \
  --disk-format raw \
  --container-format bare \
  --property hw_firmware_type=uefi --property hw_machine_type=q35 \
  --public

## Create PXE Client
CLI_PORT=$(openstack port show pxe-client-port -f value -c id)
openstack server create pxe-client \
--image ipxe-uefi \
--flavor m1.small \
--nic port-id=$CLI_PORT \
--wait \
-c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name -f value 
```

- OPTION 2: Bootable ISO

```bash
# Use the iPXE iso
wget -O ipxe.iso https://boot.ipxe.org/ipxe.iso
openstack image create ipxe-iso-uefi --file ipxe.iso --disk-format iso --container-format bare --public

# Ensure this image boots UEFI.
openstack image set ipxe-iso-uefi --property hw_firmware-typ=uefi --property hw_system_type=q35

# Delete previews created instance
openstack server delete pxe-client


CLI_PORT=$(openstack port show pxe-client-port -f value -c id)
openstack server create pxe-client \
--image ipxe-iso-uefi \
--flavor m1.small \
--nic port-id=$CLI_PORT \
-wait \
-c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name -f value 
```

## TODO

Explore creating different pxe flavors, example:

> NOTE: Setting the root disk to 0 uses the native base image size

```bash
# 1 vCPU, 512 MB RAM, 0 GB disk
openstack flavor create pxe.tiny \
  --vcpus 1 \
  --ram 512 \
  --disk 0 \
  --ephemeral 0 \
  --swap 0 \
  --public

# 2 vCPU, 2 GB RAM, 0GB disk
openstack flavor create pxe.small \
  --vcpus 2 \
  --ram 2048 \
  --disk 0 \
  --ephemeral 0 \
  --swap 0 \
  --public

# 4 vCPU, 4 GB RAM, 0GB disk
openstack flavor create pxe.medium \
  --vcpus 4 \
  --ram 4096 \
  --disk 0 \
  --ephemeral 0 \
  --swap 0 \
  --public
```

## Console Test

```bash
# The command below will show where the instance was created.
openstack server show pxe-client -c OS-EXT-SRV-ATTR:host -c OS-EXT-SRV-ATTR:instance_name -f value

openstack console log show pxe-client

# SSH to that node, ensure ibvirt-client is installer (sudo dnf install libvirt-client) and then:
virsh list
virsh reset <instance-name>
virsh console <instance-name>
# You should be able to see the VM booting from PXE using the iPXE bootable disk.
```

## Cleanup

```bash
# Server
openstack server delete pxe-server # optional
openstack port delete pxe-server-port

# Client
openstack server delete pxe-client 
openstack port delete pxe-client-port 

# Security Group
openstack security group delete pxe-sg

# Network and subnet
openstack subnet delete pxe-subnet && openstack network delete pxe-net

# Image
openstack image delete ipxe-uefi   # option
openstack image delete ipxe-iso-uefi   
```
