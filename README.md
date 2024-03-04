# Remote Disk Encryption Initramfs Hooks
This guide provides step-by-step instructions for setting up Remote Disk Encryption on  Kali Linux.

# Preparation
## 1. Install dropbear-initramfs
To initiate the setup process, ensure that your system's packages are up to date and install the **dropbear-initramfs** package.

```bash
apt update
apt install dropbear-initramfs
```

## 2. Remove cryptsetup-nuke-password
To address issues with "cryptroot-unlock" encountering a "Timeout reached while waiting for askpass" during startup, it is mandatory to remove the **cryptsetup-nuke-password** package since it is installed by default on Kali Linux.

Cyptsetup-nuke-password replaces `/lib/cryptsetup/askpass` with a script that calls the original ‘askpass’ binary (renamed to `/lib/cryptsetup/askpass.cryptsetup`),
and erases the LUKS header if its digest value matches a special “nuke” hash.

```bash
apt remove cryptsetup-nuke-password
```

# Configure dropbear-initramfs
## 1. Modify dropbear config
Edit the configuration file located at `/etc/dropbear/initramfs/dropbear.conf` and adjust the **DROPBEAR_OPTIONS** according to the specified requirements.

```bash
DROPBEAR_OPTIONS="-I 600 -j -k -p 2222 -s -c cryptroot-unlock"
```

**In this instance, the following parameters are necessary:**
- `-I 600`: Disconnect session if no traffic is transmitted or received for 600 seconds.
- `-j`: Disable local port forwarding.
- `-k`: Disable remote port forwarding.
- `-p 2222`: Set Dropbear to listen on port 2222.
- `-s`: Disable password logins.
- `-c cryptroot-unlock` Automatically execute `cryptroot-unlock` after establishing the connection.

**`/etc/dropbear/initramfs/dropbear.conf` should look like this:**
```bash
#
# Configuration options for the dropbear-initramfs boot scripts.
# Variable assignment follow shell semantics and escaping/quoting rules.
# You must run update-initramfs(8) to effect changes to this file (like
# for other files in the '/etc/dropbear/initramfs' directory).

#
# Command line options to pass to dropbear(8)
#
DROPBEAR_OPTIONS="-I 600 -j -k -p 2222 -s -c cryptroot-unlock"

#
# On local (non-NFS) mounts, interfaces matching this pattern are
# brought down before exiting the ramdisk to avoid dirty network
# configuration in the normal kernel.
# The special value 'none' keeps all interfaces up and preserves routing
# tables and addresses.
#
#IFDOWN="*"

#
# On local (non-NFS) mounts, the network stack and dropbear are started
# asynchronously at init-premount stage.  This value specifies the
# maximum number of seconds to wait (while the network/dropbear are
# being configured) at init-bottom stage before terminating dropbear and
# bringing the network down.
# If the timeout is too short, and if the boot process is not blocking
# on user input supplied via SSHd (ie no remote unlocking), then the
# initrd might pivot to init(1) too early, thereby causing a race
# condition between network configuration from initramfs vs from the
# normal system.
#
#DROPBEAR_SHUTDOWN_TIMEOUT=60
```

## 2. Create authorized_keys file
Each user must generate a unique RSA key on their client machine using the command `ssh-keygen -t rsa -f .ssh/unlock_luks`. 

Collect and append the public keys of all users requiring access to the machine to a newly created **authorized_keys** file located in `/etc/dropbear/initramfs`. This file serves as the authentication list for the Dropbear SSH server during the early boot process, allowing specified users to remotely unlock the root disk.

```bash
touch /etc/dropbear/initramfs/authorized_keys
chmod 600 /etc/dropbear/initramfs/authorized_keys
```

# Adapt initramfs
## 1. Include tun kernel module in initramfs
Set up the system to load the tun kernel module automatically during startup. This facilitates the establishment of a VPN connection from within the BusyBox environment.

```bash
echo "tun" >> modules
```

 **`/etc/initramfs-tools/modules` should look like this:**
```bash
# List of modules that you want to include in your initramfs.
# They will be loaded at boot time in the order below.
#
# Syntax:  module_name [args ...]
#
# You must run update-initramfs(8) to effect this change.
#
# Examples:
#
# raid1
# sd_mod
bochs
tun
```

## 2. Setup networking in initramfs config
Update the networking configuration in `/etc/initramfs-tools/initramfs.conf` by adapting the relevant lines with either a **dynamic** (DHCP) configuration or a **static** IP configuration. Adjust the settings based on your network requirements.

**A) Dynamic (DHCP) Configuration**:
```bash
DEVICE=eth0  
```

**B) Static Configuration (replace values with your network details):**
```bash
DEVICE=eth0
# IP=<client-ip>::<gateway-ip>:<netmask>::<device>:<autoconf>
IP=192.168.13.37::192.168.13.1:255.255.255.0::eth0:off
```

## 3. Add hook scripts
*placeholder*

```
.
├── conf.d
│   └── resume
├── hooks
│   └── openvpn
├── initramfs.conf
├── modules
├── scripts
│   ├── init-bottom
│   │   └── postcrypt
│   ├── init-premount
│   ├── init-top
│   ├── local-bottom
│   ├── local-premount
│   ├── local-top
│   │   └── initvpn
│   ├── nfs-bottom
│   ├── nfs-premount
│   ├── nfs-top
│   └── panic
└── update-initramfs.conf
```
