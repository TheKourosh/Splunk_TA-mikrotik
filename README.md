# Splunk TA — MikroTik

A Splunk Technology Add-on (TA) for parsing and extracting structured fields from MikroTik RouterOS syslog messages.

---

## Table of Contents

- [What It Does](#what-it-does)
- [Extracted Fields](#extracted-fields)
- [Supported Event Types](#supported-event-types)
- [MikroTik Configuration](#mikrotik-configuration)
- [Splunk Installation](#splunk-installation)
- [Known Issues](#known-issues)

---

## What It Does

Parses MikroTik syslog events sent to a Splunk indexer and extracts structured fields for use in searches, dashboards, and alerts. CIM-compatible aliases are included.

---

## Extracted Fields

| Field | Description |
|---|---|
| `mt_device` | Hostname of the MikroTik router |
| `mt_topic` | MikroTik log topic (e.g. `system`, `OVPN`, `L2TP`) |
| `mt_message` | Raw log message body |
| `mt_user` | Username involved in the event |
| `mt_src_ip` / `src` | Source IP address |
| `mt_tunnel_ip` | Assigned tunnel IP (OVPN events) |
| `mt_action` | Action type: `login` or `logout` |
| `mt_result` | Outcome: `success` or `failure` |
| `mt_method` | Connection method: `ovpn`, `l2tp`, `ssh`, `winbox`, `api`, etc. |

### CIM Aliases

| CIM Field | Maps From |
|---|---|
| `src` | `mt_src_ip` |
| `user` | `mt_user` |
| `dest` | `mt_device` |

---

## Supported Event Types

| Topic | Event |
|---|---|
| OVPN | Login success, login failure, logout |
| System | Login success, login failure, logout (SSH, Winbox, API) |
| L2TP / PPP | Authentication failure |

---

## MikroTik Configuration

This section explains how to configure your MikroTik routers to send logs to Splunk correctly. There are two different configurations depending on router type — standard routers and CCR (Cloud Core Router) routers — because they support different syslog time formats.

---

### Non-CCR Routers

#### Step 1 — Create the Splunk logging action

```routeros
/system logging action
add name=splunk target=remote remote=YOUR_SPLUNK_IP remote-port=514 \
    src-address=0.0.0.0 remote-log-format=syslog remote-protocol=udp \
    syslog-time-format=iso8601 syslog-facility=daemon syslog-severity=auto \
    vrf=main
```

> ⚠️ Replace `YOUR_SPLUNK_IP` with your Splunk indexer or syslog receiver IP.  
> ⚠️ Replace `0.0.0.0` in `src-address` with the actual outgoing interface IP of the router.

Non-CCR routers use `syslog-time-format=iso8601` which produces a full timestamp including milliseconds and timezone offset. This is the preferred format for accurate time parsing in Splunk.

#### Step 2 — Configure logging rules

```routeros
/system logging

# Rules 0-3: catch-all severity levels (excluding topics with dedicated rules)
add action=splunk topics=info,!account,!ppp,!interface,!firewall,!ovpn,!bgp,!ospf,!radius,!system,!wireguard,!sstp,!l2tp,!disk,!manager,!pptp,!vrrp,!mpls,!pppoe prefix="[Info]"
add action=splunk topics=error,!account,!ppp,!interface,!firewall,!ovpn,!bgp,!ospf,!radius,!system,!wireguard,!sstp,!l2tp,!disk,!manager,!pptp,!vrrp,!mpls,!pppoe prefix="[Error]"
add action=splunk topics=warning,!account,!ppp,!interface,!firewall,!ovpn,!bgp,!ospf,!radius,!system,!wireguard,!sstp,!l2tp,!disk,!manager,!pptp,!vrrp,!mpls,!pppoe prefix="[Warning]"
add action=splunk topics=critical,!account,!ppp,!interface,!firewall,!ovpn,!bgp,!ospf,!radius,!system,!wireguard,!sstp,!l2tp,!disk,!manager,!pptp,!vrrp,!mpls,!pppoe prefix="[Critical]"

# Rule 4: account topic
add action=splunk topics=account,!web-proxy,!ppp,!ovpn,!radius,!wireguard,!sstp,!l2tp,!pptp,!pppoe,!system prefix="[Account]"

# Rules 5-23: dedicated topic rules
add action=splunk topics=ppp,!debug,!packet,!raw prefix="[PPP]"
add action=splunk topics=interface,!debug,!packet,!raw prefix="[Interface]"
add action=splunk topics=firewall,!debug,!packet,!raw prefix="[FireWall]"
add action=splunk topics=ovpn,!debug,!packet,!raw prefix="[OVPN]"
add action=splunk topics=bgp,!debug,!packet,!raw prefix="[BGP]"
add action=splunk topics=ospf,!debug,!packet,!raw prefix="[OSPF]"
add action=splunk topics=radius,!debug,!packet,!raw prefix="[Radius]"
add action=splunk topics=system,!debug,!packet,!raw prefix="[System]"
add action=splunk topics=wireguard,!debug,!packet,!raw prefix="[WireGuard]"
add action=splunk topics=sstp,!debug,!packet,!raw prefix="[SSTP]"
add action=splunk topics=l2tp,!debug,!packet,!raw prefix="[L2TP]"
add action=splunk topics=disk prefix="[Disk]"
add action=splunk topics=manager,!debug,!packet,!raw prefix="[Manager]"
add action=splunk topics=pptp,!debug,!packet,!raw prefix="[PPTP]"
add action=splunk topics=vrrp,!debug,!packet,!raw prefix="[VRRP]"
add action=splunk topics=mpls,!debug,!packet,!raw prefix="[MPLS]"
add action=splunk topics=pppoe,!debug,!packet,!raw prefix="[PPPoE]"
add action=splunk topics=web-proxy,!debug,!packet,!raw,!account prefix="[Web-Proxy]"
```

---

### CCR Routers (Cloud Core Router)

CCR routers do not support `iso8601` time format in syslog. Use `bsd-syslog` instead.

#### Step 1 — Create the Splunk logging action

```routeros
/system logging action
add name=splunk target=remote remote=YOUR_SPLUNK_IP remote-port=514 \
    src-address=0.0.0.0 remote-log-format=syslog remote-protocol=udp \
    syslog-time-format=bsd-syslog syslog-facility=syslog syslog-severity=auto \
    vrf=main
```

> ⚠️ Replace `YOUR_SPLUNK_IP` with your Splunk indexer or syslog receiver IP.  
> ⚠️ Replace `0.0.0.0` in `src-address` with the actual outgoing interface IP of the router.

#### Step 2 — Configure logging rules

Same rules as non-CCR above — copy the exact same `/system logging` block.

---

### Key Differences Between Router Types

| Setting | Non-CCR | CCR |
|---|---|---|
| `syslog-time-format` | `iso8601` | `bsd-syslog` |
| `syslog-facility` | `daemon` | `syslog` |
| Timestamp precision | Milliseconds + timezone offset | Second-level only |

---

### Optional — Mirror Rules to Memory

It is recommended to also add the same logging rules for the `memory` action so logs remain visible locally on the router via Winbox or terminal:

```routeros
/system logging
add action=memory topics=ovpn,!debug,!packet,!raw prefix="[OVPN]"
add action=memory topics=system,!debug,!packet,!raw prefix="[System]"
add action=memory topics=l2tp,!debug,!packet,!raw prefix="[L2TP]"
# repeat for any other topics you want visible locally
```

---

### Best Practices

- **Always set `src-address`** to the router's actual outgoing interface IP. Leaving it as `0.0.0.0` may cause syslog packets to arrive from an unexpected source IP, making device identification harder in Splunk.
- **Exclude `debug`, `packet`, and `raw` sub-topics** from all rules. These generate extremely high log volumes with low security value and will strain both the router memory and your Splunk indexer.
- **Use dedicated rules per topic** (as configured above) rather than a single catch-all rule. This ensures the `mt_topic` field is populated correctly and prefix-based filtering works as expected in Splunk.
- **UDP syslog on port 514** is used for performance. If log delivery reliability is critical in your environment, consider placing a syslog relay (e.g. rsyslog or syslog-ng) between the router and Splunk with TCP transport.
- **CCR vs non-CCR matters.** Using `iso8601` on a CCR or `bsd-syslog` on a non-CCR will cause timestamp parsing failures in Splunk. Verify your router model before applying the config.

---

## Splunk Installation

1. Copy the app folder into your Splunk apps directory:
   ```
   $SPLUNK_HOME/etc/apps/TA-mikrotik/
   ```
2. Restart Splunk or reload the app via **Settings → Server Controls → Restart Splunk**
3. Configure your syslog UDP input on port 514 and set the sourcetype to `mikrotik:syslog`

### App Structure

```
TA-mikrotik/
└── default/
    ├── props.conf
    └── transforms.conf
```

---

## Known Issues

> ⚠️ **[#1 — Fields `mt_result` and `mt_action` not directly searchable](../../issues/1)**
>
> Fields set as hardcoded literal values in `transforms.conf` FORMAT directives (`mt_result`, `mt_action`) are currently not returning results when used in direct equality searches (e.g. `mt_result=success`). The field appears correctly when using `mt_result=*` but value-based matching fails. Root cause is under investigation.
