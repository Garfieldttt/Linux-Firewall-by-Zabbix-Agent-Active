# Linux Firewall by Zabbix Agent Active

Monitors the Linux netfilter firewall through the Zabbix Agent in active mode, without granting the agent any extra privileges. No sudo, no file ACLs, no capabilities, no UserParameters and nothing to install on the monitored host.

Covers iptables and nftables. It reports whether the ruleset was loaded into the kernel, whether it survives the next reboot, whether netfilter is active, and how full the connection tracking table is.

---

## Requirements

- Zabbix Server 7.0 or higher
- Zabbix Agent 2 in active mode, with `ServerActive` and `Hostname` set. The `systemd.unit.info` keys do not exist in the C agent.
- systemd

---

## 1. Installation

1. Download `7.0/template_linux_firewall_by_zabbix_agent.yaml`
2. In Zabbix: **Data collection -> Templates -> Import**
3. Link **Linux Firewall by Zabbix Agent Active** to the host
4. Set `{$FW.SERVICE}` if the host does not use `netfilter-persistent`

Verify on the host:

```bash
zabbix_agent2 -c /etc/zabbix/zabbix_agent2.conf -t 'systemd.unit.info[netfilter-persistent.service,ActiveState]'
```

A healthy host answers `active`, and **Firewall service: state** shows `enabled (1)`.

---

## 2. Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$FW.SERVICE}` | `netfilter-persistent.service` | The systemd unit that loads the ruleset |
| `{$CONNTRACK.PUSED.WARN}` | `80` | Conntrack utilization (%) for the warning trigger |
| `{$CONNTRACK.PUSED.CRIT}` | `95` | Conntrack utilization (%) for the high trigger |
| `{$NF.REFCOUNT.MIN}` | `10` | Lowest nf_tables reference count that still counts as a loaded ruleset |
| `{$FW.TRIGGERS.ENABLED}` | `1` | Set to 0 to silence the availability triggers |

The firewall unit per distribution:

| Distribution | `{$FW.SERVICE}` |
|--------------|-------------------|
| Debian, Ubuntu with iptables-persistent | `netfilter-persistent.service` |
| Ubuntu with ufw | `ufw.service` |
| RHEL, Rocky, Alma, Fedora | `firewalld.service`, or `iptables.service` with `iptables-services` |
| openSUSE, SLES | `firewalld.service` |
| Arch | `nftables.service` or `iptables.service` |

---

## 3. Items

| Name | Description | Type | Key |
|------|-------------|------|-----|
| Firewall service: state | 1 if the firewall unit is active, else 0 | `Agent (active)` | `systemd.unit.info[{$FW.SERVICE},ActiveState]` |
| Firewall service: sub state | Sub state of the unit, `exited` for oneshot units, `running` for daemons | `Agent (active)` | `systemd.unit.info[{$FW.SERVICE},SubState]` |
| Firewall service: unit file state | `enabled` or `disabled` at boot | `Agent (active)` | `systemd.unit.info[{$FW.SERVICE},UnitFileState]` |
| Firewall service: last exit code | Exit code of the last rule load, 0 means success | `Agent (active)` | `systemd.unit.info[{$FW.SERVICE},ExecMainStatus,Service]` |
| Firewall rules: last load time | When the ruleset was last loaded | `Agent (active)` | `systemd.unit.info[{$FW.SERVICE},StateChangeTimestamp]` |
| Kernel modules: raw | Master item, history disabled | `Agent (active)` | `vfs.file.contents[/proc/modules]` |
| Netfilter modules loaded | 1 if `ip_tables` or `nf_tables` is loaded | `Dependent` | `nf.modules.loaded` |
| nf_tables reference count | Reference count of `nf_tables` | `Dependent` | `nf.tables.refcount` |
| Conntrack entries | Tracked connections | `Agent (active)` | `vfs.file.contents[/proc/sys/net/netfilter/nf_conntrack_count]` |
| Conntrack table size | Connection tracking table size | `Agent (active)` | `vfs.file.contents[/proc/sys/net/netfilter/nf_conntrack_max]` |
| Conntrack table utilization | Conntrack utilization in percent | `Calculated` | `nf.conntrack.pused` |
| IP forwarding | 1 if the host forwards packets | `Agent (active)` | `vfs.file.contents[/proc/sys/net/ipv4/ip_forward]` |

---

## 4. Triggers

| Trigger | Severity |
|---------|----------|
| Firewall service is not active | Average |
| Firewall service is not enabled at boot | Warning |
| Firewall rule load failed | High |
| Firewall rules were reloaded | Info |
| Netfilter modules are not loaded | Average |
| Ruleset appears to be flushed | Warning |
| Conntrack table utilization is high | Warning |
| Conntrack table is almost full | High |
| IP forwarding changed | Info |

---

## 5. Dashboard

**Firewall overview** with service state, sub state, netfilter module state and conntrack utilization, plus a graph **Conntrack table utilization**.

---

## 6. Notes

- The template does not read the ruleset itself. Rules, chains, policies and packet counters require CAP_NET_ADMIN. A manual `iptables -F` still leaves the unit in state `active`, only the `nf_tables` reference count reacts to it.
- That reference count is an approximation, not a rule count. Adjust `{$NF.REFCOUNT.MIN}` on hosts with very small rulesets.
- Without `nf_conntrack` loaded, both conntrack items turn unsupported.
- If netfilter is compiled into the kernel instead of built as a module, it does not appear in `/proc/modules` and **Netfilter modules loaded** reports 0. The stock kernels of Debian, Ubuntu and RHEL use modules.

---

## Tested with

- Zabbix 7.0 LTS with Zabbix Agent 2
- Debian 13 with iptables-nft and netfilter-persistent

Other distributions have not been tested.
