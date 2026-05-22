
# HA Tool – High Availability Config Switcher

**Version 1.5.2**

`ha-tool` is a lightweight Bash script that automatically selects and activates the **lowest‑numbered available** configuration from a set of numbered config files. It periodically checks the availability of each peer (by ping) and switches to the best one without manual intervention – ideal for multi‑path or failover setups.

---

## ✨ How It Works (Algorithm)

1. **Discover all config files** – The script scans `CONF_DIR` for files matching `BASE_NAME_FORMAT*FILE_EXTENTION` (e.g., `wg*.conf`).  
2. **Test availability** – For each config, it extracts a target IP (using a user‑defined pattern) and pings it (`-c1 -W2`). Reachable configs are marked as *available*.  
3. **Select the preferred config** – From the available configs, it looks for a numeric suffix between the base name and the extension (e.g., `pth23.conf` → number `23`). The config with the **lowest** number wins. Configs without a number are ignored.  
4. **Persist the current state** – The name of the currently active config is stored in `UP_CONF_FILE` inside `CONF_DIR`.  
5. **Switch if needed** – Every 6 seconds (1s + 5s sleep), the script compares the preferred config against the current one. If they differ, it brings down the old config and brings up the new one.  

> The script runs in an infinite loop – perfect for a background service or a systemd unit.

---

## 📦 Installation

1. Download `ha-tool` and make it executable:
   ```bash
   chmod +x ha-tool
   ```
2. Edit the configuration section inside the script (see below).
3. Run manually or use ready systemd service.

---

## ⚙️ Configuration Parameters (edit inside the script)

| Variable           | Description                                                                 | Example value                          |
|--------------------|-----------------------------------------------------------------------------|----------------------------------------|
| `CONF_DIR`         | **Absolute path** to the directory containing your config files             | `/etc/wireguard`                       |
| `FILE_EXTENTION`   | File extension of your configs (including the dot)                          | `.conf`                                |
| `BASE_NAME_FORMAT` | Common prefix of all config files (without number and extension)            | `pth` (matches `pth1.conf`, `pth23.conf`) |
| `DOWN_COMMAND`     | Command to bring a config **down** (will receive the file name as argument) | `sudo wg-quick down`                   |
| `UP_COMMAND`       | Command to bring a config **up**   (will receive the file name as argument) | `sudo wg-quick up`                     |

### 🔍 Customising the IP Detection

By default, the script expects each config file to contain a line like:
```
remote.ip = 10.0.0.1
```
It extracts the IP using:
```bash
grep '^remote\.ip' "$full_path" | cut -d= -f2 | tr -d '[:space:]'
```
**You must change this line** inside the `find_available_conf()` function to match the format used in your configs (e.g., `Endpoint = 203.0.113.5:51820` → extract only the IP part).

---

## 📁 File Naming Convention

Each config file **must** follow this pattern:

```
<BASE_NAME_FORMAT><NUMBER><FILE_EXTENTION>
```

**Examples:**
- `pth1.conf`
- `pth2.conf`
- `pth99.conf`

The script extracts the `<NUMBER>` part to determine priority – lower numbers are preferred. Files that do not contain a numeric suffix are simply ignored.

---

## 🚀 Usage

```bash
# Run in foreground (manual testing)
./ha-tool

# Show version
./ha-tool --version
```

To run as a background daemon, use a systemd service or `nohup`.

---

## ⚠️ Important Notes

- The script **does not** validate that `DOWN_COMMAND` or `UP_COMMAND` are correctly installed or that the config names work with those commands (e.g., `wg-quick up pth1.conf` expects the config file name, not just the interface name – adjust accordingly).
- All commands are executed with `sudo` – make sure the user running the script has passwordless sudo for `wg-quick` and `ping`.
- The `CONF_DIR` must be an **absolute path** starting from root (e.g., `/etc/wireguard`).
- The ping test uses **one ICMP packet** with a 2‑second timeout. A reachable peer is considered “available”.
- The script writes a state file `UP_CONF_FILE` inside `CONF_DIR`. Ensure the directory is writable.

---

## 📄 License

This tool is provided “as is” – feel free to modify and distribute.

---

