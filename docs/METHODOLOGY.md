# Technical Methodology

Detailed technical documentation for the VoIP traffic interception and call reconstruction project.

---

## 1. Environment Setup

### 1.1 VoIP Server Deployment

The VoIP call server was deployed using **MagnusBilling**, an Asterisk-based PBX billing platform, providing:

- **SIP Registrar** on port `5060/UDP` for client registration and call signaling
- **RTP Media Relay** on ports `4010-4012` for voice data transport
- **SIP User Account**: `demo` (password: `demo12`)
- **Supported Codecs**: G.729, G.723, GSM, G.711A (alaw), G.711U (ulaw)
- **Trunk**: `India_tom` configured for outbound routing

The MicroSIP softphone was registered as the SIP client, connecting to the server and initiating test calls to controlled destination numbers.

### 1.2 Capture Station

**Hardware:**
- Alfa AWUS036ACH USB Wi-Fi adapter
- Chipset: Realtek RTL8812AU (802.11ac dual-band)
- Dual 5dBi RP-SMA antennas
- Supports monitor mode and packet injection

**Driver Installation:**
1. Downloaded RTL8812AU drivers from the Realtek website
2. Installed Npcap (v1.60+) as the packet capture library
3. Verified adapter appears as `Wi-Fi 2` in Wireshark interface list

**Why an External Adapter?**
```
Built-in NIC (Managed Mode):
  - Only captures frames addressed to THIS machine
  - Cannot see traffic between other devices
  - Limited to associated network traffic

Alfa Adapter (Monitor Mode):
  - Captures ALL 802.11 frames on the channel
  - Passive interception of any device's traffic
  - No association with the AP required
  - Combined with WPA2-PSK key = full decryption
```

## 2. Packet Capture Process

### 2.1 Wireshark Configuration

1. Selected `Wi-Fi 2` (Alfa adapter) as the capture interface
2. Enabled monitor mode in Wireshark capture options
3. Configured WPA2 decryption keys:
   - `Edit > Preferences > Protocols > IEEE 802.11 > Decryption Keys`
   - Added the WPA2-PSK passphrase for the target network
4. Started live capture during active VoIP calls

### 2.2 Traffic Observed

The capture contained mixed protocol traffic:

| Protocol | Count | Purpose |
|----------|-------|---------|
| SIP | ~14 packets/call | Call signaling |
| SIP/SDP | ~4 packets/call | Codec/media negotiation |
| RTP | ~774-2474 packets/stream | Voice data |
| TCP | Background | HTTP, TLS connections |
| UDP | Background | DNS, general traffic |
| NBNS | Background | NetBIOS name resolution |
| TLSv1.3 | Background | Encrypted web traffic |

## 3. SIP Signaling Analysis

### 3.1 Call Flow

```
Client (MicroSIP)              Server (MagnusBilling/Asterisk)
      |                                    |
      |--- INVITE sip:91XXXXXXXXXX@... --->|  (Call initiation + SDP offer)
      |<-- 401 Unauthorized ---------------|  (Digest auth challenge)
      |--- ACK ---------------------------->|  (Acknowledge 401)
      |--- INVITE (with credentials) ----->|  (Re-INVITE with auth)
      |<-- 100 Trying ---------------------|  (Provisional response)
      |<-- 200 OK (SDP answer) ------------|  (Call accepted, media params)
      |--- ACK ---------------------------->|  (Acknowledge 200)
      |                                    |
      |<========== RTP Media =============>|  (Bidirectional voice data)
      |                                    |
      |--- BYE ---------------------------->|  (Call termination)
      |<-- 200 OK -------------------------|  (Acknowledge termination)
```

### 3.2 Extracted Metadata

From the captured SIP headers:

- **Caller URI**: `sip:demo@195.35.6.83`
- **Server IP**: `192.168.1.5` (local) / `195.35.6.83` (SIP domain)
- **Client IP**: `192.168.1.6`
- **SIP Port**: `5060/UDP`
- **RTP Ports**: `4008-4012` (server-side), `10936-19760` (client-side)
- **Codec Negotiated**: G.711A (PCMA), 8000 Hz

## 4. RTP Stream Analysis

### 4.1 Detected Streams

6 RTP streams were identified across 2 calls:

| # | Source | Src Port | Destination | Dst Port | Codec | Packets | Duration | Loss |
|---|--------|----------|-------------|----------|-------|---------|----------|------|
| 1 | 192.168.1.5 | 4012 | 195.35.6.83 | 19760 | G.711A | 774 | 15.46s | 0.0% |
| 2 | 192.168.1.5 | 4010 | 195.35.6.83 | 10936 | G.711A | 2474 | 49.49s | 0.0% |
| 3 | 195.35.6.83 | 19760 | 192.168.1.5 | 4012 | G.711A | 669 | 13.36s | 0.0% |
| 4 | 195.35.6.83 | 19760 | 192.168.1.5 | 4012 | G.711A | 98 | 1.99s | 0.0% |
| 5 | 195.35.6.83 | 10936 | 192.168.1.5 | 4010 | G.711A | 2194 | 43.87s | 0.0% |
| 6 | 195.35.6.83 | 10936 | 192.168.1.5 | 4010 | G.711A | 272 | 5.45s | 0.0% |

### 4.2 Stream Quality Metrics (Stream 0)

- **SSRC**: `0x75f66e1a`
- **Total Packets**: 1413 (0 lost, 0 sequence errors)
- **Mean Delta**: 20.002 ms (expected for G.711 at 50 pps)
- **Max Delta**: 30.918 ms
- **Mean Jitter**: 0.932 ms
- **Max Jitter**: 2.233 ms
- **Max Skew**: -11.198 ms
- **Clock Drift**: -0 ms
- **Frequency Drift**: 7999 Hz (-0.00%)

### 4.3 Jitter Analysis

The RTP stream analysis graph shows:
- **Delta** (inter-packet arrival): Stable around 20ms baseline with occasional spikes to 30ms
- **Jitter**: Predominantly < 5ms, consistent with a LAN environment
- **Skew**: Minor clock drift, within acceptable bounds for real-time playback

## 5. Call Reconstruction

### 5.1 Wireshark Tools Used

1. **`Telephony > RTP > RTP Streams`**: Enumerated all detected RTP streams with metadata
2. **`Telephony > RTP > RTP Stream Analysis`**: Per-packet timing analysis with jitter/delta graphs
3. **`Telephony > RTP > RTP Player`**: Decoded G.711A payloads and rendered audio waveforms
4. **`Telephony > VoIP Calls`**: Correlated SIP signaling with RTP streams for full call view

### 5.2 Playback Configuration

- **Output**: SteelSeries Sonar Virtual Audio Device
- **Jitter Buffer**: 50ms (fixed)
- **Playback Timing**: Jitter Buffer mode
- **Min Silence**: 2s
- **Output Audio Rate**: Automatic

### 5.3 Results

Both intercepted calls were successfully reconstructed with full bidirectional audio:

| Call | Duration | From | To | State |
|------|----------|------|----|-------|
| 1 | 00:00:52 | `sip:demo@195.35.6.83` | `sip:91XXXXXXXXXX@195.35.6.83` | COMPLETED |
| 2 | 00:00:18 | `sip:demo@195.35.6.83` | `sip:91XXXXXXXXXX@195.35.6.83` | COMPLETED |

## 6. Wireshark Display Filters Used

```
sip                           # All SIP signaling
rtp                           # All RTP streams
sip.Method == "INVITE"        # Call initiations only
sip.Status-Code == 200        # Successful responses
rtp.ssrc == 0x75f66e1a        # Specific RTP stream
ip.addr == 192.168.1.5        # Server traffic
udp.port == 5060              # SIP signaling port
```
