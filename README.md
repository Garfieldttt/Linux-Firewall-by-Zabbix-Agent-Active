# Linux Firewall by Zabbix Agent Active

This Zabbix template monitors the Linux netfilter firewall through the Zabbix Agent in active mode, and it does so without granting the agent a single extra privilege. No sudo, no file ACLs, no capabilities, no UserParameters and no scripts on the monitored host.

It covers iptables and nftables alike. On current Debian and Ubuntu systems `iptables` is `iptables-nft`, so iptables rules end up in the `nf_tables` module either way. Hosts that load their ruleset through a different unit only override one macro.

The common approach of reading `/etc/iptables/rules.v4` does not work: that file is `0640 root:root` while the agent runs as user `zabbix`, so the item turns unsupported with permission denied. It also only proves that a rules file exists on disk, never that the rules were loaded into the kernel. This template reads the live state instead.

---

## Requirements

- Zabbix Server 7.0 or higher
- Zabbix Agent 2 in active mode. The `systemd.unit.info` keys come from the systemd plugin of Agent 2 and do not exist in the C agent. `ServerActive` and `Hostname` must be set in `zabbix_agent2.conf` and match the host name in Zabbix.
- systemd, and a unit that loads the firewall ruleset

No configuration on the monitored host, no package to install.

---

## 1. Installation

1. Download `7.0/template_linux_firewall_by_zabbix_agent.yaml`
2. In Zabbix: **Data collection -> Templates -> Import**
3. Link **Linux Firewall by Zabbix Agent Active** to the host
4. On hosts that do not use `netfilter-persistent`, set `{$FW.SERVICE}` (see below)

Interfaces stay as they are, the template uses active agent checks only.

### Verify

On the monitored host:

```bash
zabbix_agent2 -c /etc/zabbix/zabbix_agent2.conf -t 'systemd.unit.info[netfilter-persistent.service,ActiveState]'
```

A healthy host answers `active`. In Zabbix, **Firewall service: state** then shows `enabled (1)` and **Firewall service: last exit code** shows `0`.

---

## 2. Macros

### Firewall unit

| `{$FW.SERVICE}` | `netfilter-persistent.service` | systemd unit that loads the firewall ruleset. Override per host, for example nftables.service, ufw.service or firewalld.service. |

The default is the unit that `iptables-persistent` ships on Debian and Ubuntu. It runs `iptables-restore < /etc/iptables/rules.v4`, which is why its exit code is the result of loading your ruleset.

| Setup | Value |
|-------|-------|
| Debian, Ubuntu with iptables-persistent | `netfilter-persistent.service` |
| nftables | `nftables.service` |
| ufw | `ufw.service` |
| firewalld | `firewalld.service` |

### Thresholds

| Macro | Default | Description |
|-------|---------|-------------|
| `{$CONNTRACK.PUSED.WARN}` | `80` | Conntrack table utilization in percent for the warning trigger. |
| `{$CONNTRACK.PUSED.CRIT}` | `95` | Conntrack table utilization in percent for the high severity trigger. |
| `{$NF.REFCOUNT.MIN}` | `10` | Lowest nf_tables reference count that still counts as a loaded ruleset. |

### Alert enable

| Macro | Default | Description |
|-------|---------|-------------|
| `{$FW.TRIGGERS.ENABLED}` | `1` | Set to 0 to silence the availability triggers of this template. 1 keeps them active. |

---

## 3. Items

| Name | Description | Type | Key |
|------|-------------|------|-----|
| Firewall service: state | ActiveState of {$FW.SERVICE}, mapped to 1 (active) or 0 (anything else, including a missing unit). | `Zabbix agent (active)` | `systemd.unit.info[{$FW.SERVICE},ActiveState]` |
| Firewall service: sub state | SubState of {$FW.SERVICE}. For a oneshot unit like netfilter-persistent, "exited" is the normal state after a successful rule load. | `Zabbix agent (active)` | `systemd.unit.info[{$FW.SERVICE},SubState]` |
| Firewall service: unit file state | Tells whether the firewall unit is enabled at boot. A disabled or masked unit means the rules will not come back after the next reboot. | `Zabbix agent (active)` | `systemd.unit.info[{$FW.SERVICE},UnitFileState]` |
| Firewall service: last exit code | Exit code of the last rule load. 0 means the ruleset was applied without error. The third key parameter Service is mandatory for this property. | `Zabbix agent (active)` | `systemd.unit.info[{$FW.SERVICE},ExecMainStatus,Service]` |
| Firewall rules: last load time | Timestamp of the last state change of the firewall unit. The property is delivered in microseconds and converted to unixtime here. | `Zabbix agent (active)` | `systemd.unit.info[{$FW.SERVICE},StateChangeTimestamp]` |
| Kernel modules: raw | Master item. /proc/modules is world readable (0444), so no extra privileges are needed. | `Zabbix agent (active)` | `vfs.file.contents[/proc/modules]` |
| Netfilter modules loaded | Whether ip_tables or nf_tables is present in the kernel. | `Dependent item` | `nf.modules.loaded` |
| nf_tables reference count | Third column of the nf_tables line in /proc/modules. It counts the objects referencing the module and drops sharply when the ruleset is flushed. | `Dependent item` | `nf.tables.refcount` |
| Conntrack entries | Current number of tracked connections. The file is world readable (0444). | `Zabbix agent (active)` | `vfs.file.contents[/proc/sys/net/netfilter/nf_conntrack_count]` |
| Conntrack table size | Maximum number of tracked connections. | `Zabbix agent (active)` | `vfs.file.contents[/proc/sys/net/netfilter/nf_conntrack_max]` |
| Conntrack table utilization | A full conntrack table silently drops new connections, so this is the metric that usually explains "the firewall is dropping traffic". | `Calculated` | `nf.conntrack.pused` |
| IP forwarding | Whether the host routes packets between interfaces. Relevant on gateways, and an unexpected change is worth a look. | `Zabbix agent (active)` | `vfs.file.contents[/proc/sys/net/ipv4/ip_forward]` |

`Kernel modules: raw` is the master item behind the two netfilter items. Its history is disabled, it only carries the data through.

---

## 4. Triggers

| Trigger | Severity | Description |
|---------|----------|-------------|
| Firewall service is not active | Average | The unit that loads the firewall rules is not active. Either it failed, it was stopped, or the unit does not exist on this host. |
| Firewall service is not enabled at boot | Warning | The firewall rules are loaded right now, but the unit will not start on the next boot. |
| Firewall rule load failed | High | The unit ran but exited with a non zero code, so the ruleset was not applied completely. |
| Firewall rules were reloaded | Info | The firewall unit changed state, so the ruleset was reloaded. Informational, close manually. |
| Netfilter modules are not loaded | Average | Neither ip_tables nor nf_tables is loaded, so no packet filtering is happening in the kernel. |
| Ruleset appears to be flushed | Warning | The nf_tables reference count fell below {$NF.REFCOUNT.MIN}, which usually means the ruleset was flushed or never loaded. |
| Conntrack table utilization is high | Warning | The connection tracking table is filling up. Once it is full, new connections are dropped without any log entry. |
| Conntrack table is almost full | High | Raise net.netfilter.nf_conntrack_max or shorten the conntrack timeouts. |
| IP forwarding changed | Info | net.ipv4.ip_forward changed. Informational, close manually. |

Three dependencies keep the alert noise down: the rule load failure depends on the service being active, the flushed ruleset depends on the modules being loaded, and the full conntrack table depends on the high utilization trigger, so only one of the two conntrack problems is ever open.

---

## 5. Dashboard

The template ships a dashboard **Firewall overview** with the service state, the sub state, the netfilter module state and the conntrack utilization, plus a graph **Conntrack table utilization**.

---

## 6. Notes

- **Where the data comes from:** systemd unit properties are read over the system D-Bus. Reading properties is unprivileged, only starting, stopping or enabling a unit goes through polkit. `/proc/modules` is mode 0444, `/proc/sys/net/netfilter/nf_conntrack_count` is 0444, `nf_conntrack_max` and `ip_forward` are 0644. Everything the template reads is readable by any user on the system.
- **What the exit code means:** `netfilter-persistent.service` is a oneshot unit. After a successful rule load it sits in state `active` with sub state `exited` and `ExecMainStatus` 0. A non zero exit code means `iptables-restore` refused part of your ruleset.
- **`ExecMainStatus` needs three key parameters.** The property lives on the Service interface, so the key must read `systemd.unit.info[<unit>,ExecMainStatus,Service]`. Without the third parameter the item turns unsupported.
- **Boot safety:** `UnitFileState` catches the case where the rules are loaded right now but the unit is disabled, so the firewall would not come back after the next reboot. That case is invisible to anything that only looks at the running state.
- **Conntrack:** A full connection tracking table drops new connections without writing a single log line, which is the usual explanation behind "the firewall is dropping traffic". Utilization is a calculated item from count and max.
- **The reference count is an approximation.** The third column of the `nf_tables` line in `/proc/modules` counts the objects referencing the module. It drops sharply when the ruleset is flushed, which makes it a usable signal, but it is not a rule count and no threshold on it is exact. `{$NF.REFCOUNT.MIN}` may need adjusting on hosts with very small rulesets.
- **What this template cannot do:** none of these sources exposes the ruleset itself. Rules, chains, policies and per rule packet counters require CAP_NET_ADMIN, which is exactly what this template avoids. A manual `iptables -F` after a successful boot still leaves the unit in state `active`, only the reference count reacts. If you need the ruleset itself, let a root owned systemd timer write `iptables-save` into a world readable file and read that file with the agent. The agent still needs no privileges then, at the price of one unit and one timer per host.
- **Missing unit:** if the unit named in `{$FW.SERVICE}` does not exist, the key turns unsupported, preprocessing maps that to 0 and the service trigger fires. A wrong macro value produces an alert rather than silence.
- **Dependent items:** the two netfilter items map a missing match to 0 through a custom value error handler, so a host without netfilter modules reports 0 instead of turning the item unsupported.

---

## Tested with

- Zabbix 7.0 LTS with Zabbix Agent 2
- Debian 13 with iptables-nft and netfilter-persistent

Other distributions have not been tested.
