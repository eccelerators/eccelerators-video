# Getting started with Eccelerators.Video 1.0.0

## Install and import

Once 1.0.0 is published, declare the exact registry dependency:

```toml
[dependencies]
"Eccelerators.Video" = "1.0.0"
```

Run `livt sync` and add `using Eccelerators.Video` to the consuming Livt source.
The package brings Livt.IO 1.3.0 for its synchronous line memories. It does not
require a particular FPGA board, sensor control bus or Ethernet implementation.

## Start with a synthetic stream

The complete [GrayscaleVideo example](../examples/grayscale/src/GrayscaleVideo.lvt)
owns a 320×240 `PatternSource` and `PixelFilter` in one 50 MHz context. Bind its
`enabled` constructor argument to your stream enable and `pattern` to a stable
selector with value 2 (RGB565 color bars). Its consumer drives `ready` and samples
all output fields only when `valid && ready` is true at a clock edge.

The example uses a path dependency so it can be checked from this checkout before
the release is published:

```sh
livt sync --project examples/grayscale
livt validate --project examples/grayscale --offline -R
```

The signal wiring in `Wire[]()` is combinational. It connects pixel data and all
four metadata fields, then propagates readiness backwards. `PixelFilter` holds
its output when the consumer stalls. Do not connect only `data` and `valid`:
omitting readiness or a frame marker breaks the stream contract.

The example expects RGB565 input. Pattern 0 produces a Gray8 ramp and pattern 1
a Gray8 checker; use those directly with a Gray8 consumer or `Window3x3`, rather
than feeding their bytes into the RGB565 grayscale converter.

## Select a point filter

`PixelFilter` accepts RGB565 and samples these inputs on an accepted frameStart:

| requestedMode | Output | Additional setting |
|---|---|---|
| 0 | Unchanged RGB565 | None |
| 1 | Gray8 | None |
| 2 | Inverted Gray8 | None |
| 3 | Gray8 plus saturating brightness | `requestedBrightness`: signed intensity offset |
| 4 | Gray8 threshold: 255 when gray >= threshold, otherwise 0 | `requestedThreshold`: byte |

Other mode values select bypass. Gray8 occupies the low byte of the 16-bit output.
Keep downstream format metadata consistent with the applied mode. `appliedMode`
reports the sampled mode; changing requests mid-frame does not alter that frame.

## Add a neighborhood operation

Connect a Gray8 stream to `Window3x3<WIDTH, HEIGHT>`, forwarding data, markers and
frame identity, and return `upstreamReady` to the producer. The window owns two
`SynchronousRam` line stores with `MemoryStyle.Block`; each stores WIDTH bytes.
It does not allocate a complete frame buffer.

A valid output supplies nine pixels named by row and column. Pass them in row-major
order to `NeighborhoodMath.Blur`, or omit the center pixel for
`NeighborhoodMath.Sobel`. The blur rounds the mean. Sobel returns the saturated
L1 gradient magnitude, not a normalized or Euclidean magnitude. The
[WindowProbe](../src/WindowProbe.lvt) shows both operations feeding a common
elastic output register with aligned metadata.

The output raster is `(WIDTH-2) × (HEIGHT-2)`. Budget for at most one accepted
input pixel every two context edges, initial row filling and downstream stalls.
A faster producer needs sufficient buffering or a lower sustained rate.

## Replace patterns with camera input

`DvpRgb565Capture` samples high-byte-first RGB565 in the supplied PCLK context,
with active-high HREF and active-high VSYNC blanking. The consuming board design
must configure the sensor for that contract and meet physical input timing.
The package does not configure an OV5640 or generate sensor clocks.

Cross complete captured tokens through an asynchronous FIFO before using a
system-clock filter. A camera cannot pause when ready becomes false; capture
counts overflow, rejects that frame and recovers at a later frame boundary.
Its earlier pixels may already have crossed the FIFO, so require a complete
raster and frameEnd before treating an image as valid. Cross counters using a
coherent snapshot, not independent bit synchronizers.

Review [component contracts](COMPONENT_CONTRACTS.md) and the
[stream contract](STREAM_CONTRACT.md) when connecting these components to your
application’s sensor control, clock domains and output interface.
