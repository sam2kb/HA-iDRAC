## 0.4.0 - 2026-10-18

- Removed the legacy single-server add-on. This is now the only add-on in the
  repository.
- Added `MIGRATION.md` for people moving from an older HA-iDRAC build. It maps
  the old settings to the new server form and lists the behaviour changes.
- No functional changes to the add-on code.

## 0.3.0 - 2026-10-18

- Promoted the multi-server build to the main add-on: name "HA iDRAC
  Controller", slug `ha_idrac_controller`, version 0.3.0.
- Renamed the original single-server build to "HA iDRAC Controller (Legacy)",
  slug `ha_idrac_controller_legacy`, version 0.2.1.
- No functional changes to either add-on. If you installed the 0.2.0 BETA
  (`ha_idrac_controller_development`), remove it and install the main add-on.
  Its `/data` volume is tied to the old slug, so the server list does not carry
  over.

## 0.2.0 - 2026-10-18

First tagged release of the fork. It covers iDRAC 8 (13G) support and the
hardening work from the 0.1.x development builds:

- CPU temperatures are read from the IPMI processor entities (`3.x`), so
  `CPU1 Temp` and `CPU2 Temp` work on a T430, R630 or R730, with the 12G-style
  `Temp` name kept as a fallback.
- PSU health uses `PSn Status` and `PSn PG Fail`; the old `Voltage n` check
  does not exist on iDRAC 8.
- Inlet and exhaust names are anchored, so a PSU inlet sensor cannot replace
  the chassis inlet reading.
- 2 °C hysteresis on the fan thresholds, a 5% fan floor, and a hand-back to
  Dell auto when a speed write fails.
- Refused fan control (0xc1/0xd4/0xd5) is detected once and monitoring
  continues.
- Optional per-server third-party PCIe cooling response, restored on shutdown.
- MQTT retry and re-subscribe, `ipmitool -E` password handling, `/health` for
  the Supervisor watchdog, firmware and serial in the MQTT device.
- 27 tests and a GitHub Actions workflow.

## 0.1.0-dev.15 - 2026-10-18

- Inlet/exhaust sensors are matched at the start of the sensor name, so
  `PSU1 Inlet Temp` can no longer replace the chassis inlet reading.
- If the manual-mode command lands but the fan-speed command fails, the fans
  are handed back to Dell auto instead of staying at the previous speed.
- Temperature readings outside -20 to 125 °C are ignored.
- A server that refuses fan control with 0xc1/0xd4/0xd5 (iDRAC 9 or 10, or an
  account below Administrator) is detected once. Fan control is disabled and
  monitoring continues.
- New per-server "Third-party PCIe cooling" option for non-Dell cards, using
  the 12G/13G OEM command; Dell's setting is restored on shutdown.
- iDRAC firmware version and server serial number are read at startup and
  published as the MQTT device's `sw_version` and `serial_number`.
- Contradictory thresholds (critical at or below low, high below base) are
  corrected with a warning.
- The Supervisor watchdog probes `/health`, which fails only when every server
  worker has died. An unreachable iDRAC or MQTT broker does not trigger a
  restart loop.

## 0.1.0-dev.14 - 2026-10-18

- CPU temperatures are detected by IPMI processor entity (`3.x`), so
  `CPU1 Temp` and `CPU2 Temp` on T430, R630 and R730 are found. Before this,
  only a sensor literally named `Temp` (12G) matched and fan control never
  engaged on 13G servers.
- PSU status uses `PSn Status` presence and `PSn PG Fail` instead of a
  `Voltage n` sensor that iDRAC 8 does not expose.
- 2 °C hysteresis on the base/high and critical thresholds.
- Fan speeds below 5% are clamped to 5%.
- MQTT connection failures no longer stop a server worker. Reconnects
  re-subscribe and republish discovery.
- The iDRAC password is passed with `ipmitool -E` and `IPMI_PASSWORD` instead
  of the command line.
- Removed the unused `master_encryption_key` and `temperature_unit` options.
- Enabled the Supervisor watchdog.
- Added pytest coverage for SDR parsing (12G/13G), fan RPM, power, PSU status
  and the hysteresis logic, plus a GitHub Actions workflow.

## 0.1.0-dev.13 - 2025-09-24

- Added monitor-only mode on a per-server basis.
- Power consumption is parsed from the "System Level" sensor on older iDRAC 6
  servers.
- Added a Fan Control dropdown to the Add and Edit Server pages.

## 0.1.0-dev.12 - 2025-09-20

- Previous development release.
