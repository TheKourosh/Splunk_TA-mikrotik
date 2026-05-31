# Splunk MikroTik App

A Splunk app for parsing and extracting structured fields from MikroTik RouterOS syslog messages.

## What It Does

Parses MikroTik syslog events and extracts the following fields:

| Field | Description |
|---|---|
| `mt_device` | Hostname of the MikroTik router |
| `mt_topic` | MikroTik log topic (e.g. system, OVPN, L2TP) |
| `mt_message` | Raw log message body |
| `mt_user` | Username involved in the event |
| `mt_src_ip` / `src` | Source IP address |
| `mt_action` | Action type: `login` or `logout` |
| `mt_result` | Outcome: `success` or `failure` |
| `mt_method` | Connection method: `ovpn`, `l2tp`, `ssh`, `winbox`, etc. |

## Supported Event Types

- OVPN login success / failure / logout
- System login success / failure / logout (SSH, Winbox, API)
- L2TP/PPP authentication failure

## Installation

1. Copy the `default/` folder into your Splunk app directory:
```
   $SPLUNK_HOME/etc/apps/mikrotik-syslog/
```
2. Restart Splunk or reload the app
3. Assign sourcetype `mikrotik:syslog` to your MikroTik syslog inputs

## Known Issues

> ⚠️ **[#1 — Some hardcoded fields not searchable](../../issues/1)**
>
> Fields `mt_result` and `mt_action` are currently not returning results when searched directly (e.g. `mt_result=success`). This affects filtering and dashboards that rely on these fields. Investigation is ongoing. See the issue for details.

## CIM Compliance

The following CIM field aliases are included:

| CIM Field | Maps From |
|---|---|
| `src` | `mt_src_ip` |
| `user` | `mt_user` |
| `dest` | `mt_device` |
