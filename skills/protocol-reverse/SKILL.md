---
name: protocol-reverse
description: Use for authorized reverse engineering of custom binary protocols, Protobuf/gRPC, WebSocket frames, and PCAP-driven protocol recovery.
---

# Protocol Reverse Engineering

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Read `../field-journal/precedent-reverse.md` — authorization context + routine operation boundaries
2. `NOW`: Confirm the task is **protocol / traffic / serialization-format** reverse (pure web param signing → `js-reverse/`)
3. `NOW`: If there is live network interaction with a target → `../scripts/case-init.ps1` to create scope.md (tracking + network profile; authorization per precedent-auth.md)
4. `NEXT`: Read `../tool-index.md`; bootstrap missing tools (tshark/wireshark may need manual install)
5. `ACT`: Enter workflow Phase 1 — produce a draft frame layout or message dictionary

## Use cases

- Custom TCP/UDP binary protocols
- Protobuf / gRPC / FlatBuffers / MessagePack
- WebSocket / MQTT / private RPC
- PCAP / PCAPNG field and state-machine recovery
- Client-server validation, sequence numbers, encrypted frame headers

## Not this skill

| Case | Route |
|------|-------|
| HTTP param signing / JS encryption only | `js-reverse/` |
| TLS certificate issues only | `pentest-tools/` or browser proxy |
| Firmware protocol stack deep-dive + emulation | `firmware-pentest/`, then back here |

## Workflow

### Phase 1 — Capture & triage

```text
□ Get a sample: PCAP / proxy export / client logs / binary
□ Mark direction: C→S / S→C; handshake? heartbeat? reconnect?
□ Fixed header? Magic bytes? Length field? TLV? Fixed-length?
□ Compression (zlib/gzip/lz4) or encryption (AES/ChaCha in-frame)?
□ tshark -r cap.pcap -T fields -e frame.number -e ip.src -e tcp.payload
```

### Phase 2 — Frame layout recovery

```text
□ Align multiple messages of the same type; find invariant bytes / incrementing sequence numbers
□ Length field: big/little-endian, includes header or not
□ Checksums: CRC16/32, checksum, HMAC position
□ Draw the state machine: Connect → Auth → Ready → Request/Response → Close
□ Tools: Wireshark custom dissector draft / ImHex / 010 Editor template / Kaitai Struct
```

### Phase 3 — Serialization & encryption

```text
□ Protobuf: .proto recovery (blackboxprotobuf / pbtk / protoc --decode_raw)
□ gRPC: HTTP/2 headers + protobuf body
□ Encryption: find key derivation (client so/dll/JS) → join ida-reverse / js-reverse / apk-reverse
□ Replay: inside the authorized scope only; harmless fields first, sensitive ops last
```

### Phase 4 — Deliverables

```text
MUST produce:
- Message type table (name / opcode / fields)
- At least 1 reproducible decode command or script
- Evidence: raw hex excerpt + decoded result (anonymized)
```

## Toolchain

| Tool | Required | Purpose | Bootstrap |
|------|----------|---------|-----------|
| tshark / Wireshark | strongly suggested | PCAP parsing | manual / winget |
| Python3 | yes | decode scripts | system |
| blackboxprotobuf | optional | unknown protobuf | pip |
| ImHex / 010 | optional | structure templates | manual |
| IDA / r2 / Ghidra | as needed | client serialization functions | see corresponding skill |

## References

- `references/protocol-workflow.md` — frame layout & Protobuf cheat sheet
- Related: `../ida-reverse/` `../js-reverse/` `../firmware-pentest/` `../pentest-tools/`

## Routing context

**Upstream**: `MASTER-ROUTING` R21 · `routing.md`
**Downstream**: client algorithm needed → `ida-reverse`/`js-reverse`; replay/exploit → `pentest-tools`/`api-security`
**Siblings**: `malware-analysis` (C2 protocols), `digital-forensics` (traffic forensics)

## Completion self-check

- [ ] Did I recover the message layout or state machine (not just paste hex)?
- [ ] Is there a reproducible decode command?
- [ ] Did I respect scope / anonymization?
- [ ] Did I complete the field-journal / report Checklist?
