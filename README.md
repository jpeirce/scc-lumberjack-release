# 🪓 scc-lumberjack (User Guide)

> One-shot hardware + log snapshot for support and diagnostics.

`scc-lumberjack` is a single binary that gathers **hardware info**, **system logs**, **BMC/IPMI state**, **Broadcom/LSI RAID data**, and **NVIDIA GPU diagnostics** into a single archive you can hand to support.

It is designed to be:

- ✅ **Safe** – read-only, no configuration changes  
- ✅ **Deterministic** – same inputs, same archive layout  
- ✅ **Support-friendly** – all artifacts indexed in `index.json`  
- ✅ **Privacy-aware** – optional IP/MAC redaction

---

## 🔐 Privileges

This tool is designed to run as **root/Administrator**.

- **Linux**: run with `sudo`
- **Windows**: run from an elevated PowerShell or Command Prompt

If you are not root/Admin, `scc-lumberjack` will log an error and exit.

---

## 🚀 Usage

Basic pattern:

```bash
# Linux
sudo ./scc-lumberjack

# Windows (PowerShell, elevated)
.\scc-lumberjack.exe
```

### Common flags

```text
  -c, --config PATH
        Path to the configuration file (default: lumberjack.json).

  -r, --with-lsiget
        Enable LSIget collection for Broadcom/LSI support bundles.

  -l, --with-syslogs
        Enable system log collection (dmesg, syslog, Windows Event Logs).

  -g, --with-nvidia-bug-report
        On Linux, run nvidia-bug-report.sh and collect its output
        (in addition to nvidia-smi and dmesg Xid/NVRM).
        (No-op on Windows as the tool is not available).

  -u, --lsiget-url URL
        Specify a custom URL to fetch LSIget/BCMget from.

  -H, --lsiget-sha256 CHECKSUM
        Specify the expected SHA256 checksum for the LSIget download.

  -s, --sanitize-logs
        Enable redaction of IPv4, IPv6, and MAC addresses in collected text.

  -a, --all
        Enable all collection (gpus, logs, raid).

  -O, --offline
        Run in offline mode. Disables network downloads (e.g., fetching LSIget).

  --log-level LEVEL
        Set the logging level (debug, info, warn, error). Default is info.

  -d, --dry-run
        Print the commands that would be run without executing them.

  -F, --FORCE
        Run without privilege checks (Best Effort). Many collectors will fail.

  -G, --generate-config
        Generate a config file (lumberjack.json) from the provided positional arguments.

  -R, --run
        Run collection immediately after generating config (used with -G).

  -f, --force
        Overwrite existing config file during generation (used with -G).

  -t, --timeout DURATION
        Set an overall timeout for the collection run (e.g. "15m", "30m").

  -T, --temp-dir PATH
        Specify a custom directory for temporary files (default: system temp).

  -o, --output-dir PATH
        Directory to save the archive (default: current directory).

  -z, --zip
        Use .zip for output archive instead of .tar.xz.
```

---

## 🛠 Custom Commands

You can define custom commands to be run by `scc-lumberjack` in your `config.json` file. This is useful for collecting information that is not covered by the default collectors.

The `customCommands` section in `config.json` is an array of objects:

```json
{
  "customCommands": [
    {
      "command": "lspci",
      "args": ["-vvv"],
      "label": "lspci_verbose"
    }
  ]
}
```

The output will be saved to the `custom/` directory (e.g. `custom/lspci_verbose.log`).

> **⚠️ Security Warning:** Commands defined in `config.json` are executed as-is with root/Administrator privileges. Ensure you only configure trusted commands.

> **⚠️ Configuration Behavior:** If you provide a `config.json` (via `-c`), it is **merged** into the internal default configuration.
>
> This means:
> *   You can override specific tools (e.g., `journalctl`) without affecting others (like `lsiget`).
> *   New tools are added.
> *   Existing tools with the same name and OS are updated with your changes.
> *   Top-level settings (e.g., `maxLogSizeMB`) are overwritten if you specify them.

You can generate a config snippet using:
```bash
./scc-lumberjack -G "lspci -vvv"
# Creates lumberjack.json
```

To generate and run immediately:
```bash
./scc-lumberjack -G "lspci -vvv" -R
```

**Advanced:** Generate to a specific file and run immediately:
```bash
# Separate flags
./scc-lumberjack -G "lspci -vvv" "df -h" -R -c my_config.json

# Combined flags
./scc-lumberjack -GRc my_config.json "lspci -vvv" "df -h"
```

---

## ⚙️ Configuring System Tools (Advanced)

You can override the default arguments for system tools like `journalctl` and `dmesg` by adding a `tools` section to your `config.json`. This allows you to filter logs or change output formats without recompiling.

**Example `config.json`:**

```json
{
  "tools": [
    {
      "name": "journalctl",
      "commands": [
        ["-u", "ssh", "--no-pager", "-n", "5000"],
        ["-u", "docker", "--no-pager", "-n", "1000"]
      ],
      "overrideDefaults": true
    },
    {
      "name": "dmesg",
      "commands": [
        ["-T", "-k"]
      ],
      "overrideDefaults": true
    }
  ]
}
```

**Defaults:**
- **`journalctl`**: `["--no-pager", "-n", "20000"]` (Captures last 20k lines of all logs)
- **`dmesg`**: `["-T"]` (Captures kernel ring buffer with human-readable timestamps)

If `overrideDefaults` is `false` (default), your custom commands run *in addition* to the defaults. If `true`, they replace the defaults.

---

## 🎯 Example commands

### 1. Typical Linux run (everything auto-detected + logs)

```bash
sudo ./scc-lumberjack --with-syslogs --sanitize-logs --timeout 20m
```

### 2. Save to specific directory

```bash
sudo ./scc-lumberjack --output-dir /tmp/logs
```

### 3. Broadcom support focus (StorCLI + LSIget)

```bash
sudo ./scc-lumberjack \
  --with-lsiget \
  --sanitize-logs
```

---

## 📦 What the output looks like

A successful run creates a tarball in your **current working directory** (or specified via `--output-dir`), named like:

```text
scc-logs-<hostname>-<UTC-timestamp>.tar.xz
```

Inside:

```text
scc-logs-<hostname>-2025-11-20T19-42-00Z/
├─ index.json                # Run metadata + hardware info + error list
├─ lumberjack.log            # Log of the collection process itself
├─ hw/
│  ├─ cpu.txt
│  ├─ memory.txt
│  ├─ storage.txt
│  ├─ nvme.txt
│  ├─ bios.txt
│  ├─ system.txt
│  ├─ chassis.txt
│  └─ network.txt
├─ logs/
│  ├─ dmesg.log
│  ├─ journalctl.log        # Systemd journal
│  ├─ syslog.log            # or messages.log (Linux)
│  ├─ windows_system_events.txt   # etc., on Windows (will be redacted if --sanitize-logs is enabled)
│  └─ ...
├─ bmc/
│  ├─ ipmicfg_summary.log
│  ├─ ipmicfg_pminfo.log
│  ├─ saa_logs.txt          # if SAA was used
│  └─ ...
├─ storage/
│  ├─ storcli.log
│  └─ lsiget/
│      ├─ lsiget_command.log
│      └─ <vendor-generated-files>...
├─ gpu/
│  ├─ nvidia-smi.log
│  ├─ nvidia-dmesg.log
│  └─ nvidia-bug-report.log # when enabled and available
└─ ...
```

`index.json` includes:

- Hostname (optionally scrubbed)
- Hardware inventory (`HardwareInfo`)
- Collection timestamp
- A flat list of any **collection errors** (e.g. “StorCLI not found”, “NVIDIA bug-report unsupported on Windows”)

---

## 🛡️ Redaction & Privacy

When `--sanitize-logs` is enabled:

- **Redacts**
  - IPv4 addresses
  - IPv6 addresses
  - MAC addresses
  - Email addresses
  - Common secrets (passwords, API keys) in key=value format

- **Applies to**
  - BMC/IPMI logs
  - System logs (Linux + Windows, including Windows Event Log text exports)
  - `nvidia-smi` and dmesg snippets
  - Hardware network dumps (e.g., `ethtool`)
  - Hostname in `index.json`

- **Intentionally *not* redacted**
  - Vendor-generated **LSIget** log files under `storage/lsiget/`
    - Rationale: support often needs serials, drive IDs, and controller identifiers intact.

---

## 🔍 Troubleshooting

**`scc-lumberjack` fails with a privilege error**

- Ensure:
  - Linux: you used `sudo` and the account is allowed to use it.
  - Windows: PowerShell/Command Prompt is “Run as Administrator”.

---

**Broadcom/LSI data is missing**

- Check the archive’s `index.json` → `CollectionErrors` for messages about `storcli` or `lsiget`.
- If StorCLI is missing:
  - Consider re-running with `--with-lsiget` (which attempts to fetch LSIget/BCMget).
- If LSIget download fails:
  - Verify outbound network access and/or use `--lsiget-url`/`--lsiget-sha256` overrides.
  - Use `--offline` to skip fetch attempts if internet access is restricted.

**General Tip:** Use `-a` or `--all` to enable all collectors (Syslogs, LSIget, NVIDIA bug report) in one go.

---

**NVIDIA bug report is not collected**

- On Windows, `nvidia-bug-report.sh` is not supported; you’ll still get `nvidia-smi`.
- On Linux, check:
  - `nvidia-bug-report.sh` is installed and in `PATH`.
  - `index.json` for related error messages.
