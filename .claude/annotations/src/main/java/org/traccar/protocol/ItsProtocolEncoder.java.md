# ItsProtocolEncoder.java

**Role:** Encodes `ENGINE_STOP` and `ENGINE_RESUME` commands into the ITS relay control strings for AIS-140 immobilizer support. Extends `StringProtocolEncoder`.
**Fits in:** Not directly in the pipeline — see `ItsProtocol.java` notes. Used when `BaseProtocol.sendDataCommand` detects a `StringEncoder` in the pipeline and writes the string directly, OR as a `textCommandEncoder` for SMS delivery.
**Read next:** [[ItsProtocol.java]], [[StringProtocolEncoder.java]], [[BaseProtocolEncoder.java]]

## Public API

- `encodeCommand(Command command)` — returns the relay command string or null.

## Key flows

### Command strings (lines 30-34)
```
ENGINE_STOP    → "@SET#RLP,OP1,"    (immobilizer relay activate — cut engine)
ENGINE_RESUME  → "@CLR#RLP,OP1,"   (immobilizer relay deactivate — resume engine)
```
These are ITS vendor-specific AT-command-style strings. `OP1` is the relay output pin designation in the AIS-140 VLT hardware.

## Gotchas / non-obvious

- The trailing comma in `"@SET#RLP,OP1,"` is intentional — some VLT firmware variants require it for parsing.
- **RLP in the command string is unrelated to the RLP frame variant** in `ItsProtocolDecoder`. It refers to "Relay Protocol" in the command set.
- For Transport OS AIS-140 compliance: these commands implement the AIS-140 mandatory "remote engine cut" (immobilizer) requirement for ASTAM-certified VLTs.

## Line index

- 29-34 — `encodeCommand` switch
- 31 — `ENGINE_STOP` → `"@SET#RLP,OP1,"`
- 32 — `ENGINE_RESUME` → `"@CLR#RLP,OP1,"`
