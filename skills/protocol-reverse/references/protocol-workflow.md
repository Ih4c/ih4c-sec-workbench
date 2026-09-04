# Protocol reverse cheat sheet

> Applies to: `protocol-reverse` skill · 2026-07-18

## Common layout patterns

| Pattern | Signature | Hint |
|------|------|------|
| Fixed-length header + body | first 2/4 bytes are the length | note whether the header length is included |
| Magic number | fixed `0xDEAD` etc. | enables stream re-synchronization |
| TLV | type-length-value repeated | the type enum is the message dictionary |
| Protobuf | varint field numbers | `protoc --decode_raw` |
| Encrypted frames | high entropy, no plaintext URL | first look for the nonce/IV neighborhood |

## Minimal Python skeleton

```python
import struct
def parse_frame(buf: bytes):
    magic, length, msg_type = struct.unpack_from(">IHI", buf, 0)
    body = buf[10:10+length]
    return {"magic": magic, "type": msg_type, "body": body}
```

## Extracting TCP payloads from a PCAP

```bash
tshark -r cap.pcap -Y "tcp.port==4433" -T fields -e tcp.payload | head
```
