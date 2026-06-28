# Zabbix Template: Linux iptables Firewall

Monitors whether iptables rules are loaded on a Linux host by checking the saved rules file. Works with any iptables policy (INPUT DROP or ACCEPT) - triggers an alert when no rules file is found or the filter table is missing.

## Requirements

- Zabbix 7.0+
- Zabbix Agent (active mode) on the monitored host
- `iptables-persistent` or equivalent package that saves rules to a file

## How it works

The template reads the saved iptables rules file (`/etc/iptables/rules.v4` by default) using `vfs.file.contents`. It checks for the presence of `*filter` - which is present in every valid saved rules file regardless of chain policy. If the file is missing or contains no filter table, the item returns `0` and the trigger fires.

## Import

1. Download `zbx_export_templates.yaml`
2. In Zabbix: **Configuration → Templates → Import**
3. Select the file and click **Import**
4. Assign the template to your Linux hosts

## Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$RULES.V4_PATH}` | `/etc/iptables/rules.v4` | Path to the saved iptables rules file |
| `{$ON_OFF_IPTABLES_TRIGGER}` | `1` | Enable (`1`) or disable (`0`) the trigger |

## Items

| Name | Type | Key |
|------|------|-----|
| iptables status | Zabbix Agent (active) | `vfs.file.contents[{$RULES.V4_PATH}]` |

**Value mapping:** `1` = enabled, `0` = disabled

## Triggers

| Name | Severity | Condition |
|------|----------|-----------|
| iptables is disabled | Average | Status = 0 for 10s and trigger macro = 1 |

## Distribution-specific paths

The default path works on Debian and Ubuntu with `iptables-persistent`. For other distributions, override `{$RULES.V4_PATH}` on the host level:

| Distribution | Path |
|-------------|------|
| Debian / Ubuntu | `/etc/iptables/rules.v4` |
| RHEL / CentOS / Rocky / AlmaLinux | `/etc/sysconfig/iptables` |
| openSUSE | `/etc/sysconfig/iptables` |

To override per host: **Host → Macros → Add** `{$RULES.V4_PATH}` with the correct path.

## Notes

- Hosts using **nftables** natively (RHEL 8+, Debian with nftables active) are not supported by this template
- The trigger can be disabled per host by setting `{$ON_OFF_IPTABLES_TRIGGER}` to `0`
