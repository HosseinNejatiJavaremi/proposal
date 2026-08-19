zstd Message Compression
----
* Author(s): Hossein Nejati Javaremi (@HosseinNejatiJavaremi)
* Approver: markdroth
* Status: Draft
* Implemented in: (none)
* Last updated: 2026-08-19
* Discussion at: (filled after thread exists)

## Abstract

Add Zstandard as a first-class gRPC message compression algorithm. The
content-coding name is `zstd` on `grpc-encoding` and `grpc-accept-encoding`.
Compression stays per-message, same model as gzip. Sending zstd is opt-in.
An implementation that ships the codec MUST decode it and advertise `zstd`.

This gRFC does not add Brotli, Snappy, full-stream compression, or a C-core
pluggable compressor API.

## Background

gRPC message compression is specified in
[compression.md](https://github.com/grpc/grpc/blob/master/doc/compression.md)
and
[PROTOCOL-HTTP2.md](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md).
The content-coding grammar already allows extra names:

```
Content-Coding → "identity" / "gzip" / "deflate" / "snappy" / {custom}
```

C-core and wrapped languages (C++, Python, Ruby, PHP, Objective-C) implement
`identity`, `deflate`, and `gzip`. Python surfaces those three via
[L46](L46-python-compression-api.md). Java ships gzip only (not deflate). Go
ships gzip via `encoding/gzip`. Java and Go also have experimental compressor
registries; those are per-process and do not define a shared wire profile.

Gzip is the only portable non-identity codec, and it is CPU-heavy relative to
the ratio it delivers on typical protobuf payloads. Operators either pay that
cost or disable compression.

Zstandard ([RFC 8878](https://www.rfc-editor.org/rfc/rfc8878.html), IANA
content-coding
[`zstd`](https://www.iana.org/assignments/http-parameters/http-parameters.xhtml#content-coding))
is a better ratio/CPU tradeoff at its default level and is already widely
deployed as an HTTP content-coding. HTTP `Content-Encoding: zstd` is a
different header and is not a substitute for `grpc-encoding`.

Demand and the process requirement are in
[grpc/grpc#26460](https://github.com/grpc/grpc/issues/26460) (open 2021;
reopened 2026-05-26, help-wanted). A C-core patch
([grpc/grpc#41813](https://github.com/grpc/grpc/pull/41813)) was closed pending
a gRFC so C-core, Java, and Go share one name and frame profile.

### Related Proposals

* [L46](L46-python-compression-api.md) — Python compression API. This proposal
  adds `Compression.Zstd` next to `Gzip` / `Deflate`.
* [G2](G2-http3-protocol.md) — HTTP/3 transport. Request/response shape is
  unchanged; `grpc-encoding` still applies.

### Non-goals

* Brotli or Snappy as first-class codecs.
* Full-stream / HTTP-style compression (shared window across messages).
* A new C-core “register your own compressor” API.
* Stabilizing Java `CompressorRegistry` or Go `encoding.RegisterCompressor`.
* Trained dictionaries, skippable-frame application data, or concatenated
  frames as part of the gRPC profile.
* Changing the default channel algorithm away from no compression.
* Remapping `GRPC_COMPRESS_LEVEL_*` onto zstd.

## Proposal

### Wire name

The content-coding name is **`zstd`** (lowercase), matching IANA. It is sent
in `grpc-encoding` when messages on the call are compressed with this codec,
and listed in `grpc-accept-encoding` when the peer can decode it.

Update PROTOCOL-HTTP2:

```
Content-Coding → "identity" / "gzip" / "deflate" / "snappy" / "zstd" / {custom}
```

`snappy` remains a reserved name only. This gRFC does not implement it.

`grpc-encoding` is **per-call**. Every compressed message on that call uses
zstd. Individual messages MAY still have Compressed-Flag 0 (identity payload)
when compression is disabled for that message (BEAST/CRIME / “next message
uncompressed”) or when the implementation skips compression because it would
not help. Compression contexts MUST NOT be reused across messages.

### Frame profile

Each compressed gRPC message (Compressed-Flag = 1, `grpc-encoding: zstd`) is
exactly one RFC 8878 Zstandard **Frame**.

Encoder MUST:

1. Emit a single Frame whose magic is `0xFD2FB528`.
2. Set Dictionary_ID to 0 (no dictionary).
3. NOT emit skippable frames.
4. NOT concatenate additional frames.
5. Use default compression level **3** unless the application selected another
   level through a language API. The level is not on the wire.

Encoder MAY include the content checksum (xxHash-64 footer). Receivers MUST
accept frames with or without a checksum and MUST verify the checksum when
present.

Decoder MUST:

1. Decode one RFC 8878 Frame from the message bytes.
2. Treat leftover bytes after that Frame as invalid.
3. Treat a non-zero Dictionary_ID as invalid.
4. Treat a skippable frame as invalid.
5. Bound decompressed output by the call’s max-receive-message-size
   (implementation default if unset). Stop inflating and fail the RPC if the
   bound would be exceeded. `RESOURCE_EXHAUSTED` is RECOMMENDED; `INTERNAL` is
   acceptable where the language already uses it for corrupt compressed
   payloads.
6. SHOULD reject a frame whose promised window size exceeds
   max-receive-message-size (or an implementation cap, whichever is smaller)
   before allocating that window.

Invalid compressed payloads:

* Receiving **server** → `UNIMPLEMENTED`, with `grpc-accept-encoding` listing
  encodings the server accepts. That list MUST NOT include the encoding that
  caused the failure if the server does not support it. If the server *does*
  support `zstd` but the frame is malformed, `INTERNAL` is also acceptable.
* Receiving **client** → `INTERNAL`.

Existing unknown-encoding rules in compression.md still apply when the peer
does not implement zstd at all.

### Negotiation

Unchanged from compression.md, with `zstd` as another algorithm:

* Server cannot decode the request encoding → `UNIMPLEMENTED` +
  `grpc-accept-encoding`.
* Client cannot decode the response encoding → `INTERNAL`.
* A peer MAY omit encodings from `grpc-accept-encoding`. If it later accepts a
  message in an omitted-but-supported encoding, it MUST include that encoding
  on the response `grpc-accept-encoding`.
* Server MUST NOT compress a response with an encoding absent from the
  client’s most recent `grpc-accept-encoding`; send uncompressed instead.
* Asymmetry is allowed (gzip request, zstd response, or identity).
* Per-message disable is unchanged: next message MUST have Compressed-Flag 0.

Implementations MAY skip compressing a given message (empty or incompressible)
and send Compressed-Flag 0 even when the call encoding is `zstd`.

### Defaults

* Channel/call default algorithm remains **no compression**.
* Sending zstd is **opt-in** via the existing per-channel / per-call API.
* After the experimental period, an implementation that includes zstd MUST
  advertise `zstd` and MUST decode it. Encode stays opt-in.
* **v1 does not change compression-level → algorithm mapping.** Levels today
  map onto gzip/deflate. Remapping `GRPC_COMPRESS_LEVEL_HIGH` onto zstd would
  change the wire encoding for servers that use levels without an explicit
  algorithm.

### C-core, C++, wrapped languages

Add `GRPC_COMPRESS_ZSTD` immediately before `GRPC_COMPRESS_ALGORITHMS_COUNT`
so existing bitset values for NONE / DEFLATE / GZIP stay stable:

```c
typedef enum {
  GRPC_COMPRESS_NONE = 0,
  GRPC_COMPRESS_DEFLATE,
  GRPC_COMPRESS_GZIP,
  GRPC_COMPRESS_ZSTD,
  GRPC_COMPRESS_ALGORITHMS_COUNT
} grpc_compression_algorithm;
```

On-wire name for this enum value is `zstd`.

C-core SHOULD vendor a stable **release** of libzstd (not an arbitrary
commit) and SHOULD allow cmake/pkg-config to use a system libzstd, same
pattern as zlib.

Default enabled-algorithm bitset includes zstd once the codec is
non-experimental. `SetCompressionAlgorithmSupportStatus(GRPC_COMPRESS_ZSTD,
false)` (and the channel-arg bitset) remains the way to disable it.

C++ uses the existing surface; only the new enum value is added:

```cpp
ChannelArguments args;
args.SetCompressionAlgorithm(GRPC_COMPRESS_ZSTD);
auto channel = grpc::CreateCustomChannel(target, creds, args);

ClientContext context;
context.set_compression_algorithm(GRPC_COMPRESS_ZSTD);

ServerBuilder builder;
builder.SetDefaultCompressionAlgorithm(GRPC_COMPRESS_ZSTD);
```

Python (L46):

```python
class Compression(enum.IntEnum):
    NoCompression = _cygrpc.GRPC_COMPRESS_NONE
    Deflate = _cygrpc.GRPC_COMPRESS_DEFLATE
    Gzip = _cygrpc.GRPC_COMPRESS_GZIP
    Zstd = _cygrpc.GRPC_COMPRESS_ZSTD
```

```python
channel = grpc.insecure_channel(target, compression=grpc.Compression.Zstd)
stub.Method(req, compression=grpc.Compression.Zstd)
```

Ruby, PHP, and Objective-C expose the new C-core enum value the same way they
expose gzip. No wrapped-language pluggable compressor API.

Revive [grpc/grpc#41813](https://github.com/grpc/grpc/pull/41813) after this
gRFC merges.

### Java

Java `Compressor` / `Decompressor` registries stay experimental. This gRFC
does not stabilize them.

zstd MUST be a codec whose `getMessageEncoding()` returns `"zstd"`. To avoid
forcing a native library on every application, the implementation SHOULD live
in an optional artifact (for example `grpc-zstd`) that registers at runtime.
Applications that depend on the artifact get encode + decode; others behave
as today.

A pure-Java decoder/encoder is acceptable if it interops with libzstd frames
from C-core and Go. JNI to libzstd is acceptable inside the optional artifact.

`Codec.Gzip` remains the only compressor in the core artifact (Java does not
ship deflate today; this gRFC does not change that).

```java
stub.withCompression("zstd").method(req);
```

### Go

`encoding.RegisterCompressor` stays experimental. This gRFC does not change
it.

Add `google.golang.org/grpc/encoding/zstd`, analogous to `encoding/gzip`:

* `Name()` returns `"zstd"`.
* Blank import registers the compressor.
* Clients send with `grpc.UseCompressor("zstd")`.
* Servers decode automatically once the package is imported.

```go
import _ "google.golang.org/grpc/encoding/zstd"

resp, err := client.Method(ctx, req, grpc.UseCompressor("zstd"))
```

Go 1.26 `compress/zstd` is decompress-only. Encode+decode SHOULD use
`github.com/klauspost/compress/zstd` (or stdlib once it grows an encoder).
Library choice is an implementation detail; frames MUST match this profile.

### Other languages

Same wire name and frame profile. Node, C#, Dart, and others implement after
C-core ↔ Java ↔ Go interop exists. Same calendar release is not required.

### Proxies and existing custom codecs

Transparent HTTP/2 proxies that do not inspect message payloads are
unaffected.

Proxies, gateways, or transcoders that decompress gRPC messages MUST learn
`zstd` or they will fail the same way they fail unknown encodings today.
That work is outside this gRFC; the name and profile here are what they
would implement.

Applications that already registered a custom compressor under the name
`zstd` in Java or Go MUST match this frame profile or use a different private
name. First-class `zstd` owns the name.

### Interop tests

Extend existing compression interop with `zstd`:

1. Client sends zstd; implementing server decodes; response may be zstd,
   gzip, or identity.
2. Client sends zstd; server without zstd returns `UNIMPLEMENTED` and
   `grpc-accept-encoding` that does not list `zstd` as a supported coding.
3. Compressed-Flag=1, `grpc-encoding: zstd`, one valid Frame.
4. Compressed-Flag=0 still works on a call with `grpc-encoding: zstd`.
5. Identity-only still works when zstd is compiled in.
6. Inflating past max-receive-message-size fails.
7. Non-zero Dictionary_ID fails.
8. Leftover bytes after one Frame fail.
9. Asymmetry: gzip request / zstd response, and the reverse.
10. C-core ↔ Java ↔ Go, all encode/decode combinations.

### Documentation

* `doc/compression.md` — list `zstd`; point at this gRFC for the frame
  profile.
* `doc/PROTOCOL-HTTP2.md` — add `"zstd"` to `Content-Coding`.
* Language API docs and compression examples (C++, Python, Java, Go).

### Temporary environment variable protection

C-core and wrapped languages, until C-core / Java / Go interop exists:

| Variable | Default | Meaning |
| --- | --- | --- |
| `GRPC_EXPERIMENTAL_ZSTD_COMPRESSION` | unset / `false` | When not `true`, MUST NOT advertise `zstd` and MUST NOT encode with it. Incoming `zstd` is treated as an unknown encoding. |

After soak, remove the variable. Decode+advertise become default for builds
that include the codec. Encode remains API opt-in.

Java and Go do **not** need this variable while experimental: absence of the
optional artifact / blank import is the flag (same pattern as Go gzip). Those
packages MAY still honor the variable if it is cheap; not required.

## Rationale

### Why zstd, not Brotli or Snappy

Brotli is tuned for static HTTP assets (slow compress, high ratio). gRPC
compresses per RPC on the caller’s CPU. Wrong tradeoff.

Snappy is already a reserved PROTOCOL-HTTP2 name and appears in some Java
forks via the experimental registry. zstd typically dominates it on ratio at
similar or better CPU. One new first-class codec is enough.

zstd is RFC-specified, IANA-registered, and the codec requested in #26460.

### Why a built-in codec, not a C-core plugin API

Java and Go already have experimental registries; C-core does not. A
cross-language BYO API has been declined (abuse as a generic transform;
wrapped-language surface). A single named codec matches gzip and is what
#26460 needs.

### Why not full-stream compression

HTTP-style compression across messages cannot use the current per-message
Compressed-Flag model. Separate design.

### Why levels do not silently switch to zstd

Servers using `GRPC_COMPRESS_LEVEL_*` would start emitting a coding older
clients do not understand. Explicit algorithm selection is the compatible
path.

### Why dictionaries are forbidden

A shared dictionary needs distribution and versioning. Independent frames
interop without it.

### Why an optional Java artifact

grpc-java already omits deflate to keep the core artifact small and free of
extra native deps. zstd should follow gzip’s *wire* status without forcing
JNI onto every user. Go’s blank-import package is the same idea.

## Implementation

1. This gRFC — comment period, then merge. C-core, Java, and Go language
   owners should review (the original #26460 thread: markdroth, ejona86,
   dfawley).
2. C-core — revive #41813 behind `GRPC_EXPERIMENTAL_ZSTD_COMPRESSION`.
   Wrapped languages add the enum (Python `Compression.Zstd`, etc.).
3. Java and Go in parallel — optional Java artifact; Go `encoding/zstd`.
   Cross-interop against C-core.
4. Remove the C-core env var once the three interop.
5. Other languages as owners have capacity.

The gRFC author can iterate on the C-core patch. Java and Go still need
language-owner review if contributed externally.

No language is required to ship in the same release. The wire profile must
be identical.

Rolling upgrade: a client MUST NOT send zstd unless it is willing to handle
`UNIMPLEMENTED` from older servers (same as gzip). Servers that advertise
`zstd` can decode it; they still send gzip or identity to clients that do
not list `zstd`. Enabling send-zstd as a **default** on a mixed fleet is an
operator choice, not this gRFC’s default.

## Open issues

1. **Java artifact vs core.** Optional `grpc-zstd` is the recommendation. A
   pure-Java codec in core also satisfies this gRFC if the name is `zstd`
   and frames interop.
2. **Go encoder library.** klauspost/compress vs waiting for a stdlib
   encoder. Interop with libzstd is the acceptance test.
3. **libzstd version.** Pin a stable release in C-core `third_party` at
   implementation time. This gRFC does not freeze the version.
4. **Level remapping.** Later update, once zstd is widely deployed.
)
