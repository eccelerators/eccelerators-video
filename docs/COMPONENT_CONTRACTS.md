# Video component contracts

These are the Eccelerators.Video 1.0.0 component contracts. Context/reset comes
from the composition owner unless noted otherwise. Board qualification belongs
to the consuming design.
All stream tokens contain data plus frameStart, lineEnd, frameEnd and frameId.
A token transfers only at a rising context edge with both valid and ready true.

| Component | Ownership and clock | Latency / throughput | Storage / failure contract |
|---|---|---|---|
| PatternSource | Owns raster; one system domain | Combinational output; one accepted pixel/edge without configured blanking | Counters only; holds token under stalls; controls stable within frame |
| PixelBypass | Borrows explicit token signals | Zero registered latency | No storage; returns readiness directly |
| PixelFilter | Owns one elastic token and applied settings | One edge; at most one token/edge | Registers only; requested mode/threshold/brightness latch on accepted frameStart |
| DvpRgb565Capture | Owns byte assembly and counters in PCLK | One pixel per two active PCLK edges; final pixel waits for VSYNC | One output token plus pending final pixel; cannot stall camera; discards on overflow/geometry error and resumes next falling VSYNC |
| Window3x3 | Owns two independent same-clock line memories | Two edges per accepted input pixel; emits after first two rows/columns | Explicit SynchronousRam with MemoryStyle.Block; two WIDTH-byte rows; retains storage on reset but initializes before output |
| PixelMath | Pure combinational arithmetic | Depends on composition's registered paths | No storage; RGB565 luminance and saturating brightness |
| NeighborhoodMath | Pure combinational arithmetic | Depends on composition's registered paths | No storage; rounded mean and saturated L1 Sobel |
| VideoProbe / WindowProbe | Standalone synthesis compositions | See saved reports | No board pin assignments or external I/O timing: never program these probes on the physical board |

`Window3x3` crops all four borders, producing (WIDTH-2)×(HEIGHT-2).
Its input must already satisfy the complete raster contract. After an interrupted
frame, a new frameStart resets position; two new rows initialize the line stores
before a new valid window. It does not independently validate physical syncs.
Nine output pixels are named by position to make row/column ordering explicit.

`DvpRgb565Capture` expects active-high VSYNC blanking, active-high HREF and RGB565
high byte first. These are an explicit input contract, **not a statement that the
connected OV5640 has been configured that way**. It waits for a VSYNC pulse after
reset. An invalid frame cannot emit END because the final pixel waits for complete
geometry validation. Already emitted pixels of that frame may remain downstream;
they must never be treated as a complete image without END.

Capture counters belong to PCLK. Reading their binary buses directly from a
system-clock process is unsafe. An asynchronous FIFO must transfer the complete
pixel token, and coherent snapshots/handshakes must transfer diagnostics. The
consuming design owns that CDC adapter, its timing constraints, stopped-clock
reset/recovery tests and physical PCLK qualification. Livt.Collections AsyncFifo
can provide the FIFO core; reset coordination and payload storage remain explicit
integration choices. No clock crossing is hidden in this package.

Point filter mode 0 passes RGB565, 1 emits Gray8, 2 inverts Gray8, 3 adds saturating
brightness, 4 thresholds Gray8 (greater-than-or-equal gives 255). Invalid modes
select bypass. Gray8 uses the low byte of the 16-bit token. The frame's format
must be published in the transport header; do not infer it from changing requests.
Blur computes `(sum + 4) / 9`. Sobel clips `abs(Gx) + abs(Gy)` to 255; it does not
compute the Euclidean magnitude or normalize it. Numeric bounds are explicit in
the source to avoid unnecessary wide arithmetic paths.

The composition root connects individual input/output signals. Owned registers
with one combinational export keep data, metadata and valid aligned. Preserve
that alignment and return readiness through each stage when composing a pipeline.

## Supported parameters and settings

- `PatternSource<WIDTH, HEIGHT, LINE_BLANK>`: width and height are 1..4096;
  line blanking is nonnegative. Pattern 0 (and other unmatched selectors) emits
  a Gray8 ramp, 1 an 8×8 checker, and 2 RGB565 bars. Keep enable asserted and
  pattern stable while a frame is active, including during downstream stalls.
- `Window3x3<WIDTH, HEIGHT>`: width is 3..4096 and height is at least 3.
  Inputs must follow the declared raster. `inputStart` restarts its position;
  physical line/frame validation belongs upstream. The nine named outputs and
  metadata share one `valid`/`ready` handshake.
- `DvpRgb565Capture<WIDTH, HEIGHT>`: dimensions are 1..4096. It arms after an
  observed VSYNC pulse, validates active-byte geometry and resumes on a subsequent
  falling VSYNC after rejection. Counters and frame identities wrap as `uint`.
- `PixelFilter`: mode 0..4 as listed above; threshold is a Gray8 byte and brightness
  is a signed intensity offset. Only `appliedMode` is publicly reported;
  there is no built-in configuration-generation or acknowledgement protocol.

`NeighborhoodMath` arguments are row-major: `a,b,c / d,e,f / g,h,i`.
Sobel omits `e`, the center pixel. `PixelMath.Gray` expands RGB565 channels by bit
replication and computes `(77*R + 150*G + 29*B + 128) >> 8`.

Stream payload and marker outputs are meaningful only while their accompanying
valid signal is asserted. Reset cancels in-flight work; a consumer must discard an
incomplete frame. Line RAM contents are not cleared by reset.
