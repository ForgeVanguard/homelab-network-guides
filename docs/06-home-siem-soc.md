# 06 - Home SIEM / SOC / network monitoring

An open-source, low-power stack that gives a home lab what SolarWinds plus a SIEM give a small
business: endpoint security telemetry, network/device health, searchable logs, dashboards and
reliable alerting.

## Stack

| Layer | Primary | Why | Alternatives |
|---|---|---|---|
| SIEM / XDR | **Wazuh** | Agents, log analysis, FIM, vulnerability detection, CIS assessment, MITRE mapping. 1-25 agents: ~4 vCPU / 8 GB / 50 GB for 90 days of alerts | Graylog, Security Onion (heavier, network-focused) |
| Network / infra monitoring | **Zabbix** | SNMP for firewall, switches and APs; agents; discovery; templates; maps | Prometheus + exporters, LibreNMS, Checkmk |
| Dashboards | **Grafana** | One pane over Wazuh/OpenSearch, Zabbix, Prometheus, Loki, Home Assistant | Native Zabbix/Wazuh dashboards |
| Inventory / config | Wazuh + Zabbix discovery, Ansible later | Security and operational inventory | Oxidized (config backups), NetBox (IPAM) |
| Logs (extra) | Optional Loki or Graylog | Start with Wazuh only | ELK |
| Alerting | **ntfy** (self-hosted) + a dedicated phone | See below | Gotify, Pushover, email |

```text
endpoints / network gear ──agents, SNMP, syslog──▶ Wazuh + Zabbix (low-power host, MGMT/TRUSTED)
                                                          │
                                                       Grafana
                                                          │
                                         ntfy ──▶ daily phones (high + normal topics)
                                              └─▶ dedicated always-plugged-in phone (critical)
                                         optional SMS gateway for critical-only
```

## Hardware

- Intel N150-class mini PC, 16 GB RAM, 512 GB SSD + a second NVMe for indices/history.
- Idle ~6-10 W; roughly $15-30/year at typical US rates. Quiet at idle.
- 16 GB is the ceiling on this class; it is enough for a lean all-in-one stack with a handful of agents.
- Keep it a dedicated box so monitoring survives a hypervisor outage.

## Alerting design

**Primary:** ntfy topics with auth. Zabbix (webhook media type), Grafana (webhook contact point)
and Wazuh (custom integration script) all post to it. Route critical/high to a high-priority
topic, informational to a low one.

**Secondary:** an old Android phone kept on USB power in the rack area, subscribed to the
critical topic with maximum priority and a distinct sound. It survives Do Not Disturb and battery
optimization on personal phones. Optional: automation app to wake the screen or drive a speaker.

**Out-of-band (optional):** an SMS API for only the highest-severity alerts, so you still hear
about it if the ntfy host or the internet link is down. Email as an audit trail.

## Coverage

| Source | Method |
|---|---|
| OPNsense | Syslog to Wazuh (and Zabbix); SNMP |
| Core/access switches, AP | SNMP templates in Zabbix |
| Servers, PCs, VMs | Wazuh agent + Zabbix agent |
| IoT | Home Assistant for presence/health; Wazuh where an agent fits |
| Isolated lab | Agents or one-way log shipping only. Never bridge it to the home LAN |

## Phased rollout

1. Hardware, Ubuntu LTS or Debian.
2. **Zabbix** first: availability and performance for network gear and hosts (immediate value).
3. **Wazuh** all-in-one, agents on key endpoints, OPNsense syslog.
4. **Grafana** with community dashboards for Zabbix and Wazuh.
5. **ntfy**, phone apps, test the dedicated phone.
6. Tune rules and triggers; fresh installs are noisy.
7. Optional: Loki, Oxidized, Ansible, longer retention.

## Hardening the monitoring host

- Grafana (3000), Wazuh dashboard (443) and Zabbix UI only reachable from the management subnet.
  On a Windows host, `Invoke-WindowsHardening.ps1 -ManagementPort 3000 -ManagementSubnet <MGMT_SUBNET>`
  from the windows-hardening-kit repo does this.
- Change every default credential, put the UIs behind TLS, back up configs and the Wazuh/Zabbix databases.

## Cost

Mini PC $180-250, extra SSD $40-80, spare phone $0-50, software $0. Under ~$350 total.
