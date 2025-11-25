# Proxmox VE: Encrypted Root with Remote Unlock

**ZFS Encryption • Dropbear SSH • Clevis/Tang Automation**

This guide details a modified Proxmox VE installation in Debug Mode. It
achieves a fully encrypted ZFS root pool that can be unlocked remotely via SSH
(Dropbear) or automatically via a networked key server (Clevis/Tang).

**Prerequisites:**

  * Console Access (Physical/KVM)
  * Tang Server available via HTTP/HTTPS (for automation phase)

-----

## Chapter 1: The Installation

Since the Proxmox ISO does not natively support ZFS encryption in the UI, we
must inject parameters into the installer script during the live boot.

### 1\. Boot into Debug Mode

Boot the Proxmox VE ISO and select **"Install Proxmox VE (Terminal UI Debug
Mode)"**.

### 2\. Create a Temporary Key

The installer needs a file to read the encryption key from. We will create a
temporary key in the second debug shell.

1.  Exit the first debug prompt (`#1`) by pressing `Ctrl+D` or typing `exit`.
2.  At the **second** debug prompt (`#2`), run:

<!-- end list -->

```bash
echo -n "temporarypassword" > /tmp/install.key
```

### 3\. Patch the Installer Script

We need to modify the Proxmox installer perl module to include the ZFS
encryption flags.

Open the file:

```bash
vim /usr/share/perl5/Proxmox/Install.pm
```

Search for the text `zpool create`. You will find a line defining the creation
command. Replace that line with the version below, which injects
`encryption=on`, `keyformat=passphrase`, and points to our temporary key file.

```perl
# /usr/share/perl5/Proxmox/Install.pm

# ...
# OLD LINE:
# my $cmd = "zpool create -f -o cachefile=none";

# NEW LINE:
my $cmd = "zpool create -O encryption=on -O keyformat=passphrase -O keylocation=file:///tmp/install.key -f -o cachefile=none";
# ...
```

### 4\. Run the Installation

Type `exit` to launch the GUI installer and proceed with the following specific
settings:

  * **Target Harddisk:** Select `Advanced Options` -\> `zfs`. (Note: Encryption options will *not* appear here, but they are now injected in the backend).
  * **Reboot:** Uncheck **"Automatically reboot after successful installation"**.

When the installation finishes, select "Reboot Now" to exit to the 3rd debug
shell.

### 5\. Finalize and Rotate Keys

Before the system boots for the first time, we must switch from the temporary
file-based key to a real user-entered passphrase.

Import the pool:

```bash
zpool import -f -R /mnt rpool
```

Load the temporary key, then rotate to your real, permanent passphrase:

```bash
zfs load-key -a
zfs change-key -o keyformat=passphrase -o keylocation=prompt rpool
# Enter your REAL, PERMANENT passphrase now.
```

Export the pool and reboot:

```bash
zpool export rpool
exit
```

-----

## Chapter 2: System Configuration

Unlock the root filesystem at boot by entering your passphrase. Login as `root`
via console or SSH.

### Configure Repositories

We will disable enterprise repositories, enable the no-subscription
repositories, remove the `quiet` boot parameter to ensure we can see the
unlock prompt, and update the system before continuing.

*Note: you can do this part via the Proxmox web GUI instead*

```bash
# Remove 'quiet' from boot logs
sed -i '/^GRUB_CMDLINE_LINUX_DEFAULT=/ s/quiet//' /etc/default/grub

# Disable existing enterprise entries
echo "Enabled: false" >> /etc/apt/sources.list.d/pve-enterprise.sources
echo "Enabled: false" >> /etc/apt/sources.list.d/ceph.sources

# Update ceph to no-subscription
cat <<EOF | tee -a /etc/apt/sources.list.d/ceph.sources > /dev/null
Types: deb
URIs: http://download.proxmox.com/debian/ceph-squid
Suites: $(grep VERSION_CODENAME /etc/os-release | cut -d= -f2)
Components: no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

# Add proxmox no-subscription
cat <<EOF | tee -a /etc/apt/sources.list.d/proxmox.sources > /dev/null
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: $(grep VERSION_CODENAME /etc/os-release | cut -d= -f2)
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

apt update && apt dist-upgrade -y
```

### Install Dependencies

We use `--no-install-recommends` here. The default `clevis-initramfs` package
recommends `cryptsetup-initramfs`, which causes build errors on ZFS-only
systems (as it searches for missing LUKS devices).

```bash
apt install -y --no-install-recommends clevis clevis-initramfs jose jq curl iproute2 dropbear-initramfs
```

### Setup Dropbear (SSH Fallback)

Dropbear allows you to SSH into the initramfs environment to unlock the disk if
automation fails.

**1. Add SSH Keys**
Add your public key to the dropbear authorized keys file:
`/etc/dropbear/initramfs/authorized_keys` and optionally, to the system's root
account `/root/.ssh/authorized_keys`

**2. Configure Network**
Edit `/etc/initramfs-tools/initramfs.conf` to define how the system gets an IP
during boot.

  * **For DHCP:**
    ```bash
    IP=dhcp
    # OR specific interface: IP=::::hostname:eno1:dhcp
    ```
  * **For Static IP:**
    ```bash
    # IP=<client-ip>::<gateway>:<netmask>:<hostname>:<device>:<autoconf>
	# autoconf (i.e., `dhcp`) is not necessary, because we are defining a
	# static config
    IP=192.168.1.10::192.168.1.1:255.255.255.0:pve:eno1
    ```

**3. Apply Changes**
Update the initramfs and bootloader:

```bash
update-initramfs -u -k all && proxmox-boot-tool refresh
```

**4. Test Dropbear**
Reboot the system. You should see `Starting dropbear...` in the console.
Connect via SSH and run `zfsunlock`:

```bash
# Connect from your workstation
ssh root@192.168.1.227

# Inside the dropbear session
zfsunlock
```

-----

## Chapter 3: Automation (Clevis & Tang)

This phase sets up automatic unlocking using a remote Tang server.

### Trust Tang CA

If your Tang server uses a self-signed certificate or private CA, copy the CA
cert to `/usr/local/share/ca-certificates/` and run `update-ca-certificates`.

### Generate the JWE Token

Create the configuration directory:

```bash
mkdir -p /etc/zfs
```

Choose **one** of the following encryption strategies:

**Strategy A: Single Tang Server (Standard)**
Use this for a simple setup. If the server is unreachable, manual password
entry is required.

```bash
read -rs -p "ZFS Root Passphrase: " mypass && echo && \
printf "%s" "$mypass" | clevis encrypt tang '{"url": "https://tang.example.com"}' > /etc/zfs/root.jwe && \
unset mypass
```

**Strategy B: High Availability (SSS)**
Use this to define a threshold of keys (e.g., Tang1 OR Tang2). The config below
(`t: 1`) means only 1 server needs to be reachable.

```bash
CFG='{"t": 1, "pins": {"tang": [{"url": "https://tang1.example.com"}, {"url": "https://tang2.example.com"}]}}'

read -rs -p "ZFS Root Passphrase: " mypass && echo && \
printf "%s" "$mypass" | clevis encrypt sss "$CFG" > /etc/zfs/root.jwe && \
unset mypass
```

**Secure the Keyfile:**

```bash
chmod 600 /etc/zfs/root.jwe
```

### The Integration Hook

We need a hook to inject the Clevis unlock script into the native ZFS boot
process (`initramfs-tools-load-key`).

**1. Install Hook**
Download the [hook](hooks/zfs-clevis) to `/etc/initramfs-tools/hooks/zfs-clevis`:

```bash
wget -q -O /etc/initramfs-tools/hooks/zfs-clevis https://github.com/x123/proxmox-encrypted-zfs/raw/refs/heads/master/hooks/zfs-clevis
chmod +x /etc/initramfs-tools/hooks/zfs-clevis
```

**2. Update Bootloader**
This rebuilds the initramfs, including the new JWE token and the hook script.

```bash
update-initramfs -u -k all && proxmox-boot-tool refresh
```

**3. Reboot & Test**
As long as the Tang server is reachable, the system should now boot cleanly
without a passphrase prompt.

-----

## Appendix: Key Rotation Workflow

Perform this routine when you need to rotate keys on the Tang server.

### 1\. Server-Side Rotation

On the Tang server, generate new keys. The old keys will be renamed (prefixed
with `.`) but remain valid for decryption temporarily to allow clients to
update.

```bash
# (Run on Tang Server)
tangd-rotate-keys -d /var/lib/tang

# verify that the permissions, ownership, and key rotation happened
ls -alh /var/lib/tang/
```

### 2\. Client-Side Update

**Before** deleting old keys from the server, you must re-encrypt the client's token against the new keys.

**Step A: Regenerate Token**
Decrypt the existing token and immediately re-encrypt it against the new policy.

  * **If using Single Tang Server:**

    ```bash
    clevis decrypt < /etc/zfs/root.jwe | \
    clevis encrypt tang '{"url": "https://tang.example.com"}' > /etc/zfs/root.jwe.new
    ```

  * **If using SSS (High Availability):**
    *(Ensure `$CFG` matches your original policy)*

    ```bash
    clevis decrypt < /etc/zfs/root.jwe | \
    clevis encrypt sss "$CFG" > /etc/zfs/root.jwe.new
    ```

**Step B: Commit Changes**
Verify the new file is non-empty, replace the old one, and **update the bootloader**.

> **Critical:** If you fail to update the initramfs, the system will continue attempting to use the old key on the next boot.

```bash
if [ -s /etc/zfs/root.jwe.new ]; then
    mv /etc/zfs/root.jwe.new /etc/zfs/root.jwe
    chmod 600 /etc/zfs/root.jwe

    # Update bootloader to include the new key
    update-initramfs -u -k all && proxmox-boot-tool refresh
    echo "Success: JWE updated and bootloader refreshed."
else
    echo "Error: Re-encryption failed. Check connectivity to ALL Tang servers."
    rm /etc/zfs/root.jwe.new
fi
```
