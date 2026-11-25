# Bastion

Local AI system for home network monitoring, threat evaluation, and control.

Runs on an always-on Omarchy node. Acts as the single interface for the network: chat with the AI and an AI-managed dashboard. No routine use of Tailscale admin, firewall UI, or other device interfaces.

## Status

Early design / scaffolding. See the secure home network design document for overall architecture context.

## Goals

- Continuous local monitoring of network telemetry
- Threat evaluation and categorization
- AI-managed dashboard (network health, activity, threat level, connected devices, suspicious activity)
- Direct chat interface with the AI
- Ability to trigger the physical kill switch when threat level is high enough
- Fully local inference (no mandatory cloud dependency for core function)

## Planned Structure

```
/telemetry     Collectors and parsers (syslog, DNS, SNMP, etc.)
/prompts       System prompts and decision logic
/dashboard     AI-managed dashboard
/killswitch    Integration with ESP32 physical kill switch
/docs          Design notes and threat model
```

## License

Apache 2.0
