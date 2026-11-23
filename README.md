# Kolla-Ansible Multinode OpenStack on Rocky Linux 10

## 1. Proposed architecture

Nodes:

- **NUC (**`10.7.0.10`**)**: Deployment server only. Run Ansible and Kolla-Ansible here.
- **node01 (**`10.7.0.11`**)**: Control plane + network + storage.
- **node02 (**`10.7.0.12`**)**: Compute + storage.
- **node03 (**`10.7.0.13`**)**: Compute + storage.
- **1 Gb NIC - eth1g**: Management and API access.
- **2.5 Gb NIC - eth2g **: OpenStack tenant/provider networking and optionally storage traffic.
- **NVMe disks**: Cinder LVM backend initially.

| Host   | OpenStack roles                          | RAM   |
| ------ | ---------------------------------------- | ----- |
| nuc01  | Deployment only                          | 16 GB |
| node01 | Controller, Network, Monitoring, Storage | 32 GB |
| node02 | Compute, Storage                         | 16 GB |
| node03 | Compute, Storage                         | 16 GB |




## 2. Prepare the NUC deployment server

On `nuc01`, install the required packages.

```bash
sudo dnf install -y \
  git \
  python3 \
  python3-devel \
  gcc \
  openssl-devel \
  libffi-devel \
  python3-libselinux
```

- Install Kolla-Ansible:

```bash
pip install -U pip
git clone --branch stable/2026.2 https://opendev.org/openstack/kolla-ansible
pip install -e ~/kolla-ansible
# pip install git+https://opendev.org/openstack/kolla-ansible@stable/2026.2
```

- Install Ansible Galaxy dependencies:

```bash
# Exit the shell and log back in
kolla-ansible install-deps
```

- Ansible inventory - multinode:

This is where you define which hosts run which OpenStack services.

```bash
cp ~/.local/share/kolla-ansible/ansible/inventory/multinode /etc/kolla/ansible/inventory/multinode
```

Change the following groups:

```yml
[control]
node01

[network]
node01

[compute]
node02
node03

[monitoring]
node01

[storage]
node01
node02
node03

[deployment]
localhost       ansible_connection=local
```

- Kolla globals.yml:

This is the central Kolla configuration file.

```bash
cp ~/.local/share/kolla-ansible/etc_examples/kolla/globals.yml /etc/kolla/globals.yml
```

Change the following lines:

```yml
workaround_ansible_issue_8743: true
kolla_base_distro: "rocky"
kolla_internal_vip_address: "10.7.0.254"
kolla_container_engine: podman
network_interface: "eth1g"
network_address_family: "ipv4"
neutron_external_interface: "eth2g"
neutron_plugin_agent: "openvswitch"
enable_cinder_backup: true
enable_cinder_backend_lvm: true
cinder_cluster_name: "open-cluster"
enable_neutron_trunk: true
kolla_test_images: "yes"
```
- Configure Kolla passwords

Generate your password file:

```bash
kolla-genpwd
# This generates passwords in:
/etc/kolla/passwords.yml
# Protect it:
chmod 600 /etc/kolla/passwords.yml
```

## 3. SSH access and host preparation

Configure passwordless SSH from the NUC to your Dell nodes.

```bash
ssh-keygen -t ed25519
ssh-copy-id stackadmin@10.7.0.11
ssh-copy-id stackadmin@10.7.0.12
ssh-copy-id stackadmin@10.7.0.13
```

Test:

```bash
ssh stackadmin@node01 hostname
ssh stackadmin@node02 hostname
ssh stackadmin@node03 hostname
```

Configure sudo access for Ansible:

```bash
sudo visudo
```

Add:

```bash
stackadmin ALL=(ALL) NOPASSWD: ALL
```

Ensure each node has:

- A static management IP.
- A unique hostname.
- Working forward and reverse name resolution, or at minimum consistent `/etc/hosts`.
- SSH access from nuc01.
- Synchronized time.
- Internet access for pulling container images.
- Sufficient disk space.
- The eth2g (2.5 Gb) NIC configured consistently.

Example `/etc/hosts` on all four hosts:

```bash
10.7.0.10 nuc01
10.7.0.11 node01
10.7.0.12 node02
10.7.0.13 node03
10.7.0.254 cloud.example.com
```

You should also verify:

```bash
hostnamectl
ip -br addr
ip route
timedatectl
```

For the eth2g run:

```bash
nmcli con modify eth2g ipv4.method disable ipv6.method disabled connection.autoconnect yes
nmcli con up eth2g
```


## 3. Cinder LVM backend

Configure cinder lvm volumes on all storage nodes:

```bash
lsblk
sudo pvcreate /dev/nvme0n1
sudo vgcreate cinder-volumes /dev/nvme0n1
```


## 4. Validate inventory before deployment

Before executing that command, ensure your inventory is complete and your hosts are prepared.

Inspect the inventory:

```bash
ansible-inventory -i /etc/kolla/ansible/inventory/multinode --graph
```

And inspect effective variables:

```bash
ansible-inventory -i /etc/kolla/ansible/inventory/multinode --host node01
```

Then validate:

```bash
ansible -i /etc/kolla/ansible/inventory/multinode all -m ping
```

## 5. Deployment sequence

The actual deployment should follow this order.

```bash
# 1. Bootstrap remote servers (~5-10 minutes)
kolla-ansible bootstrap-servers -i /etc/kolla/ansible/inventory/multinode


# 2. Validate deployment prerequisites (~5 minutes)
kolla-ansible prechecks -i /etc/kolla/ansible/inventory/multinode


# 3. Pull container images (~5 minutes)
kolla-ansible pull -i /etc/kolla/ansible/inventory/multinode


# 4. Deploy OpenStack (~15-25 minutes)
kolla-ansible deploy -i /etc/kolla/ansible/inventory/multinode

# 5. Generate OpenStack client configuration
kolla-ansible post-deploy -i /etc/kolla/ansible/inventory/multinode
```

Kolla-Ansible documents these deployment stages and their corresponding commands.

## Apendix

- Kickstart

```bash
#version=RHEL10

# ---- General ---------------------------------------------------------------
text
reboot
eula --agreed
firstboot --disable
skipx

lang en_US.UTF-8
keyboard --vckeymap=us --xlayouts='us'
timezone America/New_York --utc

# ---- Install source (swap for a local mirror if you have one) --------------
url --url="https://dl.rockylinux.org/pub/rocky/10/BaseOS/x86_64/os/"
repo --name="AppStream" --baseurl="https://dl.rockylinux.org/pub/rocky/10/AppStream/x86_64/os/"
repo --name="CRB"       --baseurl="https://dl.rockylinux.org/pub/rocky/10/CRB/x86_64/os/"

# ---- Network ---------------------------------------------------------------
# DHCP on the first NIC with link.
network --bootproto=dhcp --device=link --activate --onboot=yes

# ---- Security --------------------------------------------------------------
# Kolla-Ansible recommends firewalld off;
firewall --disabled
# RHEL 10 can no longer fully disable SELinux via config, so use permissive.
selinux --permissive
authselect --enableshadow --passalgo=sha512

rootpw --lock
user --name=stackadmin --groups=wheel --gecos="System Administrator" --iscrypted --password=<encrypted-password>
sshkey --username=stackadmin "pubkey here"

services --enabled="chronyd,sshd,NetworkManager"

# ---- Disk detection --------------------------------------------------------
ignoredisk --only-use=sda
zerombr
clearpart --all --initlabel --drives=sda
bootloader --boot-drive=sda

# ---- Partitioning (UEFI + LVM) --------------------------------------------
# Needs roughly 165 GB. Space left over in vg_rl stays unallocated so you can
# carve out cinder-volumes / extra LVs later.
part /boot/efi --fstype="efi"   --size=600 --fsoptions="umask=0077,shortname=winnt"
part /boot     --fstype="xfs"   --size=1024
part pv.01     --fstype="lvmpv" --size=1 --grow

volgroup vg_rl --pesize=4096 pv.01
logvol /    --fstype="xfs" --size=40960  --name=lv_root --vgname=vg_rl
logvol /var --fstype="xfs" --size=122880 --name=lv_var  --vgname=vg_rl

# ---- Packages --------------------------------------------------------------
%packages
@^minimal-environment
openssh-server
chrony
dnf-plugins-core
lvm2
git
tmux
vim-enhanced
bash-completion
curl
wget
tar
rsync
lsof
pciutils
net-tools
# Build deps for Kolla-Ansible / OpenStack-Ansible virtualenvs
python3
python3-pip
python3-devel
python3-libselinux
gcc
libffi-devel
openssl-devel
# Compressed swap in RAM
zram-generator-defaults
%end

# ---- Post-install ----------------------------------------------------------
%post --log=/root/ks-post.log

# Enable CRB on the installed system (devel packages live there)
dnf config-manager --set-enabled crb || true

# Passwordless sudo for the deployment user
echo "stackadmin ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/stackadmin
chmod 0440 /etc/sudoers.d/stackadmin

# zram swap (4 GiB, zstd)
mkdir -p /etc/systemd/zram-generator.conf.d
cat > /etc/systemd/zram-generator.conf.d/openstack.conf <<'EOF'
[zram0]
zram-size = 4096
compression-algorithm = zstd
swap-priority = 100
EOF

cat > /etc/sysctl.d/90-swap.conf <<'EOF'
vm.swappiness = 60
vm.page-cluster = 0
EOF

# 4 GiB on-disk swapfile as an OOM safety net (lower priority than zram)
dd if=/dev/zero of=/swapfile bs=1M count=4096 status=none
chmod 0600 /swapfile
mkswap /swapfile
echo '/swapfile none swap defaults,pri=10 0 0' >> /etc/fstab

# Stable NIC names by PCI address (adjust to match your hardware)
cat > /etc/udev/rules.d/70-net.rules <<'EOF'
ACTION!="remove", SUBSYSTEM=="net", SUBSYSTEMS=="pci", KERNELS=="0000:02:00.0", NAME="eth2g"
ACTION!="remove", SUBSYSTEM=="net", SUBSYSTEMS=="pci", KERNELS=="0000:00:1f.6", NAME="eth1g"
EOF

%end
```

- PXE Container

```bash

cat <<'EOF' > entrypoint.sh
#!/bin/sh
set -ex
nginx
exec dnsmasq --keep-in-foreground --log-dhcp --log-facility=- --conf-file=/etc/pxe/pxe.conf
EOF

cat <<'EOF' > grub.cfg
set default=0
set timeout=5

menuentry 'Install Rocky Linux 10 (kickstart, minimal)' {
    linux  /rocky10/vmlinuz \
        inst.repo=https://dl.rockylinux.org/pub/rocky/10/BaseOS/x86_64/os \
        inst.stage2=https://dl.rockylinux.org/pub/rocky/10/BaseOS/x86_64/os/ \
        inst.ks=http://${net_default_server}/ks.cfg \
        ip=dhcp rd.neednet=1 inst.wait_for_disks=0 rd.timeout=10 \
        inst.sshd
    initrd /rocky10/initrd.img
}
EOF

cat <<'EOF' >nginx.conf
daemon on;
pid /run/nginx.pid;
error_log /dev/stderr;
events {}
http {
  access_log /dev/stdout;
  include /etc/nginx/mime.types;
  server {
    listen 80 default_server;
    root /srv/www;
    autoindex on;
  }
}
EOF

cat <<'EOF' > pxe.conf
# Network
bind-dynamic
except-interface=lo
except-interface=virbr0
except-interface=podman0
except-interface=docker0

# DHCP + TFTP only, no DNS (avoids clashing with systemd-resolved/libvirt)
port=0
enable-tftp
tftp-root=/srv/tftp

# DHCP pool
dhcp-range=10.7.0.160,10.7.0.168,12h

# DHCP options
dhcp-option=option:router,10.7.0.1
dhcp-option=option:dns-server,10.7.0.1

# Static Reservations
dhcp-host=<replace-with-eth1g-mac>,10.7.0.10,nuc,12h
dhcp-host=<replace-with-eth1g-mac>,10.7.0.11,node01,12h
dhcp-host=<replace-with-eth1g-mac>,10.7.0.12,node02,12h
dhcp-host=<replace-with-eth1g-mac>,10.7.0.13,node03,12h

# UEFI x86-64 clients (arch 7 and 9)
dhcp-match=set:efi64,option:client-arch,7
dhcp-match=set:efi64,option:client-arch,9
dhcp-boot=tag:efi64,shimx64.efi,,10.7.0.10
EOF

cat <<EOF > Containerfile
FROM quay.io/rockylinux/rockylinux:10
RUN dnf -y install dnsmasq nginx shim-x64 grub2-efi-x64 \
    && dnf clean all \
    && mkdir -p /srv/tftp /srv/www \
    && cp /boot/efi/EFI/*/shimx64.efi /boot/efi/EFI/*/grubx64.efi /srv/tftp/ \
    && chmod -R a+rX /srv
COPY nginx.conf /etc/nginx/nginx.conf
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
EOF

cat <<'EOF' > fetch.sh
#!/bin/sh
# Downloads the Rocky 10 PXE kernel + initrd into ./data/pxeboot
set -ex
cd "$(dirname "$0")"
BASE=${BASE:-https://download.rockylinux.org/pub/rocky/10/BaseOS/x86_64/os}
mkdir -p data/pxeboot
for f in vmlinuz initrd.img; do
  curl -fL -o data/pxeboot/$f "$BASE/images/pxeboot/$f"
done
ls -lh data/pxeboot
EOF

cat <<'EOF' > run.sh
#!/bin/sh
# First time: ./fetch.sh   then   ./run.sh
set -xeauo pipefail
cd "$(dirname "$0")"
bash ./fetch.sh
podman build -t pxe .
podman rm pxe --force || true
podman run \
        -d \
        --name pxe \
        --network host \
        --cap-add NET_ADMIN \
        --cap-add NET_RAW \
        --security-opt label=disable \
        -v "$PWD/pxe.conf:/etc/pxe/pxe.conf:ro" \
        -v "$PWD/grub.cfg:/srv/tftp/grub.cfg:ro" \
        -v "$PWD/ks.cfg:/srv/www/ks.cfg:ro" \
        -v "$PWD/data/pxeboot:/srv/tftp/rocky10:ro" \
        pxe
EOF

firewall-cmd --add-service={dns,http,https,ssh,tftp} --permanent
firewall-cmd --reload
```
