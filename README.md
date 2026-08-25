# WIFI/4G GateWayBLE Cloud Protocol Documentation

Version 2.0

## Table of Contents

1. Product Overview
2. System Architecture
3. Cloud Uplink Data Protocol
4. Gateway Heartbeat Frame
5. Checksum Algorithm
6. Cloud Downlink Data Protocol
7. MQTT Configuration
8. Server Parsing Examples
9. Integration Guide

---

## 1. Product Overview

WIFI4GGateWayBLE is a multi-protocol IoT gateway. It connects up to **19 BLE sensor peripherals** (SP, ECG, P1H, PO, etc.) and uploads the collected data to a cloud server over **WiFi / 4G**.

Core capabilities:

- **Multi-protocol uplink**: supports UDP, TCP, and MQTT. Each protocol can be enabled or disabled independently.
- **Data aggregation**: sensor data is buffered and aggregated on the gateway, then uploaded periodically or immediately.
- **Gateway heartbeat**: sends a heartbeat frame to the cloud every 3 seconds with the gateway serial number.
- **Downlink passthrough**: the cloud can send commands to the gateway over UDP/TCP, which are then forwarded to sensors over BLE.

---

## 2. System Architecture

```text
                     Cloud Server
              (UDP / TCP / MQTT Broker)
                        |
        UDP             TCP            MQTT
   (uplink+downlink)(uplink+downlink)(uplink:publish)
                        |
             WIFI4GGateWayBLE Gateway
                        |
      WiFi / 4G module      BLE Central role (<= 19 sensors)
      UDP/TCP/MQTT          aggregation / heartbeat / downlink
                        |
                       BLE
                        |
   BLE sensor peripherals: SP / ECG / P1H / PO ...
```

### Data Flows

| Direction | Path | Protocol |
|---|---|---|
| Uplink | BLE sensor -> Gateway (aggregation) -> UDP/TCP/MQTT -> Cloud | UDP / TCP / MQTT Publish |
| Downlink | Cloud -> UDP/TCP -> Gateway -> BLE -> Sensor | UDP / TCP |
| Heartbeat | Gateway -> UDP/TCP -> Cloud | UDP / TCP |

---

## 3. Cloud Uplink Data Protocol

UDP and TCP use the **same binary frame format**. MQTT uses the same payload format, but the payload is encoded as a hexadecimal string.

### 3.1 Aggregated Frame Overview

```text
0xAA | GW_ID[2] | Frame_TS[8] | PktCnt[2] | Per-packet data... | Checksum[2]
```

Frame length is variable. It depends on the number of BLE packets and each packet data size. The maximum frame size is **1024 bytes**.

### 3.2 Frame Header

| Offset | Length | Field | Endianness | Description |
|---:|---:|---|---|---|
| 0 | 1 | Frame Header | - | Fixed value `0xAA`, frame start marker |
| 1 | 2 | Gateway ID | Big-endian | Gateway device ID, currently `0x0001` |
| 3 | 8 | Frame Timestamp | Big-endian | Frame-level UTC millisecond timestamp (uint64) |
| 11 | 2 | Packet Count | Big-endian | Number of BLE data packets in this frame |

### 3.3 Per-Packet Data Format

```text
SN[15] | RSSI[1] | Packet_TS[8] | DataLen[2] | Data[N]
```

| Relative Offset | Length | Field | Endianness | Description |
|---:|---:|---|---|---|
| 0 | 15 | Serial Number | Raw | Sensor serial number, ASCII, right-padded with `0x00` |
| 15 | 1 | RSSI | - | Received signal strength, signed int8, dBm |
| 16 | 8 | Packet Timestamp | Big-endian | UTC millisecond timestamp when this packet was received |
| 24 | 2 | Data Length | Big-endian | Payload length, `0 ~ 512` |
| 26 | N | Data | Raw | Original bytes reported by the BLE sensor |

> The SN field is fixed 15 bytes. If the serial number is shorter than 15 characters, the right side is padded with `0x00`.
>
> Example: `"SP001"` is stored as:
>
> ```text
> "SP001\0\0\0\0\0\0\0\0\0\0\0"
> ```

### 3.4 Frame Length Calculation

```text
Frame length = 13 (header) + sum(26 + DataLen_i) for each packet + 2 (checksum)
```

### 3.5 Full Frame Example

Assume:

- `GW_ID = 0x0001`
- Frame timestamp = `1700000000000`
- Two BLE packets are received.

Packet 1:

- SN = `SP0000000001`
- RSSI = `-45`
- Timestamp = `1700000000000`
- Data = `0x48 0x65 0x6C` (`"Hel"`)

Packet 2:

- SN = `ECG000000002`
- RSSI = `-60`
- Timestamp = `1700000000100`
- Data = `0x6C 0x6F` (`"lo"`)

Example frame:

```text
Offset  Data                                        Description
------  ------------------------------------------  ------------------------------
00      AA                                          Frame header
01      00 01                                       GW_ID = 0x0001 (big-endian)
03      00 00 01 8B C1 E4 BD 00                     Frame timestamp
0B      00 02                                       PktCnt = 2 (big-endian)

-- Packet 1 --
0D      53 50 30 30 30 30 30 30 30 30 31 00 00 00 00  SN = "SP0000000001"
1C      D3                                           RSSI = -45 (0xD3)
1D      00 00 01 8B C1 E4 BD 00                     Packet timestamp
25      00 03                                        DataLen = 3 (big-endian)
27      48 65 6C                                     "Hel"

-- Packet 2 --
2A      45 43 47 30 30 30 30 30 30 30 32 00 00 00 00  SN = "ECG000000002"
39      C4                                           RSSI = -60 (0xC4)
3A      00 00 01 8B C1 E4 BD 64                     Packet timestamp
42      00 02                                        DataLen = 2 (big-endian)
44      6C 6F                                        "lo"

-- Checksum --
46      XX XX                                        Checksum over bytes 0x00..0x45
```

Total length:

```text
13 + (26 + 3) + (26 + 2) + 2 = 72 bytes
```

### 3.6 Transmission Mechanism

| Trigger | Condition | Description |
|---|---|---|
| Periodic flush | Cache timeout | Gateway aggregates and uploads after cache timeout. Default is 5000 ms. |
| Immediate send | Cache timeout set to 0 | No aggregation; each packet is sent immediately. |
| Heartbeat | Every 3 seconds | Gateway sends a heartbeat frame with `PktCnt=0`. See Section 4. |

### 3.7 Protocol Routing

The same aggregated frame may be sent to all enabled UDP, TCP, and MQTT channels. The cloud may therefore receive duplicate copies over multiple channels. Deduplicate using `(GW_ID, Frame_TS)`. See Section 9.2.

The gateway uses `net_mode` to select the actual network channel:

| net_mode | Meaning |
|---:|---|
| 0 | NONE (do not send) |
| 1 | WiFi |
| 2 | 4G, China Mobile ML307R |
| 3 | 4G, Quectel EC800 |

---

## 4. Gateway Heartbeat Frame

The gateway sends a heartbeat frame every **3 seconds** for liveness detection and online status monitoring.

The heartbeat frame reuses the aggregated frame format and is identified by `PktCnt = 0`.

### 4.1 Frame Format

```text
0xAA | GW_ID[2] | Timestamp[8] | 0x0000 | SN[15] | Checksum[2]
```

| Offset | Length | Field | Description |
|---:|---:|---|---|
| 0 | 1 | 0xAA | Frame header |
| 1 | 2 | GW_ID | Gateway ID, big-endian |
| 3 | 8 | Timestamp | Current UTC millisecond timestamp, big-endian |
| 11 | 2 | PktCnt | Fixed `0x0000`, identifies heartbeat |
| 13 | 15 | Gateway SN | Gateway serial number, ASCII, right-padded with `0x00` |
| 28 | 2 | Checksum | 16-bit checksum, see Section 5 |

The heartbeat frame is fixed **30 bytes**.

### 4.2 Cloud Identification Logic

```text
PktCnt == 0  -> Heartbeat frame
PktCnt >= 1  -> Aggregated data frame
```

Recommendation: if the cloud does not receive a heartbeat for more than **10 seconds** (3 heartbeat periods), mark the gateway as offline and trigger an alarm.

---

## 5. Checksum Algorithm

The checksum is a simple **16-bit sum**.

The checksum range starts at frame header `0xAA` and ends immediately before the Checksum field. The Checksum field itself is not included.

### Java Example

```java
public static int calcChecksum(byte[] data, int offset, int len) {
    int sum = 0;
    for (int i = 0; i < len; i++) {
        sum += (data[offset + i] & 0xFF);
    }
    return sum & 0xFFFF;
}

int expected = ((data[data.length - 2] & 0xFF) << 8)
             | (data[data.length - 1] & 0xFF);
int actual = calcChecksum(data, 0, data.length - 2);
boolean valid = (expected == actual);
```

---

## 6. Cloud Downlink Data Protocol

The cloud sends data over **UDP or TCP** to the gateway. The gateway parses the frame, routes it to the correct handler, and forwards passthrough data to a BLE sensor.

### 6.1 Downlink Frame Format

The current downlink frame uses the same binary framing and checksum as the uplink. It contains fields such as BLE MAC address, gateway ID, and command data.

Currently supported downlink command:

```text
CMD_WIFI_FIRST_TIME (0x16) - Cloud requests the gateway to upload device information.
```

Additional downlink commands may be added in future protocol versions.

---

## 7. MQTT Configuration

### 7.1 MQTT Parameters

| Parameter | Default | Maximum Length | Description |
|---|---|---:|---|
| Broker address | - | 32 | Configured on the gateway |
| Broker port | - | - | Configured on the gateway |
| Username | - | 16 | Optional |
| Password | - | 16 | Optional |
| Client ID | - | 16 | Configured on the gateway |
| Publish topic | `SeenNext-GW` | 15 | Gateway publishes uplink data to this topic |
| Subscribe topic | - | - | Gateway subscribes to the same topic or a command topic |

### 7.2 Published Payload Format

When the gateway publishes over MQTT, the binary `0xAA` frame is converted to an **uppercase hexadecimal string** and used as the MQTT payload.

Example:

```text
# Original binary frame (same format as UDP/TCP):
AA 00 01 00 00 01 8B C1 E4 BD 00 00 02 ...

# MQTT payload published by the gateway:
AA00010000018BC1E4BD000002...
```

The cloud must first convert the MQTT HEX payload back to binary, then parse it as a normal gateway frame.

```python
payload = msg.payload  # bytes, e.g. b"AA00010000018BC1E4BD000002..."
raw = bytes.fromhex(payload.decode("ascii"))
frame = parse_gateway_frame(raw)
```

> Important: MQTT payload is a HEX string, while UDP/TCP packets are raw binary. UDP/TCP data can be parsed directly. MQTT data must be hex-decoded first.

---

## 8. Server Parsing Examples

The parsing functions below expect **binary frames**.

For MQTT, first convert the hex string payload to binary:

```python
raw = bytes.fromhex(payload.decode("ascii"))
```

### 8.1 Java Parser

```java
public class GatewayFrameParser {

    public static class SensorPacket {
        public String serialNumber;
        public int rssi;
        public long timestamp;
        public byte[] data;
    }

    public static class AggregatedFrame {
        public int gatewayId;
        public long frameTimestamp;
        public List<SensorPacket> packets = new ArrayList<>();
        public boolean isHeartbeat;
    }

    public static AggregatedFrame parse(byte[] raw) {
        if (raw == null || raw.length < 15) return null;
        if ((raw[0] & 0xFF) != 0xAA) return null;

        int expectedCs = ((raw[raw.length - 2] & 0xFF) << 8)
                       | (raw[raw.length - 1] & 0xFF);
        int actualCs = calcChecksum(raw, 0, raw.length - 2);
        if (expectedCs != actualCs) return null;

        AggregatedFrame frame = new AggregatedFrame();
        frame.gatewayId = ((raw[1] & 0xFF) << 8) | (raw[2] & 0xFF);
        frame.frameTimestamp = readBigEndianUint64(raw, 3);
        int pktCnt = ((raw[11] & 0xFF) << 8) | (raw[12] & 0xFF);
        frame.isHeartbeat = (pktCnt == 0);

        int offset = 13;
        for (int i = 0; i < pktCnt; i++) {
            SensorPacket pkt = new SensorPacket();
            pkt.serialNumber = new String(raw, offset, 15,
                    java.nio.charset.StandardCharsets.US_ASCII).trim();
            pkt.rssi = raw[offset + 15];
            pkt.timestamp = readBigEndianUint64(raw, offset + 16);
            int dataLen = ((raw[offset + 24] & 0xFF) << 8)
                        | (raw[offset + 25] & 0xFF);
            pkt.data = Arrays.copyOfRange(raw, offset + 26, offset + 26 + dataLen);
            frame.packets.add(pkt);
            offset += 26 + dataLen;
        }

        return frame;
    }

    private static long readBigEndianUint64(byte[] buf, int off) {
        long value = 0;
        for (int i = 0; i < 8; i++) {
            value = (value << 8) | (buf[off + i] & 0xFF);
        }
        return value;
    }

    private static int calcChecksum(byte[] data, int off, int len) {
        int sum = 0;
        for (int i = 0; i < len; i++) {
            sum += (data[off + i] & 0xFF);
        }
        return sum & 0xFFFF;
    }
}
```

### 8.2 Python Parser

```python
import struct

def parse_gateway_frame(raw: bytes):
    if len(raw) < 15 or raw[0] != 0xAA:
        return None

    expected_cs = (raw[-2] << 8) | raw[-1]
    actual_cs = sum(raw[:-2]) & 0xFFFF
    if expected_cs != actual_cs:
        return None

    gw_id = (raw[1] << 8) | raw[2]
    frame_ts = struct.unpack('>Q', raw[3:11])[0]
    pkt_cnt = (raw[11] << 8) | raw[12]

    result = {
        'gw_id': gw_id,
        'frame_ts': frame_ts,
        'pkt_cnt': pkt_cnt,
        'is_heartbeat': pkt_cnt == 0,
        'packets': [],
    }

    offset = 13
    for _ in range(pkt_cnt):
        sn = raw[offset:offset + 15].decode('ascii').rstrip('\x00')
        rssi = raw[offset + 15]
        if rssi >= 128:
            rssi -= 256
        pkt_ts = struct.unpack('>Q', raw[offset + 16:offset + 24])[0]
        data_len = (raw[offset + 24] << 8) | raw[offset + 25]
        data = raw[offset + 26:offset + 26 + data_len]

        result['packets'].append({
            'sn': sn,
            'rssi': rssi,
            'timestamp': pkt_ts,
            'data': data,
        })
        offset += 26 + data_len

    return result
```

### 8.3 C Parser

```c
#include <stdint.h>
#include <string.h>

typedef struct {
    char sn[16];
    int8_t rssi;
    uint64_t timestamp;
    uint16_t data_len;
    uint8_t data[512];
} sensor_packet_t;

typedef struct {
    uint16_t gw_id;
    uint64_t frame_ts;
    uint16_t pkt_cnt;
    int is_heartbeat;
    sensor_packet_t packets[64];
} aggregated_frame_t;

int parse_gateway_frame(const uint8_t *buf, uint16_t len,
                        aggregated_frame_t *out) {
    if (buf == NULL || len < 15 || buf[0] != 0xAA) return -1;

    uint16_t expected = ((uint16_t)buf[len - 2] << 8) | buf[len - 1];
    uint16_t actual = 0;
    for (uint16_t i = 0; i < len - 2; i++) {
        actual += buf[i];
    }
    if (expected != actual) return -2;

    out->gw_id = ((uint16_t)buf[1] << 8) | buf[2];
    out->frame_ts =
        ((uint64_t)buf[3] << 56) | ((uint64_t)buf[4] << 48) |
        ((uint64_t)buf[5] << 40) | ((uint64_t)buf[6] << 32) |
        ((uint64_t)buf[7] << 24) | ((uint64_t)buf[8] << 16) |
        ((uint64_t)buf[9] << 8)  | buf[10];
    out->pkt_cnt = ((uint16_t)buf[11] << 8) | buf[12];
    out->is_heartbeat = (out->pkt_cnt == 0);

    uint16_t offset = 13;
    for (uint16_t i = 0; i < out->pkt_cnt; i++) {
        memcpy(out->packets[i].sn, &buf[offset], 15);
        out->packets[i].sn[15] = '\0';
        out->packets[i].rssi = (int8_t)buf[offset + 15];
        out->packets[i].timestamp =
            ((uint64_t)buf[offset + 16] << 56) |
            ((uint64_t)buf[offset + 17] << 48) |
            ((uint64_t)buf[offset + 18] << 40) |
            ((uint64_t)buf[offset + 19] << 32) |
            ((uint64_t)buf[offset + 20] << 24) |
            ((uint64_t)buf[offset + 21] << 16) |
            ((uint64_t)buf[offset + 22] << 8)  |
            buf[offset + 23];
        out->packets[i].data_len =
            ((uint16_t)buf[offset + 24] << 8) | buf[offset + 25];
        memcpy(out->packets[i].data, &buf[offset + 26],
               out->packets[i].data_len);
        offset += 26 + out->packets[i].data_len;
    }

    return 0;
}
```

---

## 9. Integration Guide

### 9.1 Cloud Server Integration

1. Start a UDP server on the configured port and receive binary frames.
2. Start a TCP server, accept gateway connections, and receive binary frames.
3. Connect to the MQTT broker and subscribe to the gateway publish topic (default `SeenNext-GW`).
4. For MQTT, convert the HEX payload to binary before parsing.
5. Use the parsers in Section 8 to extract gateway ID, timestamp, and sensor data.
6. Monitor heartbeats. If no heartbeat is received for more than 10 seconds, mark the gateway offline.
7. Send downlink commands over UDP/TCP to the gateway address.

> UDP and TCP carry the same binary frame. If both protocols are enabled, the gateway sends the same frame over both. The cloud should select one and discard the other using `(GW_ID, Frame_TS)` deduplication.

### 9.2 Frame Deduplication

Use `(GW_ID, Frame_TS)` as a deduplication key.

```java
Set<String> seenFrames = new HashSet<>();

String dedupKey = frame.gatewayId + "_" + frame.frameTimestamp;
if (seenFrames.contains(dedupKey)) {
    // Duplicate frame. Discard it.
    return;
}

seenFrames.add(dedupKey);

// Periodically remove expired keys, e.g. keys older than 60 seconds.
```

---

WIFI4GGateWayBLE Cloud Protocol Documentation - v1.0
