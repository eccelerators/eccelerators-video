# Pixel stream contract

`IPixelStream` is a single-clock ready/valid interface. One token transfers at a
rising context edge when both signals are true. The token comprises `data`,
`frameId`, `frameStart`, `lineEnd` and `frameEnd`; its fields remain stable while
valid is asserted and the consumer stalls.

Gray8 occupies the low byte with a zero high byte. RGB565 uses all 16 bits, with
red in bits 15..11. Format is agreed outside the interface and remains constant
within a frame. Markers are meaningful only on valid pixels. A 1×1 frame has all
three markers asserted; frame identity remains constant throughout the frame.

## Ownership and backpressure

The producer owns payload and valid; exactly one consumer owns ready. Wire all
token fields together, and return readiness through every stage. Pixel transport
uses signal assignments rather than scheduled function calls. `PixelBypass` has
no storage, `PixelFilter` holds one elastic token, and `Window3x3` has explicit
line storage and a maximum input rate of one pixel every two context edges.
`PatternSource` can supply one pixel per edge with no line blanking and a ready
consumer; the sustained-rate test checks 36 accepted pixels in 36 active cycles.

Keep a pattern source's enable asserted and its pattern selector stable for an
active frame, including during stalls. Deasserting enable gates valid immediately;
it does not rewind the raster or complete the current frame.

## Reset and incomplete frames

Reset cancels in-flight tokens and restores each component's control state.
Consumers discard incomplete frames and resynchronize on an accepted frameStart.
Pattern sources restart at the first pixel; DVP capture first waits for an observed
VSYNC pulse. Line RAM is retained, and the first two rows of a fresh raster fill
it before a 3×3 window becomes valid.

A physical camera cannot honor backpressure. `DvpRgb565Capture` rejects malformed
or overflowing frames, counts the fault and waits for a subsequent frame boundary.
Its final pixel/END is withheld until geometry validation. Earlier pixels of a
rejected frame may already be downstream. A consumer must require a complete,
consistent raster ending in frameEnd before displaying or publishing that frame.

## Clock crossings and configuration

Every stage operates in its assigned clock context. The package performs no
implicit CDC. Cross the entire pixel token through an asynchronous FIFO; use a
coherent snapshot or handshake for diagnostic counters. Size buffering for the
producer rate, consumer stalls and any slower processing stage.

`PixelFilter` latches requested mode, threshold and brightness on an accepted
frameStart. Its first output pixel uses the new settings. The consuming design
must keep the published image format aligned with the applied mode. Request
versions, acknowledgements and cross-clock control mailboxes are application
responsibilities; the package does not provide a configuration-generation field.
Do not synchronize changing multibit settings or counters bit by bit.
