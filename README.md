# HA-iDRAC: iDRAC 7/8 (12G/13G) fork

Home Assistant add-ons that monitor and control Dell PowerEdge servers over
IPMI. This is a fork of [Aesgarth/HA-iDRAC](https://github.com/Aesgarth/HA-iDRAC)
with support for 13th-generation servers such as the T430, R430, R630 and R730,
and with fixes for several reliability problems in the original add-ons.

## What this fork changes

The 13G sensor fixes are the reason the fork exists. iDRAC 8 reports CPU
temperatures as `CPU1 Temp` and `CPU2 Temp`, on processor entities `3.x`. The
original stable add-on only matched a sensor named exactly `Temp`, so it found
no CPU temperature on a T430 or R730 and left the fans on Dell auto. The parser
now uses the entity ID first and the 12G name as a fallback. PSU health comes
from `PSn Status` and `PSn PG Fail`, because the `Voltage 1` sensor the old
code expected does not exist on iDRAC 8. Inlet and exhaust sensors are matched
at the start of the sensor name, so `PSU1 Inlet Temp` can no longer replace the
chassis inlet reading.

Fan control is more careful. A 2 °C hysteresis band keeps the fans from
flapping between speeds, and between manual and Dell auto, on every poll.
Requests below 5% are clamped to 5%. If manual mode lands but the speed write
fails, the fans go straight back to Dell auto. A server that refuses fan
control with `0xc1`, `0xd4` or `0xd5` (iDRAC 9 or 10, or an account below
Administrator) is recorded once, fan control is disabled, and monitoring
continues. An optional per-server setting sends the 12G/13G OEM command that
disables Dell's default cooling response for third-party PCIe cards, then
restores Dell's setting on shutdown.

The operational fixes are less visible but matter on a home network. The BETA
add-on retries MQTT instead of stopping a worker when the broker is down, and
it re-subscribes and republishes discovery after a reconnect. The iDRAC
password is passed with `ipmitool -E`, so it no longer appears in `ps`. The
Supervisor watchdog restarts an add-on that exits, and the BETA add-on serves
`/health`, which fails only when every server worker has died; an unreachable
iDRAC or broker does not cause a restart loop. Firmware version and service
tag are read from the server and published as the MQTT device's `sw_version`
and `serial_number`. Implausible temperatures outside -20 to 125 °C and
contradictory thresholds are ignored or corrected with a warning.

The stale `multi-server` add-on from upstream was removed; the BETA add-on
already supports several servers. The unused `master_encryption_key` option
was removed too, since no code ever read it. Tests and a GitHub Actions
workflow cover the SDR parsers and the fan logic.

## Add-ons

1. HA iDRAC Controller (`ha_idrac_controller`, version 0.2.0). One server,
   configured from the add-on Configuration tab. Simple three-tier fan control
   with hysteresis, monitoring and MQTT discovery.
2. HA iDRAC Controller BETA (`ha_idrac_controller_development`, version 0.2.0).
   Several servers, per-server fan modes (simple, curve, target temperature),
   monitor-only mode, PSU and connectivity sensors, and a graceful shutdown
   button. This is the one to install for a T430.

Both add-ons use the same fan commands as the original project:
`raw 0x30 0x30 0x01 0x00` for manual mode and `raw 0x30 0x30 0x02 0xff <pct>`
for the speed. That is the standard 12G/13G Dell sequence. This fork does not
target iDRAC 9 (14G) or newer; the commands and sensor layout differ there.

## Installation

1. In Home Assistant, open Settings, then Add-ons, then the Add-on Store.
2. Open the three-dot menu and choose Repositories.
3. Paste this URL:

   ```
   https://github.com/sam2kb/HA-iDRAC
   ```

4. Install HA iDRAC Controller BETA for a T430.
5. On the iDRAC, open iDRAC Settings, then Network, then IPMI Settings. Enable
   IPMI over LAN, set the privilege limit to Administrator, and make sure Home
   Assistant can reach UDP port 623 on the iDRAC.
6. Start Mosquitto, or another MQTT broker, before starting the add-on. Fill in
   the MQTT settings and save. Start the add-on, add the server in the web UI
   under Manage Servers, then restart the add-on once.

Upstream also published a third add-on called `multi-server`, an older
snapshot of the BETA code. It is not part of this fork. If you added the
upstream repository earlier, remove it before adding this one, because both
repositories use the same add-on slugs.

## Configuration notes

- Fan values are percentages of the iDRAC's maximum fan speed. Start with
  `base_fan_speed_percent: 20`, `low_temp_threshold: 45`,
  `high_temp_fan_speed_percent: 50` and `critical_temp_threshold: 65`, then
  watch the temperatures for a day before lowering anything.
- The BETA add-on stores servers in `/data/servers_config.json`, which is
  root-only on HA OS, with the iDRAC password in plain text. Do not reuse an
  administrator password from another system.
- The account needs Administrator privilege. A lower account gets the fan
  commands refused with `0xd4`, and the add-on switches to monitoring only.
- On stop, the BETA add-on hands the fans back to Dell's automatic profile. If
  the container is killed without running its shutdown handler, the iDRAC can
  stay in manual mode at the last speed. The Supervisor watchdog restarts the
  add-on in that case and the configured profile is applied again.
- Sensor naming differs between firmware versions. If the log reports "No CPU
  temperature sensor found", set `log_level: debug`, find the raw SDR output in
  the log, and open an issue with the `sdr type temperature` lines for your
  model.

## Known limitations

- The third-party PCIe cooling-response command is a Dell OEM command for
  12G/13G. Some firmware does not accept it; the add-on logs that once and
  leaves the setting untouched. It is only applied when fan control is enabled.
- iDRAC 9 (14G) and newer are outside the scope of this fork.
- A `SIGKILL`, an OOM kill or a host power loss cannot run the shutdown
  handler. The iDRAC can stay in manual mode until the add-on starts again.
- The 13G parsing is covered by unit tests and a fake-IPMI run, not by a
  physical T430. Watch the first start on real hardware.

## Development

```sh
python -m pytest -q
```

The tests do not need an iDRAC, Flask or paho. They cover 12G/13G SDR parsing,
fan RPM, power and PSU parsing, the fan floor and the hysteresis logic. GitHub
Actions runs the same suite on every push.

## License

MIT, the same license as the upstream project. Original work by
[Aesgarth](https://github.com/Aesgarth/HA-iDRAC).
