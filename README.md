# ESPHome Smart Water Heater Controller

This project transforms a standard electric water heater into a smart, energy‑efficient (hopefully), and safe appliance using ESPHome on an ESP32‑S2. It is designed to run autonomously when Home Assistant or Wi‑Fi is unavailable, while prioritizing public health (Legionella prevention) and hardware safety. The controller sits in between mains and the power in going to the water heater.

Note: No hardware design documentation is provided. This repo assumes you will design and build your own hardware (wiring, enclosure, protection, clearances) to your local electrical code and safety standards.

## Features
- Salubrity (Legionella Prevention): Enforces a sanitization cycle at least once every 24 hours. When both upper and lower probes reach 60°C, holds for 30 minutes.
- Vacation Mode: Persistent switch that immediately forces the heater OFF, overriding other modes except critical failsafes. Salubrity check-up is ignored.
- Manual Override Mode: Full manual control via Home Assistant, with automatic fallback to autonomous logic if API connectivity is lost, and a safety auto‑reset if HA stays disconnected too long. Salubrity check-up is still running.
- Scheduled ON Mode: Two configurable time windows that force the heater ON for peak availability (including schedules that cross midnight).
- Off‑Peak Economy Mode: Default mode that heats only when needed; includes overshoot logic, a minimum relay toggle delay, and a boot safety delay before the automatic control logic starts making heating decisions.
- Stand‑Alone Resilience: Hardware RTC (DS3231) keeps accurate time across power/network loss; schedules and salubrity run offline using the RTC.
- Failsafe Logic: If system time is invalid (dead RTC + no network) or probes fail/stall, the controller forces the relay into its configured safe state (in this build, wired so the heater remains powered even if the controller crashes/malfunctions).
- Monitoring & Diagnostics:
  - 16×2 LCD pages for status (temps, amps, relay reason, connectivity, last salubrity time).
  - Connectivity status indicators and icons.
  - Current monitoring with a latching fault sensor: turns ON after 5s continuous out‑of‑range and persists across reboots until manually reset.
  - RTC internal temperature reporting (read directly from the DS3231 over I2C).
  - Relay cycle counter (diagnostic) to track relay wear.
- Recovery & Maintenance:
  - Built-in fallback AP + captive portal if Wi‑Fi cannot connect.
  - Lightweight onboard web server for quick local inspection.
  - Periodic “maintenance” resync from the hardware RTC to reduce drift.
  - Factory reset UX: long‑press GPIO0 to trigger reset, plus an LCD boot/reset screen.

### Mode Precedence
- Salubrity preempts all other modes (including Manual Override and Schedules) to ensure public‑health compliance.
- Only in Vacation Mode is salubrity deactivated; while Vacation Mode is ON, the heater is forced OFF and salubrity cycles are not executed.

## Hardware Used 
- AC‑DC 220V → 5V DC converter, galvanically isolated
- ESP32‑S2 Lolin Mini
- DHT11 (for enclosure temperature and humidity)
- Two Dallas 1‑Wire temperature probes (e.g., DS18B20) for upper and lower tank temps
- Single CT clamp, 30 A
- Dual 30 A relay board, both relays wired to toggle simultaneously
- RTC DS3231 module, modified to supply 3.3 V to the RTC IC (two diodes in series on supply)
- LCD HD44780 16×2 with I2C backpack (PCF8574)
- "Boot" button on Lolin S2 Mini repurposed as a factory reset button (GPIO0)

Recommended extras (design‑dependent, not provided): appropriate fusing, contactor/relay isolation, pull‑ups for 1‑Wire/I2C as needed, ferrules, DIN‑rail enclosure, strain relief, and mains‑rated wiring/clearances.

## Safety Notes
- Mains electricity is dangerous. Build only if you are qualified and comply with local electrical codes.
- Provide proper isolation, creepage/clearance, fusing, and over‑current protection.
- Verify relay/contact ratings and thermal behavior under load.
- The author does not provide hardware wiring schematics; you are responsible for your design.
- Check your local regulations on water heater salubrity guidelines 

## Configure & Flash
1. Install ESPHome CLI.
2. Adjust substitutions and secrets.
3. Build and upload.

```bash
# Inside the repo folder
esphome run alimentation-chauffe-eau.yaml
# or, to just compile
esphome compile alimentation-chauffe-eau.yaml
```

## Key Configuration Points
- Substitutions let you tune temperatures, timing, and current ranges without editing logic.
- RTC synchronization: SNTP updates the DS3231 when available; otherwise the RTC drives schedules.
- Anti‑toggle timing prevents rapid relay cycling.
- Manual override respects HA connectivity; if HA is down, autonomous logic resumes.
- Manual state changes can be queued when heating is temporarily disallowed (e.g., during boot delay, salubrity cycle, or failsafe) and applied when safe.
- Salubrity has highest priority and runs regardless of Manual Override; only Vacation Mode suppresses salubrity.
