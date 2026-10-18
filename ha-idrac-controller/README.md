# Home Assistant iDRAC Controller Add-on

Control one or more Dell PowerEdge servers from Home Assistant over IPMI. The
add-on reads temperatures, fan speeds, power consumption and PSU status,
publishes them through MQTT, and manages fan speed per server.

This is the main add-on in the [HA-iDRAC fork](https://github.com/sam2kb/HA-iDRAC).
The original single-server build is still published as
[HA iDRAC Controller (Legacy)](../ha-idrac-controller-legacy/README.md); it is
not needed for a T430.

## Features

- Multi-server support: monitor and control several PowerEdge servers from one
  add-on instance.
- Per-server fan control with three modes:
    - Simple thresholds: a three-tier system (base, high, critical).
    - Multi-point curve: a custom temperature-to-fan-speed curve.
    - Target temperature: a PID loop that holds a chosen CPU temperature.
- Server monitoring: each server becomes an MQTT device with sensors for
  individual CPU temperatures, hottest CPU, inlet and exhaust temperatures, fan
  RPM, power consumption, target fan speed, PSU status and connectivity.
- Remote action: a shutdown button for each server.
- Web UI through ingress: a live dashboard and a Manage Servers page for
  adding, editing and deleting servers.
- MQTT discovery: entities are created automatically.

## Changes in this fork (0.3.0)

This release promotes the multi-server build to the main add-on. It now uses
the `ha_idrac_controller` slug and the plain "HA iDRAC Controller" name. The
older single-server build moved to `ha_idrac_controller_legacy` and is called
"HA iDRAC Controller (Legacy)". If you installed the 0.2.0 BETA build, remove
it and install this one; its `/data` volume is tied to the old slug, so the
server list does not carry over.

The code is the same as 0.2.0. CPU temperatures come from the IPMI processor
entity (`3.x`), so `CPU1 Temp` on 13G/iDRAC 8 and `Temp` on 12G/iDRAC 7 both
work. Inlet and exhaust names are anchored, so `PSU1 Inlet Temp` cannot replace
the chassis inlet. Fan control has a 2 °C hysteresis band, a 5% floor, and a
hand-back to Dell auto when a speed write fails. A server that refuses fan
control with 0xc1/0xd4/0xd5 is detected once, then monitoring continues.

MQTT outages no longer stop a worker, passwords go through `ipmitool -E`, and
the Supervisor watchdog probes `/health`. The MQTT device receives
`sw_version` and `serial_number` from the server. Contradictory thresholds are
corrected with a warning, and temperatures outside -20 to 125 °C are ignored.

## Prerequisites

1.  Dell PowerEdge server with iDRAC 7 or 8. Both the 12th-generation sensor
    naming (`Temp`, iDRAC 7, for example R720) and the 13th-generation naming
    (`CPU1 Temp`/`CPU2 Temp`, iDRAC 8, for example T430, R630 and R730) are
    supported and covered by unit tests. The original add-on was tested on an
    R720; 13G hardware verification is still pending, so check the log on first
    start. On a T430, `ipmitool sdr type temperature` should show
    `CPU1 Temp`/`CPU2 Temp` on entities `3.1`/`3.2`, plus `Inlet Temp` and
    `Exhaust Temp` on `7.x`.
2.  Network connectivity from Home Assistant to each iDRAC address.
3.  IPMI over LAN enabled on each iDRAC. In the iDRAC web interface, open iDRAC
    Settings, then Network (or Connectivity), then IPMI Settings. Enable IPMI
    over LAN and set the channel privilege limit to Administrator.
4.  An MQTT broker that Home Assistant can reach. The `core-mosquitto` add-on is
    the usual choice.

## Installation

1.  In Home Assistant, open Settings, then Add-ons, then the Add-on Store.
2.  Open the three-dot menu and choose Repositories.
3.  Paste this URL and add it:

    ```
    https://github.com/sam2kb/HA-iDRAC
    ```

4.  Install "HA iDRAC Controller" from the store.

## Configuration

Most settings live in the add-on's web UI.

1.  On the add-on page, open the Configuration tab and fill in the MQTT broker
    details. The fan speed and temperature values there are global defaults for
    new servers. Save the settings.
2.  Start the add-on from the Info tab, then click Open Web UI.
3.  Open Manage Servers and use the Add New Server form. It is pre-filled with
    the global defaults.
4.  Restart the add-on after adding or editing servers. The web UI links to the
    add-on dashboard for this.

## Web UI

The dashboard shows live data for every enabled server and refreshes itself.
The Manage Servers page adds, edits and deletes servers, and sets the fan mode
and thresholds for each one.

## Entities in Home Assistant

Each server becomes a device with these entities:

- `button.<alias>_shutdown_server`
- `binary_sensor.<alias>_status` for online/offline
- `binary_sensor.<alias>_psu_status` for each power supply
- sensors for CPU temperatures, hottest CPU, inlet, exhaust, fan RPM, power and
  target fan speed

Entity IDs use the alias you give the server.

## Troubleshooting

- Check the add-on log first. Set the log level to `debug` or `trace` in the
  Configuration tab for more detail.
- For IPMI errors, confirm that IPMI over LAN is enabled and that the
  credentials in the web UI are correct.
- For MQTT errors, check the broker settings in the Configuration tab.
- If sensor values are missing or wrong, the log shows the raw SDR data at
  debug level. Open an issue with the `ipmitool sdr type temperature` lines for
  your model.

## Reporting issues

Open an issue at
[sam2kb/HA-iDRAC](https://github.com/sam2kb/HA-iDRAC/issues) and say which
add-on and version you are running.

## License

MIT. Original work by [Aesgarth](https://github.com/Aesgarth/HA-iDRAC).
