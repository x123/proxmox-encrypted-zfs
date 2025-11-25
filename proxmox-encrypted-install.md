# Proxmox VE: Encrypted Root (ZFS) + Dropbear + Clevis/Tang unlocking

This guide descripbes a modified Proxmox VE Install in Debug Mode to fully
encrypt the root ZFS pool and setup dropbear based SSH unlocking and optional
clevis/tang based automated unlock based on a networked host.

**Prerequisites:** Console Access (Physical/KVM), Tang Server available via HTTP/HTTPS.

-----

## Phase 1: Modified Proxmox VE Install

*Since the Proxmox ISO does not natively support ZFS encryption in the UI, we inject parameters into the installer script during the live boot.*

### 1\. Boot into Debug Mode

1.  Boot the Proxmox VE ISO.
2.  Select **"Install Proxmox VE (Terminal UI Debug Mode)"** (usually under Advanced Options).

### 2\. Create Temporary Key (Debug Shell \#2)

1.  Exit the first debug prompt (`#1`) with `Ctrl+D` or `exit`
2.  At the **second** debug prompt (`#2`), create a temporary key:
    ```bash
    echo -n "temporarypassword" > /tmp/install.key
    ```

### 3\. Patch the Installer Script

1.  Edit the installer module:
    ```bash
    vim /usr/share/perl5/Proxmox/Install.pm
    ```
2.  Search for `zpool create`
3.  Inject the encryption arguments immediately after `create`:
      * **Before:** `my $cmd = "zpool create -f -o cachefile=none";`
      * **After:**
        ```perl
        my $cmd = "zpool create -O encryption=on -O keyformat=passphrase -O keylocation=file:///tmp/install.key -f -o cachefile=none";
        ```

### 4\. Run the Installation

1.  Type `exit` to launch the GUI.
2.  **Target Harddisk:** Select `Advanced Options` -\> `zfs`. (Encryption options will not appear in UI).
3.  **Important:** Uncheck **"Automatically reboot after successful installation"**.
4.  Install. When finished, select "Reboot Now" to exit to the 3rd debug shell.

### 5\. Finalize Keys

1.  Import the pool:
    ```bash
    zpool import -f -R /mnt rpool
    ```
2.  Unlock and rotate to real password:
    ```bash
    zfs load-key -a
    zfs change-key -o keyformat=passphrase -o keylocation=prompt rpool
    # Enter your REAL, PERMANENT passphrase now.
    ```
3.  Export and reboot:
    ```bash
    zpool export rpool
    exit
    ```

-----

## Phase 2: Environment Prep

Unlock the root filesystem at boot by entering the passphrase. Login as `root` via console or SSH.

### 1\. Configure Repositories

We disable the enterprise repositories and enable the no-subscription
repositories using the modern `.sources` format. We also remove the `quiet`
boot parameter to ensure visibility during the unlock process.

```bash
# Remove 'quiet' from boot logs
sed -i 's/quiet//' /etc/default/grub

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

### 2\. Install Dependencies

**Critical:** We use `--no-install-recommends`. By default, `clevis-initramfs`
recommends `cryptsetup-initramfs`. Installing cryptsetup on a ZFS-only system
causes build errors because it searches for missing LUKS devices.

```bash
apt install -y --no-install-recommends clevis clevis-initramfs jose jq curl iproute2 dropbear-initramfs
```

-----

## Phase 3: Setup Dropbear (SSH Fallback)

1.  **Add SSH Keys:**
    Add your public key to `/etc/dropbear/initramfs/authorized_keys`.

2.  **Configure Network:**
    Edit `/etc/initramfs-tools/initramfs.conf`.

    *Option A: DHCP*

    ```bash
    IP=dhcp
    # OR specific interface: IP=::::hostname:eno1:dhcp
    ```

    *Option B: Static*

    ```bash
    # IP=<client-ip>::<gateway>:<netmask>:<hostname>:<device>:<autoconf>
    IP=192.168.1.10::192.168.1.1:255.255.255.0:pve:eno1
    ```

3.  **Update initramfs/sync boot:**

    ```bash
    update-initramfs -u -k all && proxmox-boot-tool refresh
    reboot
    ```

4.  **Reboot and test dropbear unlock:**

    Reboot the system, and ensure that dropbear starts. At console you should
    see something like:

    ```
    IP-Config: ens18 complete (dhcp from 192.168.1.1)
     address: 192.168.1.227 broadcast: 192.168.1.255    netmask: 255.255.255.0
     ...
    Begin: Starting dropbear ...
    Begin: Importing ZFS root pool 'rpool' ... done.
    Begin: Importing pool 'rpool' using defaults ... done.
    Enter passphrase for 'rpool':
    ```

    Upon SSH into dropbear, `zfsunlock` will allow you to unlock the pool:

    ```bash
    Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
    Warning: Permanently added '192.168.1.227' (ED25519) to the list of known hosts.

    BusyBox v1.37.0 (Debian 1:1.37.0-6+b3) built-in shell (ash)
    Enter 'help' for a list of built-in commands.

    ~ # zfsunlock

    Unlocking encrypted ZFS filesystems...
    Enter the password or press Ctrl-C to exit.

    🔐 Encrypted ZFS password for rpool: (press TAB for no echo)
    Password for rpool accepted.
    Unlocking complete.  Resuming boot sequence...
    Please reconnect in a while.
    ```

-----

## Phase 4: Setup Clevis & Tang

1.  **Trust Tang CA (If private/self-signed):**
    Copy CA cert to `/usr/local/share/ca-certificates/` and run `update-ca-certificates`.

2.  **Generate JWE Token:**

    ```bash
    mkdir -p /etc/zfs
	```

    **Choose your configuration strategy below:**

    ### Option A: Single Tang Server (Standard)

    Simple setup. If this tang server is unreachable, manual password entry is required.

    ```bash
    read -rs -p "ZFS Root Passphrase: " mypass && echo && \
    printf "%s" "$mypass" | clevis encrypt tang '{"url": "https://tang.example.com"}' > /etc/zfs/root.jwe && \
    unset mypass
    ```

    ### Option B: SSS (High Availability / Redundancy)

	Uses Shamir's Secret Sharing. Defined by a threshold `t` (how many keys
	are needed) and a list of `pins`. See `clevis-encrypt-ss` manpage for more
	details on SSS

    *Example: Define 2 Tang servers with a threshold of 1 (Logic: Tang1 OR Tang2).*

    ```bash
    # Define the SSS policy JSON
    # t=1 means only 1 of the defined servers needs to respond to unlock.
    CFG='{"t": 1, "pins": {"tang": [{"url": "https://tang1.example.com"}, {"url": "https://tang2.example.com"}]}}'

    # Encrypt
    read -rs -p "ZFS Root Passphrase: " mypass && echo && \
    printf "%s" "$mypass" | clevis encrypt sss "$CFG" > /etc/zfs/root.jwe && \
    unset mypass
    ```

3.  **Secure the File:**

    ```bash
    chmod 600 /etc/zfs/root.jwe
    ```

-----

## Phase 5: The Integration Hook

This hook injects a script into the native ZFS boot process (`initramfs-tools-load-key`), ensuring Clevis runs at the exact moment ZFS expects a key.

**Create File:** `/etc/initramfs-tools/hooks/zfs-clevis`, using the contents
from [hooks/zfs-clevis](hooks/zfs-clevis)

```bash
wget -q -O /etc/initramfs-tools/hooks/zfs-clevis https://github.com/x123/proxmox-encrypted-zfs/raw/refs/heads/master/hooks/zfs-clevis
chmod +x /etc/initramfs-tools/hooks/zfs-clevis
```

**Update initramfs / sync bootloader:**

```bash
update-initramfs -u -k all && proxmox-boot-tool refresh
```

**Reboot & Test:**

As long as the tang server(s) are available, the system should boot cleanly
without prompting for a passphrase. If the tang server is unavailable, it will
fall back to the console and dropbear based zfsunlock workflow.

-----

## Maintenance: Tang server key rotation

On the tang server, use `tangd-rotate-keys` to generate a new set of keys. The
old ones will be renamed with a `.` prefixing their names. At this point, both
the new keys and the old ones are still available to any clients requesting
decryption, but tang will advertise the new key for any clients wanting to
encrypt.

You should now follow the guide below to update any active clients that should
maintain automatic `clevis` based unlocking *before* removing the legacy keys.

Once all desired clients have had their client JWE tokens updated to the new
key, you can then remove the legacy keys (or set their perms to `0000`).

-----

## Maintenance: Key Rotation and Updating the JWE Token on the client-side

*Perform this step when the Tang server has advertised new keys, but *before*
the old keys are deleted/rotated out of the Tang database on the server.*

### 1\. Re-encrypt the JWE Token

We must decrypt the existing token (using the old keys) and immediately re-encrypt it against the current policies.

1.  **Regenerate the Token:**
    *Use the command corresponding to your setup in Phase 4.*

    **Option A: Single Tang Server**

    ```bash
    clevis decrypt < /etc/zfs/root.jwe | \
    clevis encrypt tang '{"url": "https://tang.example.com"}' > /etc/zfs/root.jwe.new
    ```

    **Option B: SSS (High Availability)**
    *Ensure your JSON config matches your original policy.*

    ```bash
    CFG='{"t": 1, "pins": {"tang": [{"url": "https://tang1.example.com"}, {"url": "https://tang2.example.com"}]}}'

    clevis decrypt < /etc/zfs/root.jwe | \
    clevis encrypt sss "$CFG" > /etc/zfs/root.jwe.new
    ```

2.  **Verify and Swap:**
    Ensure the new file was created successfully before overwriting the old one.

    ```bash
    if [ -s /etc/zfs/root.jwe.new ]; then
        mv /etc/zfs/root.jwe.new /etc/zfs/root.jwe
        chmod 600 /etc/zfs/root.jwe
        echo "Success: JWE updated."
    else
        echo "Error: Re-encryption failed. Check connectivity to ALL Tang servers."
        rm /etc/zfs/root.jwe.new
    fi
    ```

### 2\. Update Bootloader

**Critical Step:** The `root.jwe` file is copied into the initramfs image at build time. Updating the file in `/etc/zfs` is not enough; you must rebuild the boot image or the system will continue to use the old key configuration.

```bash
update-initramfs -u -k all && proxmox-boot-tool refresh
```
