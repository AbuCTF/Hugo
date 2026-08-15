---
title: "Proserv"
role: "Cloud Operations Engineer"
period: "May 2024 – Jun 2025"
location: "Chennai, India · Hybrid"
images:
  - "/images/work/proserv-mqtt.jpg"
  - "/images/work/proserv-domain.jpg"
draft: false
---

Part-time cloud operations at Proserv, over fourteen months.

## The project

An IoT-driven hydraulic actuation and monitoring system for the petroleum industry: remote-controlled, with Raspberry Pi controllers on a centralised LAN reporting into a cloud server.

Petroleum sites are the awkward case for IoT. The equipment is remote, connectivity is bad, and the thing being actuated is hydraulic, so a dropped message isn't a refreshed dashboard, it's a valve in the wrong state.

## MQTT

Which is why the transport is MQTT rather than HTTP. Publish/subscribe over a broker, small payloads, and quality-of-service levels that let you say what happens when the link drops mid-message.

The screenshot above is the plumbing being proven end to end: `mosquitto_sub` holding a subscription on `device/status` on one box while `mosquitto_pub` publishes to the same topic on another, both pointed at the broker.

```bash
mosquitto_sub -h 192.168.1.8 -t device/status
mosquitto_pub -h 192.168.1.8 -t device/status -m "Hello from RPi"
```

Two terminals and a topic name is the entire test, and it's the one worth doing first. Before any device code exists, you want to know the broker is reachable, the topic is right and the message arrives.

## Infrastructure

VPS management and SSH access to the remote units. Standard, and most of the actual job: keeping the boxes reachable and the access to them controlled.

The second screenshot is domain reconnaissance during that work: registration date, org size, location. Knowing what you're actually connecting to is part of connecting to it.
