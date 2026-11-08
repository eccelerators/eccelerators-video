# Changelog

## 1.0.1

- Add package discovery metadata and include the README, changelog and license
  in publication.
- Update the Livt.IO dependency to 1.8.1.

## 1.0.0 — 2026-10-23

Initial stable release of Eccelerators.Video.

### Added

- `IPixelStream` with coherent ready/valid pixel data, frame markers and identity.
- Parameterized ramp, checker and RGB565 color-bar sources, line blanking and
  combinational pixel forwarding.
- Elastic RGB565 point filtering: bypass, grayscale, inversion, saturating
  brightness and threshold, with settings sampled at accepted frame starts.
- A cropped Gray8 3×3 window with two explicit synchronous block-RAM line stores.
- Rounded blur and saturated L1 Sobel arithmetic, plus RGB565 luminance helpers.
- Parallel high-byte-first RGB565 capture with malformed-frame and overflow
  counters, next-frame recovery and delayed final-pixel publication.
- Eight Livt test components, source/window synthesis probes and a compilable
  grayscale composition example.

### Changed

- Use published Livt.IO 1.3.0 for synchronous line RAM.
- Align package and probe versions with 1.0.0.
- Apply design-guide naming and documentation conventions, including descriptive
  private window registers and explicit ownership, reset and throughput contracts.
- Replace development-stage documentation with consumer-facing API and integration
  guidance while preserving existing clock boundaries.

### Integration scope

- Components operate in the context supplied by their composition owner. The
  package does not include a clock-domain bridge, camera register configuration,
  frame transport or a board pin/constraint design.
- DVP capture cannot pause a physical sensor. Consumers must handle incomplete
  frames and cross entire tokens and diagnostic snapshots coherently.
- The window emits `(WIDTH-2) × (HEIGHT-2)` pixels and accepts at most one input
  pixel every two context edges; pipeline sizing must account for that rate.
- Package simulations and historical synthesis probes are distinct from board
  timing closure and physical camera qualification.
