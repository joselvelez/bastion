# Bastion

Primary project repository for the secure home network design and its local AI monitoring system.

**https://github.com/joselvelez/bastion**

## What lives here

- **Network design document** (`network-design.md`) — first-class document at the repo root
- Local AI system (telemetry, prompts, decision logic, dashboard, kill-switch integration)
- All related scripts, configs, and documentation

## Status

Early design / scaffolding.

## Goals

- Continuous local monitoring of network telemetry
- Threat evaluation and categorization
- AI-managed dashboard (network health, activity, threat level, connected devices, suspicious activity)
- Direct chat interface with the AI
- Ability to trigger the physical kill switch when threat level is high enough
- Fully local inference (no mandatory cloud dependency for core function)

## Structure

```
/network-design.md   Secure home network design (first-class)
/docs                Additional documentation
/telemetry           Collectors and parsers (syslog, DNS, SNMP, etc.)
/prompts             System prompts and decision logic
/dashboard           AI-managed dashboard
/killswitch          Integration with ESP32 physical kill switch
```

## License

Apache 2.0
