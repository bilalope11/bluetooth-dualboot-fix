# Fix Bluetooth Re-Pairing on Windows + Fedora Dual Boot

Stop re-pairing your Bluetooth earbuds (or mouse, keyboard, headphones) every
time you switch between Windows and Linux on a dual-boot PC.

Tested with: **Windows 11 + Fedora**, **CMF Buds Pro 2**.

---

## The problem

### Simple explanation

When you pair Bluetooth earbuds with a computer, the two agree on a **secret
password** (a "link key"). Both sides save it, and they use it every time they
reconnect.

On a dual-boot PC, Windows and Linux use the **same Bluetooth chip**, so the
earbuds think they are talking to **one computer**. But each operating system
saves its **own** password.

1. You pair in Linux. The earbuds and Linux save password **A**.
2. You boot Windows and pair again. The earbuds **replace** password A with
   password **B**. Windows saves B.
3. You boot Linux again. Linux still has password A, the earbuds only know B.
   The connection fails, and you have to pair again.

Every re-pair breaks the other system. The fix is to make **both systems use
the same password**.

### Technical explanation

- Classic Bluetooth (BR/EDR) devices bond using Secure Simple Pairing. The
  result is a **128-bit link key** stored by both the host and the device.
- The device stores **one link key per host Bluetooth address (BD_ADDR)**.
  Both operating systems share the same adapter, so they present the
  **same BD_ADDR**.
- Re-pairing from one OS overwrites the key stored on the device. The other
  OS still holds the old key, so link authentication fails.
- Each OS keeps its keys in a different place:

| OS | Location of the link key |
| --- | --- |
| Windows | Registry: `HKLM\SYSTEM\CurrentControlSet\Services\BTHPORT\Parameters\Keys\<adapter>\<device>` (16-byte `REG_BINARY`) |
| Linux (BlueZ) | File: `/var/lib/bluetooth/<ADAPTER>/<DEVICE>/info`, section `[LinkKey]`, field `Key=` |

**Fix:** pair in both operating systems (Windows last), then copy the link key
from the Windows registry into the BlueZ `info` file. Both systems now hold
the key the device expects.

---

## Requirements

- A dual-boot PC with Windows and Fedora (other distros work with their own
  package manager).
- Windows **Fast Startup disabled**, otherwise the Windows partition is left
  in a hibernated state and Linux can't read it safely.
  - Run in an admin terminal on Windows: `powercfg /h off`
- Windows drive **not encrypted with BitLocker** (or unlocked first).
- `chntpw`, a tool that reads Windows registry files from Linux.

---

## Step-by-step fix

### 1. Pair in Linux first

Boot Fedora, pair the device, and note its address:

```bash
bluetoothctl devices
# Example: Device 11:22:33:44:55:66 My Earbuds
```

### 2. Pair in Windows last

Boot Windows and pair the device (remove it first if it's already listed).
Windows now holds the current key. **Do not pair in Linux again after this.**
Shut down fully (not restart).

### 3. Install chntpw in Fedora

```bash
sudo dnf install chntpw
```

### 4. Mount the Windows partition read-only

Find the Windows partition (NTFS, usually labeled "Windows"):

```bash
lsblk -f
```

Mount it read-only so nothing in Windows can be changed:

```bash
sudo mkdir -p /mnt/windows
sudo mount -o ro /dev/nvme0n1p3 /mnt/windows   # replace with your partition
```

### 5. Read the key from the Windows registry

```bash
cd /mnt/windows/Windows/System32/config
sudo chntpw -e SYSTEM
```

Inside chntpw:

```text
cd ControlSet001\Services\BTHPORT\Parameters\Keys
ls                      # shows your adapter address, e.g. aabbccddeeff
cd aabbccddeeff
ls                      # shows paired devices by address
hex 112233445566        # your device address, lowercase, no colons
```

Example output:

```text
:00000  1A 2B 3C 4D 5E 6F 70 81 92 A3 B4 C5 D6 E7 F8 09  .+<M^op.........
```

The 16 hex pairs in the middle are the key. Join them with no spaces:

```text
1A2B3C4D5E6F708192A3B4C5D6E7F809
```

Quit with `q`. If asked to write changes, answer **`n`**.

> Note: an offline registry has no `CurrentControlSet`. `ControlSet001` is
> usually the active one. If your key isn't there, check `ControlSet002`.

### 6. Put the key into Linux

Linux writes addresses in uppercase with colons.

```bash
sudo -i
cd /var/lib/bluetooth/AA:BB:CC:DD:EE:FF/11:22:33:44:55:66
cp info info.backup
nano info
```

Edit only the `Key=` line in the `[LinkKey]` section:

```ini
[LinkKey]
Key=1A2B3C4D5E6F708192A3B4C5D6E7F809
Type=4
PINLength=0
```

Save (Ctrl+O, Enter), exit (Ctrl+X), then leave root:

```bash
exit
```

### 7. Restart Bluetooth

```bash
sudo systemctl restart bluetooth
```

Turn on the device. It should now connect in **both** Fedora and Windows
without re-pairing.

---


## Troubleshooting

- **Mount fails with a hibernation or "unsafe state" error:** Fast Startup is
  still on. Disable it in Windows and do a full shutdown.
- **The device shows up as a folder (subkey) instead of a value in chntpw:**
  it uses **Bluetooth Low Energy** pairing. LE stores several values (`LTK`,
  `ERand`, `EDIV`, `IRK`) that map to `[LongTermKey]` and
  `[IdentityResolvingKey]` in the BlueZ file, with format conversions (for
  example, `ERand` must be byte-reversed and converted to decimal). This guide
  covers classic Bluetooth only.
- **True wireless earbuds with separate left/right addresses:** repeat the
  steps for each address.
- **It stopped working later:** you probably re-paired in one OS. Pair in
  Windows last and copy the key again.

To undo a mistake in the Linux file:

```bash
sudo cp /var/lib/bluetooth/<ADAPTER>/<DEVICE>/info.backup \
        /var/lib/bluetooth/<ADAPTER>/<DEVICE>/info
sudo systemctl restart bluetooth
```

---

## Automated alternative

[bt-dualboot](https://github.com/x2es/bt-dualboot) is a script that automates
steps 5 and 6. It also requires `chntpw` and a mounted Windows partition.
Read its README before using it.

---

## Safety notes

- The Windows partition is mounted **read-only** and chntpw changes are
  **never saved**, so Windows is not modified.
- Always back up the BlueZ `info` file before editing.
- The keys shown in this guide are examples. Don't publish your real link
  keys.
