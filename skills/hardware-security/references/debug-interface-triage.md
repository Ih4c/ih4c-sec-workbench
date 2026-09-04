# Debug Interface Triage (调试接口分诊)

1. Find the silkscreen markings: TX RX GND VCC TDI TDO TCK TMS
2. Match voltage levels before connecting
3. First only read the serial logs
4. Record the U-Boot interrupt key and environment variables (do not casually saveenv)
5. SHA256 the image after dumping
