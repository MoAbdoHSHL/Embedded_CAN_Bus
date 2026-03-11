# Embedded_CAN_Bus to KUKSA.Val Databroker Pipeline

This project implements a simple **CAN → Vehicle Signal Specification (VSS) pipeline** using Python and the **KUKSA Databroker**.

The system listens to CAN frames, decodes signals based on predefined rules, and publishes them to the KUKSA Databroker.

This architecture simulates a simplified **vehicle data pipeline** often used in automotive software stacks.

---

# Architecture

The system is composed of three Embedded modules:

1. **CAN Listener**
2. **CAN Processor**
3. **KUKSA Publisher**

Data flows through the system as follows:

```
CAN Bus (vcan0) as Virtual CAN till I get Access
      │
      ▼
CAN Listener
      │
      ▼
CAN Processor (decode signals)
      │
      ▼
KUKSA Publisher
      │
      ▼
KUKSA Databroker
```

---

# Project Components

## 1. CAN Listener (`can_listener.py`)

Responsibilities:

* Connect to the virtual CAN interface `vcan0` or the REAL CAN later via SSH
* Listen for CAN messages
* Filter relevant CAN IDs
* Push received frames to a processing queue

Example output:

```
Listener: RX 0x65 12345678
```

---

## 2. CAN Processor (`can_processor.py`)

Responsibilities:

* Retrieve CAN frames from the queue
* Decode the signals according to the decoding table
* Apply scaling and byte ordering
* Send decoded values to the publisher queue

Example output:

```
Processor: Decoded vehicle_speed = 4660
```

---

## 3. KUKSA Publisher (`kuksaval_publisher.py`)

Responsibilities:

* Connect to the KUKSA Databroker via the Local host 55555
* Receive decoded signals
* Publish values to the corresponding VSS path

Example output:

```
Publisher: Sent Vehicle.Speed = 4660
```

---

# CAN Signal Mapping

The decoding rules are defined in `CAN_DECODING` from the CSV:

| CAN ID | Signal Name         | Bytes | Endian | Type   | Scale |
| ------ | ------------------- | ----- | ------ | ------ | ----- |
| 0x009  | target_speed        | (0,1) | little | uint16 | 0.1   |
| 0x06A  | precise_speed       | (0,3) | big    | uint32 | 0.01  |
| 0x065  | vehicle_speed       | (0,1) | big    | uint16 | 1.0   |
| 0x066  | cumulative_distance | (0,3) | big    | uint32 | 1.0   |
| 0x06B  | raw_wheel_RPM       | (0,1) | big    | uint16 | 1.0   |
| 0x188  | wheel_freq          | (0,1) | little | uint16 | 1.0   |
| 0x067  | SOC                 | (0,1) | big    | uint16 | 1.0   |
| 0x068  | cumulative_voltage  | (0,3) | big    | uint32 | 0.1   |
| 0x069  | current             | (0,3) | big    | uint32 | 0.1   |

---

# Requirements

Software required:

* Python 3.10+
* Docker
* python-can
* kuksa-client
* can-utils

Install Python dependencies:

```
pip install python-can kuksa-client
```

Install CAN utilities:

```
sudo apt install can-utils
```

---

# Setup Virtual CAN Interface

Create the virtual CAN interface used for testing.

```
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0
```

Verify:

```
ip link show vcan0
```

---

# Start KUKSA Databroker

Run the databroker container:

```
sudo docker run -it -p 55555:55555 ghcr.io/eclipse-kuksa/kuksa-databroker:main
```

The server will start and listen on:

```
localhost:55555
```

---

# Run the Pipeline

Start the Python pipeline:

```
python main.py
```

You should see:

```
Listener: Listening on vcan0...
Processor: Waiting for CAN messages...
Publisher: Connecting to KUKSA Databroker...
```

---

# Simulate CAN Messages

You can simulate CAN messages using `cansend`.

Example:

```
cansend vcan0 065#12345678
```

Or run a continuous simulator:

```
while true; do
  cansend vcan0 009#0102
  cansend vcan0 06A#00000064
  cansend vcan0 065#12345678
  cansend vcan0 066#01020304
  cansend vcan0 06B#0A0B
  cansend vcan0 188#0203
  cansend vcan0 067#0032
  cansend vcan0 068#01020304
  cansend vcan0 069#00010203
  sleep 1
done
```

---

# Example Output

```
Listener: RX 0x65 12345678
Processor: Decoded vehicle_speed = 4660
Publisher: Sent Vehicle.Speed = 4660
```

---

# Future Improvements

Possible improvements for this project:

* Load CAN decoding rules from a CSV or DBC file
* Add logging system
* Implement error handling and reconnection
* Add signal visualization
* Integrate with real CAN hardware
* Support multiple CAN buses

---

# Author

Mohamed Abdo
