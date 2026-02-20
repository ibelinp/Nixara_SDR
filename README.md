# FPGA SDR — VRT Protocol Specification & Recommendations

## 1. Protocol Overview

VITA 49.0 (VRT) over UDP for all IQ data and context metadata. No TCP control channel initially — control is a future addition using the same VRT command packet infrastructure over unicast UDP.

**Design goals:**
- Minimal VRT header (20 bytes) — no Class ID, no Trailer
- VITA 49 compliant for market acceptance and tooling (Wireshark, GNU Radio)
- 500 MHz ADC clock as master timestamp reference
- Multi-radio phased array synchronization via shared PPS + clock
- Three sample formats: 8-bit, 16-bit, 32-bit signed integer IQ (sc8/sc16/sc32)
- Clean operation at both 1500-byte and jumbo MTUs
- Simple, deterministic FPGA implementation

**What we use from VRT:**
- IF Data Packet (type `0001`) with Stream ID + timestamps
- Context Packet (type `0100`) with CIF0 bitmask
- 32-bit word alignment throughout

**What we skip:**
- Class ID (C=0) — saves 8 bytes, device type is known from context packet
- Trailer (T=0) — saves 4 bytes, no CRC needed on a LAN
- Extension Context (CIF1+) — CIF0 fields are sufficient

---

## 2. VRT IF Data Packet Format

Total header: **20 bytes** (5 × 32-bit words)

```
Word 0 — VRT Header (4 bytes)
┌────────┬───┬───┬────┬────┬─────┬─────┬──────────┬────────────────┐
│ PktType│ C │ T │Ind │Ind │ TSI │ TSF │ PktCount │  Packet Size   │
│ [31:28]│27 │26 │25  │24  │23:22│21:20│ [19:16]  │    [15:0]      │
│  0001  │ 0 │ 0 │ 0  │ 0  │ 01  │ 01  │ mod-16   │  32-bit words  │
└────────┴───┴───┴────┴────┴─────┴─────┴──────────┴────────────────┘

Word 1 — Stream ID (4 bytes)
┌────────────────────────────────────────────────────────────────────┐
│                        Stream Identifier                          │
│           [31:16] Device ID  |  [15:0] Channel Number             │
└────────────────────────────────────────────────────────────────────┘

Word 2 — Integer Timestamp (4 bytes)
┌────────────────────────────────────────────────────────────────────┐
│                     UTC Seconds (from PPS)                        │
│                  TSI=01: POSIX time (epoch)                       │
└────────────────────────────────────────────────────────────────────┘

Words 3-4 — Fractional Timestamp (8 bytes)
┌────────────────────────────────────────────────────────────────────┐
│              64-bit Sample Count since last PPS                   │
│          TSF=01: counts at 500 MHz → 0 to 499,999,999            │
│            (resets each second on PPS edge)                       │
└────────────────────────────────────────────────────────────────────┘

Words 5+ — IQ Payload
┌────────────────────────────────────────────────────────────────────┐
│                     IQ Sample Data                                │
│         Format defined by Context Packet (see Section 4)          │
└────────────────────────────────────────────────────────────────────┘
```

### Fixed header bits for all IQ packets

| Field | Bits | Value | Meaning |
|-------|------|-------|---------|
| Packet Type | [31:28] | `0001` | IF Data with Stream ID |
| C (Class ID) | [27] | `0` | No Class ID present |
| T (Trailer) | [26] | `0` | No Trailer present |
| TSI | [23:22] | `01` | Integer timestamp = UTC |
| TSF | [21:20] | `01` | Fractional timestamp = sample count |

The FPGA only needs to update three fields per packet: **Packet Count** (mod-16 counter), **Packet Size** (constant for a given MTU/format), and **timestamps**.

---

## 3. VRT Context Packet Format

Context packets share the same 20-byte prologue, followed by a CIF0 bitmask and variable-length fields.

```
Word 0 — VRT Header (4 bytes)
│ PktType = 0100 │ C=0 │ T=0 │ TSI=01 │ TSF=01 │ PktCount │ Size │

Word 1 — Stream ID (4 bytes)
│ Same Device ID : Channel as the IQ stream it describes            │

Words 2-4 — Timestamps (12 bytes)
│ Same format as IQ packets — marks when this context is valid      │

Word 5 — Context Indicator Field 0 (CIF0) (4 bytes)
┌────────────────────────────────────────────────────────────────────┐
│  Bit 31: CHANGE INDICATOR    │  0 = periodic heartbeat            │
│         (no payload bytes)   │  1 = PARAMETER CHANGED — receivers │
│                              │      must reconfigure              │
│  Bit 27: Bandwidth           │  Bit 25: RF Reference Freq        │
│  Bit 21: Sample Rate         │  Bit 14: Gain                     │
│  Bit 9:  Over-range Count    │  Bit 7:  Reference Level          │
│  Bit 1:  Data Packet Payload Format (critical — sample format)   │
└────────────────────────────────────────────────────────────────────┘

Words 6+ — Context Fields (variable, only fields with CIF0 bit set)
│ Fields appear in bit order (31 down to 0), each at defined size   │
```

### Recommended CIF0 fields for initial implementation

| CIF0 Bit | Field | Size | Description |
|----------|-------|------|-------------|
| 31 | Change Indicator | 0 bytes (flag only) | **0 = periodic (steady state), 1 = parameter changed — receivers must reconfigure** |
| 27 | Bandwidth | 8 bytes | Analog bandwidth in Hz (64-bit) |
| 25 | RF Reference Frequency | 8 bytes | Center frequency in Hz (64-bit) |
| 21 | Sample Rate | 8 bytes | Output sample rate in Hz (64-bit) |
| 14 | Gain | 4 bytes | Stage 1 (back-end) + Stage 2 (front-end), 16-bit each, dB×128 |
| 9 | Over-range Count | 4 bytes | ADC clipping event count since last context |
| 7 | Reference Level | 4 bytes | Expected max signal level, dBm×128 |
| 1 | Data Packet Payload Format | 8 bytes | **Sample format descriptor (see Section 4)** |

### Context packet transmission rules

Context packets are sent on a **time-based interval**, not a packet-count interval. This keeps the context rate predictable regardless of sample rate, MTU, or sample format.

1. **Periodic**: Send every **100–200 ms** (configurable, default 100 ms). This guarantees a new client receives full stream state within 200 ms of joining.
2. **On-change**: Send immediately when any parameter changes (frequency retune, gain change, sample rate change). The Change Indicator (CIF0 bit 31) is set to **1** to warn downstream receivers that parameters have changed and they must reconfigure.
3. **Periodic packets** (Change Indicator = 0) include **ALL fields** — full state snapshot for late-joining clients.
4. **On-change packets** (Change Indicator = 1) also include **ALL fields** — the receiver may need any field to reconfigure. The Change Indicator flag is the signal that something changed; the full field set tells the receiver the new state without requiring it to diff against previous context.

### Receiver startup sequence

A receiver joining a stream follows this sequence:

```
1. Open UDP socket, join multicast group (or bind unicast port)
2. Wait for first Context Packet (max 200 ms at default interval)
3. Read context fields:
   - Data Packet Payload Format → configure sample parser (8/16/32-bit)
   - RF Reference Frequency     → set display center frequency
   - Sample Rate                → set display span, configure FFT
   - Bandwidth                  → set analog bandwidth display
   - Gain, Reference Level      → set Y-axis / color scaling
4. Begin processing IQ Data Packets
5. On receiving context with Change Indicator = 1:
   - Reconfigure display/processing with new parameters
   - Optionally flush/reset averaging, waterfall history, etc.
```

IQ packets received **before** the first context packet are discarded — the receiver does not know the sample format or RF parameters yet. This is safe because at most 200 ms of data is lost on initial connect.

### FPGA context timing implementation

The FPGA uses a simple timer rather than counting IQ packets:

```verilog
// Timer-based context packet insertion
reg [23:0] context_timer;       // counts clock cycles
reg        param_changed;
reg        context_due;

localparam CONTEXT_INTERVAL_CLKS = 24'd15_625_000;  // 100 ms at 156.25 MHz

always @(posedge clk) begin
    if (context_timer >= CONTEXT_INTERVAL_CLKS) begin
        context_due <= 1;
        context_timer <= 0;
    end else begin
        context_timer <= context_timer + 1;
    end

    if (freq_changed || gain_changed || rate_changed) begin
        param_changed <= 1;       // latch change flag
    end

    // Insert context at next IQ packet boundary
    if (iq_packet_sent && (context_due || param_changed)) begin
        insert_context_packet();  // always sends full state
        context_due <= 0;
        param_changed <= 0;
    end
end
```

Context packets are inserted **between** IQ packets (at packet boundaries), never interrupting an IQ packet in progress. The Change Indicator bit in CIF0 is set to `param_changed` at the time of insertion.

---

## 4. Sample Formats

Three supported formats, identified by the VRT **Data Packet Payload Format** field (CIF0 bit 1):

| Format | I size | Q size | Bytes per sample | VRT Packing | Use case |
|--------|--------|--------|-----------------|-------------|----------|
| 8-bit integer (sc8) | 8-bit signed | 8-bit signed | 2 | 2 samples per 32-bit word | Wideband survey, scanning |
| 16-bit integer (sc16) | 16-bit signed | 16-bit signed | 4 | 1 sample per 32-bit word | Primary — best throughput/dynamic range balance |
| 32-bit integer (sc32) | 32-bit signed | 32-bit signed | 8 | 2 words per sample | Full precision after DDC/decimation |

All three formats are signed fixed-point integers. No floating-point conversion is performed in the FPGA — the sc32 format directly carries the full-width accumulator output from the decimation/DDC chain. Host-side drivers (SoapySDR, GNU Radio OOT module) convert to fc32 (float complex) if the application requests it. This is standard practice (matches Ettus UHD sc16→fc32 path) and avoids wasting FPGA resources on IEEE 754 conversion that Artix-7 has no hardware support for.

### VRT Data Packet Payload Format Field (8 bytes)

```
Word 0:
┌──────────┬──────────┬────────┬──────────┬──────────┬──────────────┐
│ Packing  │ RealCplx │ DataFmt│ Rpt Ind  │ Event Sz │ Channel Sz   │
│ [31:30]  │ [29:28]  │ [27:24]│ [23]     │ [22:20]  │ [19:0]       │
└──────────┴──────────┴────────┴──────────┴──────────┴──────────────┘

Word 1:
┌──────────────────┬───────────────────┬────────────────────────────┐
│ Data Item Size   │ Repeat Count      │ Vector Size                │
│ [31:26]          │ [25:16]           │ [15:0]                     │
└──────────────────┴───────────────────┴────────────────────────────┘
```

| Format | RealCplx | DataFmt | Data Item Size | Packing |
|--------|----------|---------|---------------|---------|
| sc8 (8-bit int IQ) | 01 (complex) | 0000 (signed fixed) | 8 | 01 (link-efficient) |
| sc16 (16-bit int IQ) | 01 (complex) | 0000 (signed fixed) | 16 | 01 (link-efficient) |
| sc32 (32-bit int IQ) | 01 (complex) | 0000 (signed fixed) | 32 | 01 (link-efficient) |

### 32-bit word packing for each format

```
8-bit IQ (2 samples per word, link-efficient packing):
┌─────────┬─────────┬─────────┬─────────┐
│  I[0]   │  Q[0]   │  I[1]   │  Q[1]   │
│  8-bit  │  8-bit  │  8-bit  │  8-bit  │
└─────────┴─────────┴─────────┴─────────┘

16-bit IQ (1 sample per word):
┌───────────────────┬───────────────────┐
│      I[n]         │      Q[n]         │
│     16-bit        │     16-bit        │
└───────────────────┴───────────────────┘

32-bit integer IQ (2 words per sample):
┌───────────────────────────────────────┐
│              I[n] (int32)             │
├───────────────────────────────────────┤
│              Q[n] (int32)             │
└───────────────────────────────────────┘
```

---

## 5. MTU & Packet Sizing

### Standard Ethernet (MTU 1500)

```
Ethernet frame:   1518 bytes (incl. CRC)
  Ethernet header:  14 bytes
  IP header:        20 bytes
  UDP header:        8 bytes
  ─────────────────────────
  UDP payload max: 1472 bytes  →  VRT packet max
  VRT header:       20 bytes
  IQ payload max:  1452 bytes
```

| Format | Samples/packet | Payload bytes | VRT packet (bytes) | VRT packet (words) |
|--------|---------------|--------------|-------------------|-------------------|
| sc8 | 726 | 1452 | 1472 | 368 |
| sc16 | 363 | 1452 | 1472 | 368 |
| sc32 | 181 | 1448 | 1468 | 367 |

### Jumbo Frames — 10GbE target (MTU ~9000)

Target VRT packet size: **8000 bytes** (2000 words) — clean number, safe margin below 9000-byte jumbo limit.

```
  UDP payload:     8000 bytes  →  VRT packet
  VRT header:        20 bytes
  IQ payload:      7980 bytes
```

| Format | Samples/packet | Payload bytes | VRT packet (bytes) | VRT packet (words) |
|--------|---------------|--------------|-------------------|-------------------|
| sc8 | 3990 | 7980 | 8000 | 2000 |
| sc16 | 1995 | 7980 | 8000 | 2000 |
| sc32 | 997 | 7976 | 7996 | 1999 |

### Packet rate at various sample rates (16-bit IQ)

| Sample rate | Throughput | Pkts/sec (1500 MTU) | Pkts/sec (jumbo) |
|------------|-----------|-------------------|-----------------|
| 2 MS/s | 8 MB/s | 5,510 | 1,003 |
| 10 MS/s | 40 MB/s | 27,548 | 5,013 |
| 50 MS/s | 200 MB/s | 137,741 | 25,063 |
| 100 MS/s | 400 MB/s | 275,482 | 50,125 |
| 200 MS/s | 800 MB/s | — | 100,251 |
| 500 MS/s | 2 GB/s | — | 250,627 |

> Above ~20 MS/s with 1500 MTU, packet rates exceed what a standard kernel UDP stack can handle reliably. Jumbo frames are required for high sample rates. At 200+ MS/s, DPDK or kernel-bypass on the host side becomes necessary.

---

## 6. Timestamp Design — 500 MHz ADC Clock

### Architecture

All radios in the phased array share:
1. **PPS** (1 pulse per second) — from GPS, master clock, or White Rabbit
2. **500 MHz reference clock** — distributed via clock fanout or synthesized from 10 MHz ref

### VRT timestamp fields

```
Integer Timestamp (TSI=01, UTC):
  32-bit POSIX seconds — set from GPS/NTP at boot, increments on PPS

Fractional Timestamp (TSF=01, Sample Count):
  64-bit counter — counts ADC samples at 500 MHz
  Resets to 0 on each PPS rising edge
  Range per second: 0 to 499,999,999
```

### Why sample count (TSF=01) and not picoseconds (TSF=10)

| | Sample Count (TSF=01) | Real-Time Picoseconds (TSF=10) |
|-|-----------------------|-------------------------------|
| FPGA logic | Simple 64-bit counter, +1 per ADC clock | Multiply: each tick = 2000 ps |
| Precision | Exact — no rounding | Exact at 500 MHz (2000 divides evenly) |
| Multi-rate | Counter rate = ADC clock, independent of decimation | Same |
| Phased array sync | Compare sample counts directly | Must divide by 2000 to get sample offset |
| GNU Radio | Sample count is native — aligns with stream tags | Must convert |
| **Recommendation** | **Use this** | Adds unnecessary multiplication |

### Decimation and timestamps

When the FPGA decimates (e.g., 500 MHz ADC → 50 MS/s output), the timestamp in each IQ packet still references the **ADC clock**:

```
Packet 1:  Integer=1740000000  Fractional=0          (first sample at PPS)
Packet 2:  Integer=1740000000  Fractional=19950      (1995 output samples later, ×10 decimation = 19950 ADC ticks)
Packet 3:  Integer=1740000000  Fractional=39900
...
```

The context packet's **Sample Rate** field tells the receiver the output rate (50 MS/s). The receiver calculates:

```
ADC ticks between output samples = Fractional[n+1] - Fractional[n] / samples_per_packet
```

This preserves sub-sample timing information through the decimation chain.

---

## 7. Multi-Radio Phased Array Architecture

### Network topology

```
                    ┌──────────────┐
                    │  PPS + 10MHz │
                    │  Distribution│
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
     │  SDR 0  │     │  SDR 1  │     │  SDR 2  │
     │ Dev 0x00│     │ Dev 0x01│     │ Dev 0x02│
     └────┬────┘     └────┬────┘     └────┬────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                    ┌──────┴───────┐
                    │   10GbE      │
                    │   Switch     │
                    │  (IGMP)      │
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
     │ Display │     │ Recorder│     │ DSP     │
     │  Host   │     │  Host   │     │ Cluster │
     └─────────┘     └─────────┘     └─────────┘
```

### Multicast addressing scheme

| Stream | Multicast Group | Port | Content |
|--------|----------------|------|---------|
| SDR 0, Ch 0 | 239.1.0.0 | 50001 | IQ + context |
| SDR 0, Ch 1 | 239.1.0.1 | 50001 | IQ + context |
| SDR 1, Ch 0 | 239.1.1.0 | 50001 | IQ + context |
| SDR 1, Ch 1 | 239.1.1.1 | 50001 | IQ + context |
| SDR N, Ch M | 239.1.N.M | 50001 | IQ + context |
| Control (all) | unicast | 50000 | Commands + ACKs |

**Encoding:** `239.1.<device_id>.<channel>` — up to 256 devices × 256 channels.

For larger arrays, use `239.1.0.0/16` range with 16-bit encoding: `239.1.<high_byte>.<low_byte>` where the 16-bit value = `(device_id << 4) | channel`, supporting 4096 devices × 16 channels.

### Unicast fallback

For simple single-radio setups, all streams go to a single unicast destination. Configured via control command or DIP switch / default.

### Synchronization requirements

| Parameter | Requirement | How achieved |
|-----------|------------|--------------|
| Time alignment | < 1 ADC sample (2 ns) | Shared PPS + 500 MHz clock |
| Frequency accuracy | < 1 ppb | Shared 10 MHz reference → PLL |
| Phase coherence | Stable across minutes | Matched cable lengths, common LO if possible |
| Timestamp agreement | Exact match at PPS | All counters reset simultaneously |

A phased array beamformer on the host reads Stream IDs to identify antennas, aligns data using fractional timestamps, and applies phase/delay weights per element.

---

## 8. Stream ID Convention

```
Stream ID (32 bits):
┌────────────────────────────┬──────────────────────────────┐
│   [31:16] Device ID        │   [15:0] Channel Number      │
│   Unique per SDR unit      │   0 = Ch A, 1 = Ch B, etc.   │
└────────────────────────────┴──────────────────────────────┘
```

- **Device ID**: Set per unit via control command, EEPROM, or DIP switches. Must be unique in the array.
- **Channel Number**: For multi-channel SDRs (dual ADC, quad ADC). Single-channel units always use 0.
- **Context packets** use the same Stream ID as the IQ stream they describe.
- A receiver filters on Stream ID to select which radio/channel to process.

---

## 9. Control Channel (Future — Phase 2)

Not needed for initial passive listener mode, but designed for future implementation.

### Protocol: Reliable UDP (not TCP)

```
Client → SDR:   Command packet (unicast to SDR IP:50000)
SDR → Client:    ACK packet (unicast back to client)
```

**Command packet format (VRT Extension Command Packet, type `0110`):**
```
VRT Header (4 bytes)  — PktType=0110
Stream ID (4 bytes)   — target device:channel
Sequence (4 bytes)    — for ACK matching and retransmit
Command ID (4 bytes)  — set_frequency, set_gain, set_sample_rate, etc.
Payload (variable)    — command arguments
```

**Reliability:** Client retransmits after 50 ms timeout, up to 3 retries. Simple for FPGA — no TCP state machine, just match sequence numbers.

### Suggested command set

| Command ID | Payload | Description |
|-----------|---------|-------------|
| 0x0001 | 8 bytes (Hz) | Set center frequency |
| 0x0002 | 8 bytes (Hz) | Set sample rate |
| 0x0003 | 4 bytes (dB×128) | Set gain |
| 0x0004 | 4 bytes | Set decimation factor |
| 0x0005 | 4 bytes (ms) | Set context packet interval (default 100 ms) |
| 0x0010 | 0 bytes | Start streaming |
| 0x0011 | 0 bytes | Stop streaming |
| 0x00F0 | 0 bytes | Query full status (triggers context packet) |

---

## 10. GNU Radio & Software Integration

### Integration paths (prioritized)

#### 1. SoapySDR driver (Recommended — highest market reach)

SoapySDR is the universal SDR abstraction layer. One driver enables:
- GNU Radio (via `gr-soapy` — included in GNU Radio 3.10+)
- CubicSDR
- SDR++
- SDRangel
- Gqrx
- Any SoapySDR-compatible application

**Implementation:** C++ shared library implementing the SoapySDR API. Receives VRT packets, parses headers, delivers samples. Sends control commands for Phase 2. Performs sc16/sc32 → fc32 (float complex) conversion on the host using SIMD (SSE/AVX `CVTDQ2PS` on x86, NEON `vcvtq_f32_s32` on ARM). This matches the standard UHD model — integer on the wire, float to the application.

#### 2. Native GNU Radio OOT module (`gr-yoursdr`)

For users wanting direct VRT access without the SoapySDR abstraction layer:
- `vrt_source` block — receives VRT IQ packets, outputs GNU Radio stream
- `vrt_context_source` block — extracts metadata as stream tags
- Timestamps map directly to GNU Radio's `rx_time` tag (integer seconds, fractional seconds)

#### 3. PySpectraVue `VRTRadio`

The existing pySpectraVue application gets a VRT-native radio backend:
- On connect: waits for first context packet, auto-configures display (center freq, span, sample rate, scale)
- Parses VRT headers for timestamp and stream identification
- On context with Change Indicator = 1: reconfigures display parameters, optionally resets waterfall history
- Supports multiple simultaneous streams (stream selector by Stream ID) for array visualization

### GNU Radio stream tag mapping

| VRT Field | GNU Radio Tag | Type | Notes |
|-----------|--------------|------|-------|
| Integer + Fractional Timestamp | `rx_time` | tuple(int, float) | Every IQ packet |
| RF Reference Frequency (context) | `rx_freq` | float | On first context + every change |
| Sample Rate (context) | `rx_rate` | float | On first context + every change |
| Bandwidth (context) | `rx_bandwidth` | float | On first context + every change |
| Gain (context) | `rx_gain` | float | On first context + every change |
| Stream ID | `rx_stream_id` | int | On first packet |
| Change Indicator (CIF0 bit 31) | `rx_param_change` | bool | **True = parameters changed, downstream must reconfigure** |

The `rx_param_change` tag is the key signal for downstream blocks. When a GNU Radio block sees this tag, it should re-read `rx_freq`, `rx_rate`, etc. and reconfigure its processing (retune DDC, resize FFT, update display). Blocks that don't care about parameter changes can ignore this tag.

---

## 11. FPGA Implementation Notes

### Packet construction pipeline

```
ADC Samples (500 MHz)
       │
       ▼
┌──────────────┐
│  Decimator   │  (programmable: ÷1 to ÷N)
│  + Timestamp │  (captures fractional ts at first sample of group)
│    Capture   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Packet      │  (counts samples, inserts header when full)
│  Framer      │  (muxes context packet on 100ms timer or param change)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  UDP/IP/ETH  │  (Alex Forencich verilog-ethernet stack)
│  Stack       │  (64-bit AXI-Stream @ 156.25 MHz for 10GbE)
└──────┬───────┘
       │
       ▼
    10GbE MAC
```

### VRT header as FPGA constants + registers

Most of the 20-byte VRT header is constant or slow-changing:

| Header field | Source | Update rate |
|-------------|--------|------------|
| Packet Type, C, T, TSI, TSF | **Constant** (hardwired) | Never |
| Packet Count | **4-bit counter** | Every packet |
| Packet Size | **Register** (set once per MTU/format config) | On config change |
| Stream ID | **Register** (device ID + channel) | On config change |
| Integer Timestamp | **Counter** (increments on PPS) | Every second |
| Fractional Timestamp | **Counter** (500 MHz, resets on PPS) | Captured per packet |

The packet framer only needs to:
1. Write 5 constant/register words (header)
2. Latch the timestamp counters at the first sample of each packet
3. Stream N sample words
4. Increment packet count

### Context packet insertion

Context insertion uses the timer-based approach described in Section 3. The framer checks `context_due || param_changed` after each IQ packet completes, and inserts a full-state context packet at the next packet boundary. See Section 3 for the Verilog pseudocode.

### AXI-Stream considerations for Alex Forencich stack

- 10GbE: 64-bit data path @ 156.25 MHz (64/66b encoding)
- VRT header is 20 bytes = 2.5 × 64-bit words — first beat is 8 bytes (words 0-1), second is 8 bytes (words 2-3), third is 4 bytes (word 4) + first 4 bytes of payload
- Use `tkeep` for the partial last beat of each packet
- `tlast` asserted on final beat of each VRT packet (maps to UDP packet boundary)
- The Forencich stack handles UDP/IP/Ethernet encapsulation — you feed it raw UDP payload (= VRT packet) on AXI-Stream

### Resource estimate (rough, Xilinx 7-series)

| Block | LUTs | FFs | BRAM |
|-------|------|-----|------|
| Timestamp counters | ~100 | ~128 | 0 |
| Packet framer | ~500 | ~300 | 1 (header template) |
| Context packet builder | ~800 | ~400 | 1 (CIF field storage) |
| Forencich UDP/IP/ETH | ~3000 | ~2000 | 4-8 |
| **Total** | **~4400** | **~2800** | **~6-10** |

Fits easily alongside ADC interface + decimation + DDC logic.

---

## 12. Recommended Implementation Phases

### Phase 1 — Passive Listener (PySpectraVue)

- FPGA: TX-only (streams VRT IQ + context packets)
- Host: `VRTListenerRadio` class in pySpectraVue receives and displays
- Fixed configuration via FPGA registers (no runtime control)
- Single stream, unicast, 16-bit IQ only
- 1500 MTU

### Phase 2 — Multi-format + Jumbo + Multicast

- Add sc8 and sc32 format support
- Jumbo frame support (8000-byte VRT packets)
- Multicast for multiple listeners
- Context packets with full CIF0 field set

### Phase 3 — Control Channel + SoapySDR

- UDP control commands (set freq, gain, rate, start/stop)
- SoapySDR driver for GNU Radio + ecosystem compatibility
- Multi-channel support (dual/quad ADC)

### Phase 4 — Phased Array Scale

- Multi-device synchronization validation
- GNU Radio OOT module with beamforming blocks
- High-rate host library (C++ with DPDK or kernel-bypass)
- Array calibration and phase coherence tools

---

## 13. Reference Documents

| Document | Relevance |
|----------|-----------|
| ANSI/VITA 49.0-2015 | VRT packet format specification |
| ANSI/VITA 49.2 | VRT context packet extensions |
| DIFI Consortium Spec | Simplified VRT profile for IF streaming (useful reference) |
| Alex Forencich verilog-ethernet | FPGA UDP/IP/Ethernet stack |
| SoapySDR API | Driver interface specification |
| GNU Radio Stream Tags | Metadata tagging for SDR source blocks |
| Per Vices PVAN-11 | Example commercial VRT implementation |

---

## Appendix A — Quick Reference: Byte Layout

### IQ Data Packet (16-bit, 1500 MTU, 363 samples)

```
Offset  Bytes  Field
──────  ─────  ─────────────────────────────
0       4      VRT Header [0x1404_xxxx]  (type=0001, TSI=01, TSF=01, size=368)
4       4      Stream ID
8       4      Integer Timestamp (UTC seconds)
12      8      Fractional Timestamp (sample count at 500 MHz)
20      1452   IQ Samples (363 × 4 bytes)
──────  ─────
Total:  1472 bytes = 368 32-bit words
```

### IQ Data Packet (16-bit, Jumbo, 1995 samples)

```
Offset  Bytes  Field
──────  ─────  ─────────────────────────────
0       4      VRT Header [0x1404_xxxx]  (type=0001, TSI=01, TSF=01, size=2000)
4       4      Stream ID
8       4      Integer Timestamp (UTC seconds)
12      8      Fractional Timestamp (sample count at 500 MHz)
20      7980   IQ Samples (1995 × 4 bytes)
──────  ─────
Total:  8000 bytes = 2000 32-bit words
```

### Context Packet (full state, typical)

```
Offset  Bytes  Field
──────  ─────  ─────────────────────────────
0       4      VRT Header [0x4404_xxxx]  (type=0100, TSI=01, TSF=01)
4       4      Stream ID
8       4      Integer Timestamp
12      8      Fractional Timestamp
20      4      CIF0 bitmask
24      8      RF Reference Frequency (Hz)
32      8      Sample Rate (Hz)
40      8      Bandwidth (Hz)
48      4      Gain (dB × 128)
52      4      Over-range Count
56      4      Reference Level (dBm × 128)
60      8      Data Packet Payload Format
──────  ─────
Total:  68 bytes = 17 32-bit words
```
