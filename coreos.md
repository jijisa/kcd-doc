# CoreOS

## Install CoreOS from dom0
1. Create qcow2 vm image.
* qemu-img create -f qcow2 coreos1.qcow2 20G
2. Load nbd module.
* modprobe nbd
3. Connect coreos1.qcow2 to /dev/nbd0
* qemu-nbd -c /dev/nbd0 coreos1.qcow2
4. Get coreos-install script.
* wget https://raw.github.com/coreos/init/master/bin/coreos-install
5. Install CoreOS into /dev/nbd0 with xen OEM and stable channel.
* ./coreos-install -v -d /dev/nbd0 -o xen -C stable
Downloading the signature for http://stable.release.core-os.net/amd64-usr/current/coreos_production_xen_image.bin.bz2...
2015-10-10 14:55:21 URL:http://stable.release.core-os.net/amd64-usr/current/coreos_production_xen_image.bin.bz2.sig [543/543] -> "/tmp/coreos-install.2JNPWtAxGH/coreos_production_xen_image.bin.bz2.sig" [1]
Downloading, writing and verifying coreos_production_xen_image.bin.bz2...
2015-10-10 14:56:14 URL:http://stable.release.core-os.net/amd64-usr/current/coreos_production_xen_image.bin.bz2 [195077663/195077663] -> "-" [1]
gpg: Signature made 2015년 09월 17일 (목)  using RSA key ID 1CB5FA26
gpg: key 93D2DCB4 marked as ultimately trusted
gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
gpg: Good signature from "CoreOS Buildbot (Offical Builds) <buildbot@coreos.com>"
Success! CoreOS stable current (xen) is installed on /dev/nbd0


6. Disconnect /dev/nbd0
* qemu-nbd -d /dev/nbd0


## Install CoreOS domu using iso

1. Create qcow2 vm image.
* qemu-img create -f qcow2 coreos2.qcow2 20G
2. Get CoreOS iso file.
* wget http://beta.release.core-os.net/amd64-usr/current/coreos_production_iso_image.iso
3. Create coreos2.cfg xen config file.
* Network connection is needed.
4. Boot coreos2.qcow2 
* xl create coreos2.cfg
5. Connect vnc port of coreos2.
6. Install CoreOS to /dev/xvda with xen OEM type and stable channel
* sudo coreos-install -d /dev/xvda -o xen -C stable
7. Shutdown coreos2.

## Post installation
CoreOS has only two users - root and core.
Both users passwords are disabled by default.
So if you boot up CoreOS, you cannot login.
There are two ways to login - enable password or put ssh public key.
To do so, you have to mount root filesystem of coreos first.

Mount coreos image
* qemu-nbd -c /dev/nbd0 coreos1.qcow2
* kpartx -av /dev/nbd0
* mount /dev/mapper/nbd0p9 /mnt

Enable core's password
1. Create SHA-512 password for core.
* python -c "import crypt, random, string; print crypt.crypt(raw_input('password: '), '\$6\$' + ''.join([random.choice(string.ascii_letters + string.digits) for _ in range(16)]))"
2.  Replace the second column with the output string.
* vi /mnt/etc/shadow
core:*:15887:0::::: 

Put ssh public key
1. Generate ssh key pair.
* ssh-keygen
2. Put ssh public key into core's .ssh/authorized_keys.
* cp .ssh/id_rsa.pub /mnt/home/core/.ssh/authorized_keys
3. Set the ownership of authorized_keys to core uid(500)
* chown 500:500 /mnt/home/core/.ssh/authorized_keys

Unmount coreos image
* umount /mnt
* kpartx -dv /dev/nbd0
* qemu-nbd -d /dev/nbd0


## Network configuration with networkd
* vi /etc/systemd/network/static.network
[Match]
Name=eth0

[Network]
Address=192.168.24.12/24
Gateway=192.168.24.1

* sudo systemctl restart systemd-networkd

## Hostname configuration with hostnamectl
* sudo hostnamectl set-hostname coreos2
* hostnamectl

## Setting up coreos when installing using cloud-init yaml file.

If you want to insert cloud-init config during boot, 
* ./coreos-install -v -d /dev/nbd0 -o xen -C stable -c coreos1-cloudinit.yml

coreos-cloudinit.yml is in the same dir as this document.
Refer to the file.
After installation finished, if you boot coreos1, 
coreos1-cloudinit.yml can be found in /var/lib/coreos-install/user_data file.
If you modify user_data file, the modified values are honored at the next boot.

## CoreOS Components
* ETCD: Service Discovery Key Value Store(KVS)
  - etcdctl set /message "Hi, Heechul"
  - etcdctl ls / --recursive
  - etcdctl get /message
* Docker: Container Management
* Fleet: Cluster-aware process management
  - fleetctl list-[machines|units|unit-files]
  - fleetctl sumbit -> load -> start [unit_file].service
  - fleetctl stop -> unload -> destroy [unit_file].service
  - fleetctl [ssh|status|journal] [unit_file].service
* Flannel: virtual network for use with container.
* cloudinit: provisioning tool for CoreOS with cloud-config yaml format
* Ignition: another provisioning tool for CoreOS with json format.

## Build my own etcd server
etcd server is for clustering coreos nodes.
etcd server can be run as a docker container.


## References

