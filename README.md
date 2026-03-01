# blackroad-iot-gateway

> Production-grade IoT device gateway with MQTT and HTTP bridge — part of the [BlackRoad Hardware](https://blackroadhardware.com) platform.

![CI](https://github.com/BlackRoad-Hardware/blackroad-iot-gateway/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/badge/python-3.10%20%7C%203.11%20%7C%203.12-blue)
![License](https://img.shields.io/badge/license-Proprietary-red)

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Installation](#installation)
4. [Quick Start](#quick-start)
5. [Usage](#usage)
   - [Device Management](#device-management)
   - [MQTT Messaging](#mqtt-messaging)
   - [Command Dispatch](#command-dispatch)
   - [Firmware Updates](#firmware-updates)
   - [Network Topology](#network-topology)
   - [Statistics](#statistics)
6. [API Reference](#api-reference)
   - [IoTGateway](#iotgateway-class)
   - [Device](#device-dataclass)
   - [CommandRecord](#commandrecord-dataclass)
   - [MQTTMessage](#mqttmessage-dataclass)
   - [TopicSubscription](#topicsubscription-dataclass)
   - [Enumerations](#enumerations)
7. [CLI Reference](#cli-reference)
8. [Database Schema](#database-schema)
9. [Device Types](#device-types)
10. [Protocols](#protocols)
11. [Testing](#testing)
12. [License](#license)

---

## Overview

`blackroad-iot-gateway` is the core device-management and message-routing library for the BlackRoad Hardware platform. It provides a single `IoTGateway` class that manages device registration, MQTT pub/sub routing, command queuing with acknowledgement tracking, firmware version history, and full network-topology export — all backed by a local SQLite database with WAL journaling.

---

## Features

| Feature | Description |
|---------|-------------|
| **Device Registry** | Register and track IoT devices (sensors, actuators, cameras, thermostats, locks) |
| **Multi-Protocol** | First-class support for MQTT, HTTP, CoAP, and Modbus |
| **Command Dispatch** | Queue commands to devices with full status lifecycle (`pending → sent → acked / failed`) |
| **Topic Subscriptions** | MQTT pub/sub routing — subscribe devices to topics and route incoming messages |
| **Firmware Tracking** | Record firmware versions and maintain complete update history per device |
| **Network Topology** | Export full device graph as structured JSON |
| **Bulk Operations** | Send the same command to multiple devices in a single call |
| **Device Status** | Rich status summaries including pending commands, subscriptions, and last-seen time |
| **Heartbeat** | Lightweight liveness tracking — update `last_seen` without a full re-registration |

---

## Installation

**Requirements:** Python 3.10 or later.

```bash
pip install -r requirements.txt
```

The gateway uses only the Python standard library at runtime (`sqlite3`, `uuid`, `hashlib`, `json`, `logging`, `dataclasses`). The `requirements.txt` lists test-only dependencies (`pytest`, `pytest-cov`).

---

## Quick Start

```bash
# Show gateway statistics
python iot_gateway.py stats

# List all registered sensors
python iot_gateway.py list --type sensor

# List only online devices
python iot_gateway.py list --online

# Export network topology as JSON
python iot_gateway.py topology
```

---

## Usage

### Device Management

```python
from iot_gateway import IoTGateway

gw = IoTGateway()  # uses iot_gateway.db in the current directory

# Register a new device (or update if MAC already exists)
device = gw.register_device(
    mac="AA:BB:CC:DD:EE:01",
    name="Living Room Sensor",
    device_type="sensor",   # sensor | actuator | camera | thermostat | lock
    protocol="mqtt",        # mqtt | http | coap | modbus
    ip="192.168.1.42",
    firmware_version="1.0.0",
    capabilities=["temperature", "humidity"],
)

# Retrieve a device by ID
device = gw.get_device(device.id)

# List all devices (optional filters)
all_devices   = gw.list_devices()
sensors_only  = gw.list_devices(device_type="sensor")
online_only   = gw.list_devices(online_only=True)

# Mark a device online / offline
gw.mark_online(device.id)
gw.mark_offline(device.id)

# Heartbeat — update last_seen without changing other fields
gw.heartbeat(device.id)

# Rich status summary
status = gw.get_device_status(device.id)
# {
#   "device": {...},
#   "pending_commands": 2,
#   "topic_subscriptions": 3,
#   "last_message_topic": "home/living/sensors",
#   "last_message_at": "2024-01-15T10:30:00+00:00"
# }
```

### MQTT Messaging

```python
# Subscribe a device to a topic
sub = gw.subscribe_topic(device.id, "home/living/sensors")

# Process an incoming MQTT message (persists it and updates last_seen)
msg = gw.process_message(
    "home/living/sensors",
    {"temp": 22.5, "humidity": 60},
    qos=1,
    retained=False,
)

# Retrieve recent messages for a topic
messages = gw.get_topic_messages("home/living/sensors", limit=50)

# Unsubscribe a device from a topic
removed = gw.unsubscribe_topic(device.id, "home/living/sensors")
```

### Command Dispatch

```python
# Send a command to a device
cmd = gw.send_command(device.id, "set_interval", {"seconds": 30})
# cmd.status == CommandStatus.PENDING

# Acknowledge (device confirmed execution)
cmd = gw.acknowledge_command(cmd.id)
# cmd.status == CommandStatus.ACKNOWLEDGED

# Mark as failed
cmd = gw.fail_command(cmd.id)
# cmd.status == CommandStatus.FAILED

# List all pending commands for a device
pending = gw.get_pending_commands(device.id)

# Send the same command to many devices at once
results = gw.bulk_update([device_a.id, device_b.id], "reboot", {"delay": 5})
```

### Firmware Updates

```python
# Record a firmware upgrade
result = gw.update_firmware(device.id, "2.1.0")
# {
#   "update_id": "...",
#   "device_id": "...",
#   "old_version": "1.0.0",
#   "new_version": "2.1.0",
#   "updated_at": "2024-01-15T10:30:00+00:00"
# }

# Retrieve the full firmware history for a device
history = gw.get_firmware_history(device.id)
```

### Network Topology

```python
# Export topology as a Python dict
topology = gw.export_topology()

# Export topology as a JSON string (optionally write to a file)
json_str = gw.export_topology_json()
gw.export_topology_json(path="topology.json")
```

### Statistics

```python
stats = gw.get_stats()
# {
#   "total_devices": 42,
#   "online_devices": 38,
#   "offline_devices": 4,
#   "total_messages": 12500,
#   "total_commands": 890,
#   "pending_commands": 7,
#   "device_types": {"sensor": 20, "actuator": 10, ...},
#   "protocols": {"mqtt": 35, "http": 7}
# }
```

---

## API Reference

### `IoTGateway` Class

```
IoTGateway(db_path: Path = Path("iot_gateway.db"))
```

The central gateway object. Creates and migrates the SQLite database on first use.

| Method | Signature | Description |
|--------|-----------|-------------|
| `register_device` | `(mac, name, device_type, protocol, ip, firmware_version, capabilities) → Device` | Register or upsert a device by MAC address |
| `get_device` | `(device_id: str) → Device` | Retrieve a device; raises `ValueError` if not found |
| `list_devices` | `(device_type?, online_only?) → List[Device]` | List devices with optional filters |
| `get_device_status` | `(device_id: str) → dict` | Rich status summary for a single device |
| `mark_online` | `(device_id: str) → None` | Set `online=True` and update `last_seen` |
| `mark_offline` | `(device_id: str) → None` | Set `online=False` |
| `heartbeat` | `(device_id: str) → None` | Update `last_seen` without other changes |
| `update_firmware` | `(device_id, version) → dict` | Record a firmware version change |
| `get_firmware_history` | `(device_id: str) → List[dict]` | Full firmware update history |
| `send_command` | `(device_id, command, params?) → CommandRecord` | Enqueue a command with status `pending` |
| `acknowledge_command` | `(command_id: str) → CommandRecord` | Transition command to `acked` |
| `fail_command` | `(command_id: str) → CommandRecord` | Transition command to `failed` |
| `get_pending_commands` | `(device_id: str) → List[CommandRecord]` | All pending commands for a device |
| `bulk_update` | `(device_ids, command, params?) → List[CommandRecord]` | Send the same command to multiple devices |
| `subscribe_topic` | `(device_id, topic) → TopicSubscription` | Subscribe a device to an MQTT topic (idempotent) |
| `unsubscribe_topic` | `(device_id, topic) → bool` | Remove a topic subscription |
| `process_message` | `(topic, payload, qos?, retained?) → MQTTMessage` | Persist a message and update subscribed devices |
| `get_topic_messages` | `(topic, limit?) → List[dict]` | Retrieve recent messages for a topic |
| `export_topology` | `() → dict` | Full network topology as a Python dict |
| `export_topology_json` | `(path?) → str` | Topology as a JSON string, optionally saved to disk |
| `get_stats` | `() → dict` | Aggregate gateway statistics |
| `generate_device_hash` | `(mac: str) → str` *(static)* | 16-char SHA-256 fingerprint derived from MAC address |

### `Device` Dataclass

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | UUID primary key |
| `mac` | `str` | Hardware MAC address (unique) |
| `name` | `str` | Human-readable device name |
| `type` | `DeviceType` | Device category enum |
| `ip` | `str` | Last known IP address |
| `protocol` | `Protocol` | Communication protocol enum |
| `last_seen` | `Optional[str]` | ISO-8601 timestamp of last activity |
| `firmware_version` | `str` | Current firmware version string |
| `online` | `bool` | Whether the device is currently online |
| `capabilities` | `List[str]` | List of supported capability tags |

### `CommandRecord` Dataclass

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | UUID primary key |
| `device_id` | `str` | Target device ID |
| `command` | `str` | Command name |
| `params` | `Dict[str, Any]` | Command parameters |
| `status` | `CommandStatus` | Current status enum |
| `created_at` | `str` | ISO-8601 creation timestamp |
| `updated_at` | `str` | ISO-8601 last-updated timestamp |

### `MQTTMessage` Dataclass

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `topic` | `str` | — | MQTT topic string |
| `payload` | `Dict[str, Any]` | — | Message payload |
| `qos` | `int` | `0` | Quality-of-service level (0, 1, or 2) |
| `retained` | `bool` | `False` | Whether the broker retains this message |
| `timestamp` | `str` | UTC now | ISO-8601 creation timestamp |

### `TopicSubscription` Dataclass

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | UUID primary key |
| `device_id` | `str` | Subscribing device ID |
| `topic` | `str` | MQTT topic string |
| `created_at` | `str` | ISO-8601 subscription timestamp |

### Enumerations

**`DeviceType`**

| Value | Description |
|-------|-------------|
| `sensor` | Environmental or measurement sensor |
| `actuator` | Relay, motor, or control device |
| `camera` | Video/image capture device |
| `thermostat` | Temperature control device |
| `lock` | Access control / smart lock |

**`Protocol`**

| Value | Description |
|-------|-------------|
| `mqtt` | Message Queuing Telemetry Transport |
| `http` | RESTful HTTP polling |
| `coap` | Constrained Application Protocol |
| `modbus` | Industrial serial protocol |

**`CommandStatus`**

| Value | Description |
|-------|-------------|
| `pending` | Command has been queued, not yet delivered |
| `sent` | Command has been dispatched to the device |
| `acked` | Device confirmed successful execution |
| `failed` | Command delivery or execution failed |

---

## CLI Reference

```
python iot_gateway.py <command> [options]
```

| Command | Options | Description |
|---------|---------|-------------|
| `stats` | — | Print aggregate gateway statistics as JSON |
| `topology` | — | Print full network topology as JSON |
| `list` | `--type <type>`, `--online` | List registered devices; filter by type or online status |

**Examples**

```bash
# Show statistics
python iot_gateway.py stats

# List all thermostats
python iot_gateway.py list --type thermostat

# List only online devices
python iot_gateway.py list --online

# Export topology
python iot_gateway.py topology
```

---

## Database Schema

The gateway uses a local SQLite database (default: `iot_gateway.db`) with WAL journaling and foreign-key enforcement.

| Table | Primary Key | Description |
|-------|-------------|-------------|
| `devices` | `id` (UUID) | Registered device registry |
| `mqtt_messages` | `id` (UUID) | Persisted MQTT message log |
| `commands` | `id` (UUID) | Command queue and history |
| `topic_subscriptions` | `id` (UUID) | Device-to-topic subscription map |
| `firmware_updates` | `id` (UUID) | Firmware version change history |

**Indexes**

| Index | Table | Column(s) |
|-------|-------|-----------|
| `idx_devices_mac` | `devices` | `mac` |
| `idx_commands_device` | `commands` | `device_id` |
| `idx_msgs_topic` | `mqtt_messages` | `topic` |
| `idx_subs_device` | `topic_subscriptions` | `device_id` |

To use a custom database path:

```python
from pathlib import Path
from iot_gateway import IoTGateway

gw = IoTGateway(db_path=Path("/data/production.db"))
```

---

## Device Types

| Type | Description |
|------|-------------|
| `sensor` | Environmental or measurement sensor |
| `actuator` | Relay, motor, or control device |
| `camera` | Video/image capture device |
| `thermostat` | Temperature control device |
| `lock` | Access control / smart lock |

---

## Protocols

| Protocol | Description |
|----------|-------------|
| `mqtt` | Message Queuing Telemetry Transport |
| `http` | RESTful HTTP polling |
| `coap` | Constrained Application Protocol |
| `modbus` | Industrial serial protocol |

---

## Testing

```bash
# Run full test suite with coverage report
pytest --tb=short --cov=. --cov-report=term-missing -v
```

Tests cover device registration, firmware updates, command lifecycle, MQTT pub/sub, topology export, statistics, and CLI behaviour. A temporary in-memory SQLite database is used per test so tests are fully isolated.

---

## License

Proprietary — BlackRoad OS, Inc. All rights reserved.
