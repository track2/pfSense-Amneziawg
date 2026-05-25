# AmneziaWG for pfSense

> Kernel module (`if_awg.ko`) and `amneziawg-tools` for pfSense — client mode setup guide.

---

## ⚠️ Warning

> **Do NOT use this module on incompatible versions of pfSense!**
> Using it on an unsupported version will almost certainly cause instability and system crashes.

---

## Installation & Configuration (Client Mode)

### Step 1 — Copy the kernel module

Copy the appropriate `if_awg.ko` for your pfSense version to `/boot/modules`.

### Step 2 — Set permissions

```sh
chmod 755 /boot/modules/if_awg.ko
```

### Step 3 — Load and verify the module

```sh
kldload -n if_awg
kldstat | grep if_awg
```

Expected output:
```
 2    1 0xffffffff8359e000    3d840 if_awg.ko
```

If the output is not empty, you can proceed.

### Step 4 — Enable autoload on boot

```sh
vi /boot/loader.conf
```

Add the following line:

```
if_awg_load="YES"
```

> **Important:** Do **not** replace your entire `/boot/loader.conf` with the file from this repository. Only add this one line to your existing file.

### Step 5 — Copy binaries and set permissions

Copy the contents of `/usr/local/bin/` to the corresponding location on your pfSense system and set the required permissions:

```sh
chmod 755 /usr/local/bin/awg
chmod 755 /usr/local/bin/amneziawg-go
rehash
```

> **Note:** `amneziawg-go` is only needed if you plan to run in **userspace** mode. It is not required for **kernelspace** operation.
>
> The `awg-quick` script is intentionally **not included** in this repository. On a firewall, automatic route assignment via `awg-quick` can be dangerous — a misconfigured file could cause you to lose remote access to your pfSense instance.

### Step 6 — Create the config directory and copy your config

```sh
mkdir -p /usr/local/etc/amnezia/amneziawg/
cp ~/awg0.conf /usr/local/etc/amnezia/amneziawg/
```

### Step 7 — Comment out `Address` and `DNS` in the config

Open your config file and comment out the `Address` and `DNS` lines (if present).
The IP address is assigned directly when the `awg0` interface is brought up (at service start).

### Step 8 — Configure the startup script

Copy the `awg` script to `/usr/local/etc/rc.d/` and edit it to match your network settings.

Example of what the relevant line should look like:

```sh
/sbin/ifconfig awg0 inet 10.0.14.88/24 up
```

Replace the IP address and subnet with your own values.

### Step 9 — Start the service

```sh
service awg start
```

The `awg0` interface should come up with the configured network settings.

### Step 10 — Verify the connection

```sh
awg show
```

Example output:

```
interface: awg0
  public key: <CUT>
  private key: (hidden)
  listening port: 64807
  jc: 46
  jmin: 140
  jmax: 781
  s1: 54
  s2: 62
  h1: 1785112540
  h2: 765208426
  h3: 1838104557
  h4: 1972791550

peer: <CUT>
  endpoint: <CUT_IP>:<CUT_PORT>
  allowed ips: 0.0.0.0/0
  latest handshake: 6 seconds ago
  transfer: 1.88 KiB received, 87.19 KiB sent
  persistent keepalive: every 1 minute
```

### Step 11 — Add the service to autostart

Install the **Shellcmd** package from the pfSense package manager:

**System → Package Manager → Available Packages** — search for `Shellcmd` and install it.

Then add the service via **Services → Shellcmd → Add**:

| Field | Value |
|-------|-------|
| Command | `service awg start` |
| Shellcmd Type | `earlyshellcmd` ⚠️ |

> **Why `earlyshellcmd`?** If you select a later start type, pfSense will not find the `awg0` interface during interface assignment on reboot and will stuck.
>
> This Shellcmd workaround is necessary because pfSense does not support autostart scripts via `/etc/rc.conf` the way standard FreeBSD does. The `rc.conf` on pfSense itself states: `# THIS FILE DOES NOTHING, DO NOT MAKE CONFIG CHANGES HERE`.

### Step 12 — Assign the interface in pfSense

Go to **Interfaces → Assignments** and assign `awg0`.

Configure the interface:
- **Name:** e.g. `AWG0`
- **IP type:** Static IPv4
- **IP address / mask:** your tunnel address and subnet (e.g. `10.0.14.88/24`)
- **Gateway:** e.g. `10.0.14.1` (will be created automatically)

You can verify the gateway was created under **System → Routing**.

To test connectivity, use **Diagnostics → Ping**.

---

## ✅ Setup Complete

The tunnel is now configured. Further steps — such as routing policies, firewall rules, and NAT — depend on your specific use case and are outside the scope of this document.

---

## Checksums

MD5 checksums:

| Version | `if_awg.ko` | `awg` | `amneziawg-go` |
|---------|-------------|-------|----------------|
| 2.7.2 | `7fa9df9d91c38b38672e02ceec94eb38` | `7a6b7ade2a6585eb61dce4d62246777b` | `feee8f9c81008c30f66f0e9e7315269c` |
| 2.7.0 | `0ce56e96dcc946be22ae702ad35c39d3` | `7a6b7ade2a6585eb61dce4d62246777b` | `de4611fa5c657201eab93f71396f1115` |
