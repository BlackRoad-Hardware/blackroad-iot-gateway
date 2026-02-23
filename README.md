# blackroad-iot-gateway

> IoT device gateway with MQTT and HTTP bridge — part of the BlackRoad Hardware platform.

## Features

- **Device Registry** — Register and track IoT devices (sensors, actuators, cameras, thermostats, locks)
- **Multi-Protocol** — MQTT, HTTP, CoAP, Modbus
- **Command Dispatch** — Queue commands to devices with acknowledgement tracking
- **Topic Subscriptions** — MQTT pub/sub routing per device
- **Firmware Tracking** — Record firmware versions and update history
- **Network Topology** — Export full device graph as JSON
- **Bulk Operations** — Send commands to multiple devices at once

## Quick Start

```bash
pip install -r requirements.txt
python iot_gateway.py stats
python iot_gateway.py list --type sensor
python iot_gateway.py topology
```

## Usage

```python
from iot_gateway import IoTGateway

gw = IoTGateway()

# Register a device
device = gw.register_device(
    mac="AA:BB:CC:DD:EE:01",
    name="Living Room Sensor",
    device_type="sensor",
    protocol="mqtt",
    ip="192.168.1.42",
    capabilities=["temperature", "humidity"]
)

# Subscribe to a topic
gw.subscribe_topic(device.id, "home/living/sensors")

# Process incoming MQTT message
gw.process_message("home/living/sensors", {"temp": 22.5, "humidity": 60})

# Send a command
cmd = gw.send_command(device.id, "set_interval", {"seconds": 30})

# Update firmware
gw.update_firmware(device.id, "2.1.0")

# Export network topology
print(gw.export_topology_json())
```

## Device Types

| Type | Description |
|------|-------------|
| `sensor` | Environmental or measurement sensor |
| `actuator` | Relay, motor, or control device |
| `camera` | Video/image capture device |
| `thermostat` | Temperature control device |
| `lock` | Access control / smart lock |

## Protocols

| Protocol | Description |
|----------|-------------|
| `mqtt` | Message Queuing Telemetry Transport |
| `http` | RESTful HTTP polling |
| `coap` | Constrained Application Protocol |
| `modbus` | Industrial serial protocol |

## Testing

```bash
pytest --tb=short -v
```

## License

Proprietary — BlackRoad OS, Inc. All rights reserved.
