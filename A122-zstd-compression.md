zstd Message Compression
----
* Author(s): Hossein Nejati Javaremi (@HosseinNejatiJavaremi)
* Approver: markdroth
* Status: Draft
* Implemented in: (none)
* Last updated: 2026-08-19
* Discussion at: (filled after thread exists)

Table of Contents
----
  * [Abstract](#abstract)
  * [Background](#background)
     * [Current support](#current-support)
     * [Related Proposals](#related-proposals)
     * [Non-goals](#non-goals)
  * [Proposal](#proposal)
     * [Overview](#overview)
     * [Wire name](#wire-name)
     * [Call vs message](#call-vs-message)
     * [Frame profile](#frame-profile)
     * [Worked example](#worked-example)
     * [Negotiation and errors](#negotiation-and-errors)
     * [Defaults and compression levels](#defaults-and-compression-levels)
     * [Safety](#safety)
     * [C-core, C++, wrapped languages](#c-core-c-wrapped-languages)
     * [Java](#java)
     * [Go](#go)
     * [Other languages](#other-languages)
     * [Proxies and existing custom codecs](#proxies-and-existing-custom-codecs)
     * [Observability](#observability)
     * [Interop tests](#interop-tests)
     * [Documentation](#documentation)
     * [Temporary environment variable protection](#temporary-environment-variable-protection)
  * [Rationale](#rationale)
  * [Implementation](#implementation)
  * [Compatibility and rollout](#compatibility-and-rollout)
  * [Open issues](#open-issues)

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
A compressed message is a length-prefixed payload with Compressed-Flag = 1,
using the algorithm named in the per-call `grpc-encoding` header. Compression
contexts are **not** maintained across messages.

The content-coding grammar already allows extra names:

```
Content-Coding → "identity" / "gzip" / "deflate" / "snappy" / {custom}
```

`gzip` is the only non-identity codec that is portable across C-core, Java,
and Go. `deflate` exists in C-core and is absent from Java. `snappy` is a
reserved name only.

Gzip is CPU-heavy relative to the ratio it delivers on typical protobuf
payloads. Operators who care about bandwidth either pay that cost or disable
compression. That is the problem reported in
[grpc/grpc#26460](https://github.com/grpc/grpc/issues/26460) (open 2021;
reopened 2026-05-26, help-wanted).

Zstandard ([RFC 8878](https://www.rfc-editor.org/rfc/rfc8878.html), IANA
content-coding
[`zstd`](https://www.iana.org/assignments/http-parameters/http-parameters.xhtml#content-coding))
is a better ratio/CPU tradeoff at its default level and is already widely
deployed as an HTTP content-coding. HTTP `Content-Encoding: zstd` is a
**different header** and is not a substitute for `grpc-encoding`. Peers that
speak HTTP zstd still cannot speak gRPC zstd until this codec exists on both
ends of the RPC.

A C-core patch ([grpc/grpc#41813](https://github.com/grpc/grpc/pull/41813))
was closed pending a gRFC so C-core, Java, and Go share one name and frame
profile. Java and Go already have experimental compressor registries; those
are per-process and do not define a shared wire profile. Python surfaces
C-core algorithms only, via [L46](L46-python-compression-api.md).

### Current support

| Implementation | `identity` | `gzip` | `deflate` | `zstd` today |
| --- | --- | --- | --- | --- |
| C-core / C++ / Python / Ruby / PHP / Obj-C | yes | yes | yes | no |
| Java | yes | yes | no | custom registry only |
| Go | yes | `encoding/gzip` import | no | custom `RegisterCompressor` only |
| Node, C#, Dart, others | identity + gzip typical | varies | no | no first-class codec |

“Custom registry only” means an application can register a compressor named
`zstd` **inside one process**. The peer in another language will not decode
it unless it implements the same frame. That is not a portable codec.

### Related Proposals

* [L46](L46-python-compression-api.md) — Python compression API. This proposal
  adds `Compression.Zstd` next to `Gzip` / `Deflate`.
* [G2](G2-http3-protocol.md) — HTTP/3 transport. Request/response shape is
  unchanged; `grpc-encoding` still applies.
* [A66](A66-otel-stats.md) — compressed message-size metrics. zstd changes
  the numbers, not the metric definitions.
* [A16](A16-binary-logging.md) — logs application messages. Implementations
  SHOULD log **uncompressed** payloads (same as gzip).

### Non-goals

* Brotli or Snappy as first-class codecs.
* Full-stream / HTTP-style compression (shared window across messages).
* A new C-core “register your own compressor” API.
* Stabilizing Java `CompressorRegistry` or Go `encoding.RegisterCompressor`.
* Trained dictionaries, skippable-frame application data, or concatenated
  frames as part of the gRPC profile.
* Changing the default channel algorithm away from no compression.
* Remapping `GRPC_COMPRESS_LEVEL_*` onto zstd in v1.
* Requiring every language to ship in the same calendar release.

## Proposal

### Overview

1. Standardize the content-coding name `zstd`.
2. Standardize the per-message frame: one RFC 8878 Frame, no dictionary.
3. Add the codec to C-core (enum + libzstd), Java (optional artifact), and Go
   (`encoding/zstd`). Wrapped languages follow C-core.
4. Encode remains opt-in. Decode+advertise becomes default after an
   experimental period.
5. Existing gzip/deflate/identity behavior does not change.

### Wire name

The content-coding name is **`zstd`** (lowercase), matching IANA. It is sent
in `grpc-encoding` when messages on the call may be compressed with this
codec, and listed in `grpc-accept-encoding` when the peer can decode it.

Update PROTOCOL-HTTP2:

```
Content-Coding → "identity" / "gzip" / "deflate" / "snappy" / "zstd" / {custom}
```

`snappy` remains a reserved name only. This gRFC does not implement it.

### Call vs message

`grpc-encoding` is **per-call** (request headers and response headers). Every
compressed message on that call uses zstd.

Individual messages MAY still have Compressed-Flag 0 (raw protobuf bytes)
when:

* the application disabled compression for the next message (BEAST/CRIME);
* the implementation skips compression because the payload is empty or
  incompressible.

Compression contexts MUST NOT be reused across messages. A streaming RPC with
N messages is N independent zstd frames (or uncompressed messages), never one
long zstd stream.

If `grpc-encoding` is omitted, Compressed-Flag MUST be 0, as today.

### Frame profile

Each compressed gRPC message (Compressed-Flag = 1, `grpc-encoding: zstd`) is
exactly one RFC 8878 Zstandard **Frame**.

```
Length-Prefixed-Message
├── Compressed-Flag     1 byte, value 1
├── Message-Length      4 bytes, big-endian, length of Message
└── Message             N bytes = one RFC 8878 Frame
    ├── Magic_Number    4 bytes, 0xFD2FB528
    ├── Frame_Header    including Dictionary_ID = 0
    ├── Data_Blocks     …
    └── Content_Checksum  optional xxHash-64
```

Encoder MUST:

1. Emit a single Frame whose magic is `0xFD2FB528`.
2. Set Dictionary_ID to 0 (no dictionary).
3. NOT emit skippable frames.
4. NOT concatenate additional frames.
5. Use default compression level **3** unless the application selected another
   level through a language API. The level is not on the wire.

Encoder MAY include the content checksum (xxHash-64 footer).

Decoder MUST:

1. Decode one RFC 8878 Frame from the message bytes.
2. Treat leftover bytes after that Frame as invalid.
3. Treat a non-zero Dictionary_ID as invalid.
4. Treat a skippable frame as invalid.
5. Accept frames with or without a content checksum, and MUST verify the
   checksum when present.
6. Bound decompressed output (see [Safety](#safety)).

Level, window log, and checksum presence are encoder choices. They are not
negotiated. Interop is “any encoder in this profile, any decoder in this
profile.”

### Worked example

Unary call, client opts into zstd, server also speaks zstd and echoes it.

**Request**

```
HEADERS (flags = END_HEADERS)
:method = POST
:scheme = http
:path = /helloworld.Greeter/SayHello
:authority = example.test
content-type = application/grpc
te = trailers
grpc-encoding = zstd
grpc-accept-encoding = identity,deflate,gzip,zstd

DATA (flags = END_STREAM)
<Length-Prefixed-Message: Compressed-Flag=1, zstd Frame of HelloRequest>
```

**Response**

```
HEADERS (flags = END_HEADERS)
:status = 200
content-type = application/grpc
grpc-encoding = zstd
grpc-accept-encoding = identity,deflate,gzip,zstd

DATA
<Length-Prefixed-Message: Compressed-Flag=1, zstd Frame of HelloReply>

HEADERS (flags = END_STREAM, END_HEADERS)
grpc-status = 0
```

Same call if the server does not implement zstd:

```
HEADERS (flags = END_HEADERS)
:status = 200
content-type = application/grpc
grpc-accept-encoding = identity,gzip

HEADERS (flags = END_STREAM, END_HEADERS)
grpc-status = 12   # UNIMPLEMENTED
grpc-message = %sCompression algorithm not supported
```

(`grpc-accept-encoding` MUST NOT list `zstd` in that failure. Exact
`grpc-message` text is implementation-defined; status code is not.)

Asymmetric example: client sends gzip, lists `zstd` in
`grpc-accept-encoding`; server responds with zstd. Valid.

### Negotiation and errors

Existing compression.md rules apply, with `zstd` as another algorithm.

| Situation | Who fails | Status | Notes |
| --- | --- | --- | --- |
| Peer sends `grpc-encoding: zstd`, receiver has no zstd | server | `UNIMPLEMENTED` | `grpc-accept-encoding` lists encodings the server **does** accept; MUST NOT include `zstd` |
| Same, on a client receiving the response | client | `INTERNAL` | |
| Receiver has zstd, frame is malformed / leftover bytes / dictionary / skippable | server | `INTERNAL` (or `UNIMPLEMENTED` if treated as unknown) | prefer `INTERNAL` once zstd is advertised |
| Same, on a client | client | `INTERNAL` | |
| Decompressed size > max-receive-message-size | receiver | `RESOURCE_EXHAUSTED` recommended; `INTERNAL` acceptable | see Safety |
| Server wants zstd, client did not list `zstd` in `grpc-accept-encoding` | — | — | server MUST send uncompressed (or another listed coding) |
| Next-message uncompressed requested | — | — | Compressed-Flag MUST be 0 |

A peer MAY omit encodings from `grpc-accept-encoding`. If it later accepts a
message in an omitted-but-supported encoding, it MUST include that encoding
on the response `grpc-accept-encoding`.

Implementations MAY skip compressing a given message and send Compressed-Flag
0 even when the call encoding is `zstd`.

### Defaults and compression levels

* Channel/call default algorithm remains **no compression**.
* Sending zstd is **opt-in** via the existing per-channel / per-call API.
* After the experimental period, an implementation that includes zstd MUST
  advertise `zstd` and MUST decode it. Encode stays opt-in.

C-core compression **levels** (`NONE` / `LOW` / `MED` / `HIGH`) currently
map onto gzip/deflate based on what the peer advertised. **v1 does not
change that mapping.** Silently remapping `HIGH` onto zstd would change the
wire encoding for servers that set a level and never named an algorithm.

| API | v1 behavior |
| --- | --- |
| Explicit algorithm `ZSTD` / `"zstd"` | encode zstd (if peer accepts it) |
| Explicit algorithm gzip/deflate/none | unchanged |
| Level only, no algorithm | still gzip/deflate as today |
| No algorithm, no level | identity (unchanged) |

A later gRFC may remap levels onto zstd once it is ubiquitous.

### Safety

Decompressed size MUST be bounded by the call’s max-receive-message-size
(implementation default if unset). Implementations MUST stop inflating and
fail the RPC when that bound would be exceeded.

Implementations SHOULD reject a frame whose promised window size exceeds
max-receive-message-size (or an implementation cap, whichever is smaller)
before allocating that window.

This is the same bomb model gzip already needs. zstd’s higher ratio makes
the bound **more** important, not less.

CRIME/BEAST: per-message disable is unchanged. Shared-window full-stream
compression is a non-goal partly because it would worsen those attacks.

### C-core, C++, wrapped languages

Add `GRPC_COMPRESS_ZSTD` immediately before `GRPC_COMPRESS_ALGORITHMS_COUNT`
so existing bitset values for NONE / DEFLATE / GZIP stay stable:

```c
typedef enum {
  GRPC_COMPRESS_NONE = 0,     /* bit 0 */
  GRPC_COMPRESS_DEFLATE,      /* bit 1 */
  GRPC_COMPRESS_GZIP,         /* bit 2 */
  GRPC_COMPRESS_ZSTD,         /* bit 3 */
  GRPC_COMPRESS_ALGORITHMS_COUNT
} grpc_compression_algorithm;
```

On-wire name for this enum value is `zstd`.

C-core SHOULD vendor a stable **release** of libzstd (not an arbitrary
commit) and SHOULD allow cmake/pkg-config to use a system libzstd, same
pattern as zlib. New-dependency cost is acknowledged; zlib is already
required, and zstd is comparably packaged.

Default enabled-algorithm bitset includes zstd once the codec is
non-experimental. Disable with the existing API:

```cpp
builder.SetCompressionAlgorithmSupportStatus(GRPC_COMPRESS_ZSTD, false);
```

or the channel-arg bitset
`GRPC_COMPRESSION_CHANNEL_ENABLED_ALGORITHMS_BITSET`.

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
server = grpc.server(executor, compression=grpc.Compression.Zstd)
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
in an optional artifact (for example `grpc-zstd`) that registers at runtime
(service loader and/or default `DecompressorRegistry`). Applications that
depend on the artifact get encode + decode; others behave as today.

A pure-Java decoder/encoder is acceptable if it interops with libzstd frames
from C-core and Go. JNI to libzstd is acceptable inside the optional artifact.

`Codec.Gzip` remains the only compressor in the core artifact (Java does not
ship deflate today; this gRFC does not change that).

```java
// after depending on the zstd artifact
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

Same wire name and frame profile. Node, C# (`Grpc.Net.Client` /
`Grpc.AspNetCore`), Dart, and others implement after C-core ↔ Java ↔ Go
interop exists. Same calendar release is not required.

C# already maps `CompressionLevel` onto gzip. v1 SHOULD add an explicit
algorithm/name (`"zstd"`) rather than silently retargeting `CompressionLevel`.

### Proxies and existing custom codecs

Transparent HTTP/2 proxies that do not inspect message payloads are
unaffected. DATA frames are opaque.

Proxies, gateways, Envoy filters, or transcoders that **decompress** gRPC
messages MUST learn `zstd` or they will fail the same way they fail unknown
encodings today. That work is outside this gRFC; the name and profile here
are what they would implement.

grpc-web / JSON transcoding typically does not use `grpc-encoding` zstd
today. No change required until those stacks opt in.

Applications that already registered a custom compressor under the name
`zstd` in Java or Go MUST match this frame profile or use a different private
name. First-class `zstd` owns the name. Private experiments SHOULD have been
using a non-IANA name; if they used `zstd` anyway, they need a one-time
compat check against libzstd frames.

### Observability

[A66](A66-otel-stats.md) already records compressed message sizes
(`sent_total_compressed_message_size` /
`rcvd_total_compressed_message_size`). Those metrics stay. zstd SHOULD make
the compressed histogram smaller for the same uncompressed payload; no new
metric is required.

Binary logging ([A16](A16-binary-logging.md)) SHOULD continue to record
uncompressed application messages. Logging compressed bytes is optional and
out of scope.

Channelz does not need a new object; the existing compression algorithm on
the call is enough if surfaced at all.

### Interop tests

Extend existing compression interop (compression.md test cases) with `zstd`.
Minimum matrix: C-core, Java, Go as both client and server.

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
9. Skippable-frame-only payload fails.
10. Frame with checksum and frame without checksum both succeed.
11. Asymmetry: gzip request / zstd response, and the reverse.
12. Streaming: several messages, mix of Compressed-Flag 0 and 1, each zstd
    message independently decodable.
13. C-core ↔ Java ↔ Go, all encode/decode combinations.

### Documentation

* `doc/compression.md` — list `zstd`; point at this gRFC for the frame
  profile.
* `doc/PROTOCOL-HTTP2.md` — add `"zstd"` to `Content-Coding`.
* Language API docs and compression examples (C++, Python, Java, Go).
* Interop test instructions in `tools/run_tests` / language equivalents.

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

| Codec | Role | Why not first-class in this gRFC |
| --- | --- | --- |
| gzip | portable today | keep; too slow to be the only option |
| deflate | C-core only | Java already skipped it; do not expand |
| snappy | reserved name, some Java forks | zstd typically dominates ratio at similar CPU; one new codec |
| brotli | HTTP static assets | slow compress; wrong for per-RPC CPU |
| zstd | this gRFC | RFC + IANA + #26460 + HTTP name reuse for the **frame**, not the header |

HTTP clients adding Brotli/zstd `Content-Encoding` is not evidence that gRPC
should add Brotli. HTTP compresses a few large documents; gRPC compresses
many small messages on the RPC path.

### Why a built-in codec, not a C-core plugin API

Java and Go already have experimental registries; C-core does not. A
cross-language BYO API has been declined (abuse as a generic transform;
wrapped-language surface). A single named codec matches gzip and is what
#26460 needs.

### Why not full-stream compression

HTTP-style compression across messages cannot use the current per-message
Compressed-Flag model. It also shares a window across secrets and
attacker-controlled bytes (CRIME). Separate design, if ever.

### Why levels do not silently switch to zstd

Servers using `GRPC_COMPRESS_LEVEL_*` would start emitting a coding older
clients do not understand. Explicit algorithm selection is the compatible
path. Same reason Java should not retarget `CompressionLevel` in v1.

### Why dictionaries are forbidden

A shared dictionary needs distribution, versioning, and a name in metadata.
Independent frames interop without it. A later gRFC can add dictionaries if
someone shows a fleet that needs them.

### Why an optional Java artifact

grpc-java already omits deflate to keep the core artifact small and free of
extra native deps. zstd should follow gzip’s *wire* status without forcing
JNI onto every user. Go’s blank-import package is the same idea.

### Why default remains identity

Turning on send-zstd by default would break mixed fleets the same way
default-gzip would. Decode+advertise is safe: old clients ignore unknown
entries in `grpc-accept-encoding`. Encode is the breaking direction.

## Implementation

1. This gRFC — comment period (≥ 10 business days), then merge. C-core, Java,
   and Go language owners should review (the original #26460 thread:
   markdroth, ejona86, dfawley).
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

Suggested soak: interop tests green for C-core ↔ Java ↔ Go, then at least
one minor release with the env var / optional artifact before decode is
unconditional in C-core.

## Compatibility and rollout

| Client | Server | Client sends zstd | Result |
| --- | --- | --- | --- |
| old | old | no | unchanged |
| old | new (decode+advertise) | no | unchanged; server may list `zstd` in accept; old client ignores it |
| new (encode) | old | yes | `UNIMPLEMENTED`; client must handle or not send |
| new (encode) | new | yes | success |
| new (decode only) | new (encode) | no | server may send zstd because client advertised it |

A client MUST NOT send zstd unless it is willing to handle `UNIMPLEMENTED`
from older servers (same as gzip). Enabling send-zstd as a **channel
default** on a mixed fleet is an operator choice, not this gRFC’s default.

Partial language rollout is expected: C-core may land first. Until Java and
Go decode zstd, C-core clients MUST treat missing `zstd` in
`grpc-accept-encoding` as “do not send zstd” — which is already the
compression.md rule.

## Open issues

1. **Java artifact vs core.** Optional `grpc-zstd` is the recommendation. A
   pure-Java codec in core also satisfies this gRFC if the name is `zstd`
   and frames interop.
2. **Go encoder library.** klauspost/compress vs waiting for a stdlib
   encoder. Interop with libzstd is the acceptance test.
3. **libzstd version.** Pin a stable release in C-core `third_party` at
   implementation time. This gRFC does not freeze the version.
4. **Level remapping.** Later update, once zstd is widely deployed.
5. **Encoder checksum default.** MAY include xxHash-64. Implementations
   should pick one default and stick to it; either is interoperable.
)
