# pixtreme-transport-ffmpeg

Demux and mux containers, decode and encode video and audio with FFmpeg software
codecs, and read video files as independently owned GPU `pixtreme.PixelBuffer`
values. The demuxer, decoders, encoders and muxer can be used separately, with
file paths, byte callbacks or caller-supplied packets as appropriate. The video
reader combines demux and decode, preserving sample layout, bit depth, colour
declarations, display order and exact presentation times without colour conversion.

All operations are available through Python, a C ABI and a header-only C++
wrapper. Codecs run on the CPU, including for codecs that NVDEC cannot decode,
such as ProRes. Video output uses GPU PixelBuffers and video encoding copies
input PixelBuffers back to the CPU. These transfers require a CUDA-capable
NVIDIA GPU and a driver compatible with pixtreme. Demux, mux, audio operations
and capability queries do not initialise a GPU; opening a reader or video
decoder and inspecting metadata also leave the GPU uninitialised.

## Installation

Install the Python package from [PyPI](https://pypi.org/project/pixtreme-transport-ffmpeg/):

```sh
python -m pip install pixtreme-transport-ffmpeg
```

The package requires CPython 3.12 or newer and `pixtreme>=2.33.0`. Wheels use the
CPython stable ABI (`abi3`). Linux x86_64 wheels require glibc 2.28 or newer.
Check the assets in the [Release for your version](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/releases)
for available platforms. Windows x86_64 artifacts are added only for versions
that have passed native Windows build and runtime checks.

For C and C++, obtain the SDK archive from the same Release. The SDK contains
shared libraries, public headers and a CMake package. A Conan 2 recipe named
`pixtreme-transport-ffmpeg`, which downloads this SDK, is distributed separately
to the publisher's organisation projects. The pixtreme SDK, version 2.33.0 or newer, is also required. When installing
archives manually, make both SDKs discoverable to CMake and their shared
libraries discoverable to the operating system's library loader.

FFmpeg shared libraries are included in the wheel and SDK. Installing a separate
FFmpeg executable is unnecessary for using the library.

## Python reader

```python
from fractions import Fraction
from pixtreme.transport import ffmpeg

with ffmpeg.open("clip.mov", device=0) as reader:
    info = reader.stream_info
    print(info.codec, info.width, info.height, info.pixel_format, info.bit_depth)
    reader.seek(Fraction(5, 1))
    while (picture := reader.read()) is not None:
        pixels = picture.pixels  # pixtreme.PixelBuffer with its own GPU storage
        print(picture.presentation_time, picture.duration)
```

`ffmpeg.Reader(path, device=0)` also opens a file directly. Paths accept strings
and `os.PathLike` values. `device` selects a CUDA device ordinal after any
`CUDA_VISIBLE_DEVICES` selection.

`stream_info` describes the first decoded picture: FFmpeg's codec name, width,
height, pixtreme pixel-format token, effective bit depth and `frame_period`.
The frame period is a `fractions.Fraction` in seconds, or `None` if unknown.
`read()` returns a `Picture`, or `None` at the end of the stream. Each picture
has a `PixelBuffer`, a `presentation_time` in `Fraction` seconds and a `duration`
in `Fraction` seconds or `None`.

`ReaderError` exposes the input `path` and an `ErrorReason` in `reason`, such as
`OPEN_FAILED`, `NO_VIDEO_STREAM`, `UNSUPPORTED_CODEC`, `UNSUPPORTED_PIXEL_FORMAT`,
`INTERLACED` or `UNSUPPORTED_COLOUR`. Packet and decoding failures encountered
later in the stream also raise errors. Opening diagnoses the first picture
before returning a reader.

The context manager closes the reader even if the body raises an exception.
`close()` can be repeated. Returned PixelBuffers stay valid after later reads
and after closing the reader. Reads, seeks and metadata queries on a closed
reader fail. Calls on one reader must not overlap.

## Demux, decode, encode and mux

These operations are independent of the reader and do not require
`pixtreme.transport` core. Media operations run synchronously on the calling
thread, without an application packet queue or worker thread. Calls on one
instance must not overlap, except that a demuxer's or muxer's `interrupt()` may
run from another thread. Finish those calls before destroying the instance.
A stop request is terminal for that instance.

### Packets, streams and demux

`Demuxer(path)` or `Demuxer(read, seek=seek)` probes a container and exposes
`streams`, a list of owned `StreamDescription` snapshots. `read()` returns an
owned `Packet`, or `None` at EOF. Demux does not require a supported decoder.
The packet carries encoded `payload` bytes, `stream_index`, flags, side data
and optional integer `pts`, `dts` and `duration`; absent timestamps are `None`.
The original `time_base_numerator` and `time_base_denominator` integers retain
unknown clock values. `time_base` provides a `Fraction` for valid clocks.
Stream snapshots
include `codec`, `kind`, codec `configuration`, time base, video/audio parameters
and `packet_format` (`length_prefixed`, `annex_b`, `native` or `unknown`). Demux
preserves these values without filling timestamps. By default, it returns the
original packet bytes and configuration.

Stream descriptions also expose `container_color_range`,
`container_color_primaries`, `container_color_transfer`, `container_color_space`
and `container_chroma_location` in C, C++ `info()` and Python. These are the
declarations read from MOV/MP4 `colr` or Matroska Colour headers before FFmpeg
probes the stream. Missing declarations and other containers return FFmpeg's
unspecified codes: 2 for primaries, transfer and space; 0 for range and chroma
location. The original `color_*` and `chroma_location` fields keep their probed
values.

Reader and video decoder copy each colour field reported by the decoder to the
`PixelBuffer`. For fields the decoder leaves unspecified, a demuxed description
supplies its container declaration first, then its original probed value.
Container colours are not passed to the decoder. For caller-created
descriptions, pass colours only in the original fields; the container fields
are outputs and remain unspecified. The C creation function also accepts the
original record size through `side_data_count`.

Pass `annex_b=True` to either Python constructor to convert every length-prefixed
H.264/HEVC/VVC stream with FFmpeg's `h264_mp4toannexb` / `hevc_mp4toannexb` /
`vvc_mp4toannexb` filter.
Each returned packet still represents one access unit and retains its stream
index, PTS, DTS, duration, time base and flags. The stream description pairs the
filter's Annex B configuration with `packet_format="annex_b"` and
`nal_length_size=0`. Already Annex B streams and other codecs pass through
unchanged. Unknown framing or conversion failure raises `ReaderError` with
`DEMUX_FAILED`. Packets and configuration retain their usual independent
lifetimes. Seeking resets the filter's state.

C++ uses `DemuxerConfig config; config.annex_b = true;` with
`Demuxer(path, config)` or `Demuxer(read, seek, config)` (use `{}` for an absent
seek callback). C uses `pixtreme_transport_ffmpeg_demuxer_config`, initialized
with `PIXTREME_TRANSPORT_FFMPEG_DEMUXER_CONFIG_INIT`, and
`demuxer_open_with_config` / `demuxer_open_io_with_config`. Set `annex_b` to a
nonzero value to request conversion; a null config selects defaults. The
existing C open functions and C++ constructors keep their original defaults.

For external encoded input, construct `Packet(payload, ...)` and
`StreamDescription(codec, kind=..., ...)` with the same information instead of
using demux. Their constructors copy configuration, payload and side data.
Packets and descriptions remain valid after later reads, seeks and closing the
demuxer. Python byte properties return owned `bytes`; C++ `info()` and C
`*_get_info` pointers borrow from their packet or description until its last
owner is destroyed.

A Python read callback accepts a maximum byte count and returns at most that
many `bytes`; `b""` means permanent EOF. An optional seek callback takes
`(offset, whence)` and returns the absolute byte position. Whence is 0, 1 or 2,
or `ffmpeg.IO_SIZE` to query size without moving (return -1 if unknown).
`demux.seek(stream_index, timestamp, backward=True)` uses that stream's integer
ticks and seeks to a preceding keyframe by default. Without a path or seek
callback, seeking raises `NOT_SEEKABLE`; the demuxer determines the exact landing
packet. Flush a separate decoder after seeking before submitting new packets.

Python callback exceptions become `ReaderError` with `IO_FAILED`;
`InterruptedError` instead requests a stop and reports `INTERRUPTED`, including
during probing. Callbacks are retained until the demuxer or muxer is destroyed;
keep captured resources alive through destruction. No callback runs after close.
`interrupt()` cannot preempt a blocked callback: the caller must wake it and
make it return on stop. C callbacks return byte counts, 0 for read EOF,
`PIXTREME_TRANSPORT_FFMPEG_IO_INTERRUPTED` for stop, or another negative value for
failure, and must not throw. Keep their context valid through destruction.
C++ callables use the same return values; exceptions become `IO_FAILED`.

### Video and audio decode

`VideoDecoder(stream, device=0)` accepts packets for the configured stream index
and the same unreduced time-base integers. `receive()` returns a
`DecodedPicture` with owned GPU `pixels`, the decoder's `codec`, optional integer
`pts`/`duration` and the original time base. Unlike the reader's picture times,
these are stream ticks and can remain unknown. Software decoding runs on the
CPU; `receive()` transfers samples to the GPU. The transfer completes before
return, and pixels survive decoder reuse, flush and close.

`receive_info()` consumes the next picture without initializing a GPU or checking
whether its pixel format, interlace or colour can be represented by `PixelBuffer`.
It returns the same `DecodedPicture` type with `pixels=None`. Both receive
operations consume the same display-order sequence: alternating them never
returns a picture twice. `receive()` always supplies a `PixelBuffer`.

Both operations return these optional picture-report fields (Python uses `None`,
C++ uses `std::optional`, and C has a separate `<name>_known` bit for each value):

| Fields | Meaning |
|---|---|
| `picture_type` | FFmpeg picture type: I=1, P=2, B=3, S=4, SI=5, SP=6, BI=7; NONE=0 is unknown |
| `key_frame`, `interlaced`, `top_field_first` | The decoder's picture flags, as booleans |
| `repeat_pict` | Additional fields repeated for display |
| `coded_width`, `coded_height` | This picture's dimensions before cropping |
| `crop_top`, `crop_bottom`, `crop_left`, `crop_right` | Crop on each edge, in pixels; GPU pictures retain the cropped dimensions and samples |
| `color_range`, `color_primaries`, `color_transfer`, `color_space`, `chroma_location` | FFmpeg colour codes from the decoded picture, without container fallback; unspecified codes are unknown |
| `packet_index` | The originating packet's zero-based decode-order number |
| `pts`, `duration` | The originating packet's optional ticks, in the existing time base fields |

These values are the decoder's report, which can differ from the bitstream's
declarations. Decoder defaults can appear in the colour fields. In particular,
HEVC interlace flags can remain set on progressive pictures after an earlier
field stream. The `interlaced` and `top_field_first` fields preserve that decoder
report. Each value belongs to that picture; values from
a later decoder context or an earlier picture are never copied into the report.
Container fallback applies only to the GPU `PixelBuffer` colour declarations.

HEVC pictures additionally expose the originating access unit's bitstream
syntax, with the same definitions as pixtreme-transport-nvdec 0.14.0:

| Field | Bitstream declaration |
|---|---|
| `nal_unit_type` | The picture's slice segment NAL unit type |
| `slice_type` | The first slice segment's HEVC type: 0 = B, 1 = P, 2 = I |
| `general_profile_idc`, `general_profile_compatibility_flags` | The referenced SPS profile; compatibility flag j occupies bit j |
| `vui_timing` | The unmodified `(num_units_in_tick, time_scale)` integers, when VUI timing is present |
| `field_seq_flag` | The referenced SPS VUI flag; false when that SPS has no VUI |
| `pic_struct` | Present only when this picture's timing SEI signals it |
| `vui_colour_primaries`, `vui_transfer_characteristics`, `vui_matrix_coeffs` | H.265 colour codes, present only with `colour_description_present_flag` |
| `vui_video_full_range_flag` | Present only with `video_signal_type_present_flag`; false means limited range |
| `vui_chroma_sample_loc_type` | The declared 4:2:0 chroma location (0–5); bottom for a field picture with `pic_struct` 2, 10 or 12, otherwise top |

The `vui_*` values are bitstream declarations; the existing `color_*` values
remain the decoder's report. Missing declarations and values that cannot be
read from damaged input are `None`, independently for each field; HEVC syntax
fields are all `None` for other codecs. An unreadable VUI does not stop decoding
or retain the preceding SPS's declarations. SPS/PPS references and the packet's
record preserve each picture's values across reordered output and SPS changes.

HEVC upload and reader interlace checks use that picture's `field_seq_flag`
and `pic_struct`, rather than the decoder report: a true field flag or a
`pic_struct` of 1–6 or 9–12 is interlaced. Unavailable syntax is left unknown
and does not inherit an earlier picture's interlace flag. Thus progressive
pictures remain uploadable after a field sequence. Other codecs retain their
FFmpeg frame-flag check. CPU `receive_info()` accepts both kinds of picture.

`packet_index` starts at zero when the decoder is created. A `send()` returning
success or raising `DECODE_FAILED` consumes one number. `AGAIN` (Python `False`)
consumes none; receive output and retry the same packet. `INVALID_ARGUMENT` and
`INVALID_STATE`, drain and flush consume none, and flush does not reset the
counter. PTS is never used to match packets to pictures: it can be missing or
repeated. If several packets produce one picture, FFmpeg associates the first
packet. If FFmpeg supplies no association, `packet_index` is unknown.

Video decoder errors permit the following handling on the same handle:

| Result or reason | Handling |
|---|---|
| `AGAIN`, `END` | Existing flow-control rules; handle state is unchanged |
| `INVALID_ARGUMENT`, `INVALID_STATE` | Correct the call and continue; handle state is unchanged |
| `INTERLACED`, `UNSUPPORTED_PIXEL_FORMAT`, `UNSUPPORTED_COLOUR` | Only that picture is consumed; receive subsequent pictures |
| `DECODE_FAILED` | The library leaves the handle open and accepts subsequent operations under the normal send/receive/drain/flush/close rules; do not resend the failed packet |
| `OUT_OF_MEMORY`, `GPU_FAILED`, any other reason | Recreate the decoder; subsequent behavior is not promised |

After `DECODE_FAILED`, the caller chooses whether to continue, flush and restart
from a key packet, or recreate the decoder. The library does not automatically
flush or resend input. Following damaged or missing HEVC/H.264 packets, correct
output is promised only from a subsequent undamaged HEVC IRAP (including IDR/CRA)
or H.264 IDR with the necessary parameter sets, followed by undamaged input.
Other codecs and causes have no recovery guarantee. Decoders can conceal damage
without an error, so absence of an error does not prove correct samples.
Drain and receive pending pictures before flush to keep them: flush discards
pending output. Begin with a key packet after construction or flush.

`AudioDecoder(stream)` accepts the same packet type and returns `Audio` values:
`pcm` is owned interleaved 48000 Hz stereo signed 16-bit little-endian bytes,
and `presentation_time` is the first sample's position in exact `Fraction`
seconds. Audio decoding requires a positive stream time base and uses no GPU.
Both decoder constructors copy the stream description; `send()` borrows its
packet only for the call. Empty packet payloads are rejected; use `drain()` for
end of input.

For example, demux a file and decode its first video stream. This requires a
GPU when pictures are received:

```python
from pixtreme.transport import ffmpeg


def receive_available(decoder):
    while True:
        output = decoder.receive()
        if output in (ffmpeg.DecodeStatus.AGAIN, ffmpeg.DecodeStatus.END):
            return output
        print(output.codec, output.pts, output.time_base, output.pixels.format)


with ffmpeg.Demuxer("clip.mov") as demux:
    stream = next(s for s in demux.streams if s.kind == "video")
    with ffmpeg.VideoDecoder(stream, device=0) as decoder:
        while (packet := demux.read()) is not None:
            if packet.stream_index != stream.index:
                continue
            while not decoder.send(packet):
                receive_available(decoder)
            receive_available(decoder)
        while not decoder.drain():
            receive_available(decoder)
        assert receive_available(decoder) == ffmpeg.DecodeStatus.END
```

### Video and audio encode

`default_encoder(kind)` returns `"libopenh264"` for `"video"` and `"aac"` for
`"audio"`. Unknown kinds raise `ReaderError` with `INVALID_ARGUMENT`.
`default_pixel_format(codec)` returns the pixtreme format token and bit depth
`("yuv420p", 8)` for `"h264"` and `"hevc"`. Other codecs raise
`UNSUPPORTED_PIXEL_FORMAT`: the caller must choose their pixel format explicitly.
Use codec names from `EncoderInfo.codec` or `StreamDescription.codec`, rather
than encoder implementation names, for this query. These queries define the
defaults independently of the muxer and do not open an encoder or initialize a
GPU. Creating an encoder with the returned name checks availability; a missing
encoder fails without selecting another implementation.

The caller converts pixels to the returned target with pixtreme's public
conversion API, such as `pixtreme.io.to_pixel_buffer(frame, format=format,
bit_depth=depth)` or `pixtreme.io.repack_pixel_buffer(pixels, format=format,
bit_depth=depth)`. Pass that format and depth explicitly to `VideoEncoder`.
The encoder takes an explicit name and performs no pixel conversion.

`VideoEncoder(name, time_base=Fraction(...), format=..., width=..., height=...,
...)` takes an exact pixtreme PixelBuffer description, including bit depth,
alignment and colour declarations when specified. `send(pixels, pts)` requires
matching layout, dimensions and colour; `pts` is an integer in the configured
time base. The GPU-to-CPU copy finishes before `send()` returns, even when it
returns `False`, so input storage is reusable. Encoding performs no colour or
layout conversion. `frame_rate=Fraction(...)` is a separate optional declaration,
not an inference from the timestamp clock.

`AudioEncoder("libopus", time_base=Fraction(1, 48000))` accepts the same PCM
format that `AudioDecoder` returns. `send(pcm, presentation_time)` takes exact
`Fraction` or integer seconds for the first sample and copies accepted bytes.
Times round to the nearest sample, with ties away from zero; accepted chunks
must be contiguous, without gaps or overlaps. Audio encoding uses no GPU and
requires the 1/48000-second packet clock; rescaling belongs to mux or the caller.

Both encoders return owned `Packet` values from `receive()` and owned snapshots
from `stream_description()`, independent of encoder lifetime. Preserve packet
side data, including codec configuration updates and audio skip/discard samples.
Take a fresh stream snapshot after drain when final padding information is
needed. Input options are `options=[("key", "value"), ...]` FFmpeg AVOptions.
An option whose value FFmpeg cannot parse, or a key the encoder does not consume,
raises `ReaderError` with `UNSUPPORTED_ENCODER_SETTING`. Its message identifies
the option name and reason; neither the message nor other error fields include
the option value. The library silences its bundled FFmpeg's logs for reader,
demux, decode, encode, mux and capability queries, with no public log controls.
Codec threads default to one; the `threads` option changes that count. Explicit
encoder names take precedence over codec names:
`libopenh264` selects H.264, `libopus` selects Opus and `libaom-av1` selects AV1.
In particular, `opus` names FFmpeg's experimental native encoder. An unsupported
encoder or setting fails without selecting another implementation.

### Codec flow control and drain

Decoder and encoder `send()` and `drain()` return `True` when accepted, or
`False` when output must be received before retrying the unchanged input.
`receive()` returns a value, `DecodeStatus.AGAIN` / `EncodeStatus.AGAIN` when
more input is needed, or `END` from the corresponding enum after drain has
exhausted the codec and any resampler. One input can yield zero, one or several
outputs. At EOF, retry `drain()` as needed and receive through `END` before
closing; an empty packet or PCM chunk is not a drain request.

Accepted drain is idempotent and rejects further input. Decoder `flush()`
discards buffered output and allows input again; an encoder must be recreated
to restart. `close()` is idempotent and releases resources; context managers
and C++ destructors close handles but do not substitute for explicit drain.

### Mux

`Muxer(path, format="mp4")` or `Muxer(write, format="matroska", seek=seek)`
accepts encoded stream descriptions and packets from demux, an encoder or an
external source. A path extension can infer the format; a write callback needs
an explicit FFmpeg muxer name. Mux uses no GPU. Call operations in this order:

1. `add_stream(description)` copies the description and returns its output
   stream index, independent of the input index. The time base must be positive.
2. `write_header()` fixes the output stream time bases.
3. `write_packet(output_index, packet)` borrows the packet for the call, without
   consuming or modifying it. It rescales PTS/DTS/duration to the output clock.
   The caller interleaves streams and supplies monotonic DTS as the format
   requires; missing timestamps are not filled.
4. `write_trailer()` finalises the container, then `close()` closes the output.

A Python write callback takes `bytes` and returns the number written. Short
writes are retried; zero and invalid counts fail with `IO_FAILED`.
Its seek and stop contracts match demux callbacks. Nonseekable destinations
require a container that supports them; ordinary MP4 output requires seek.
Header, packet and trailer failures report `HEADER_FAILED`, `PACKET_FAILED`
and `TRAILER_FAILED`; callback failures and stops retain `IO_FAILED` and
`INTERRUPTED`, with the stage in the message. A write failure is terminal.
Close/destruction abandons unfinished output without writing a trailer;
explicit close after a successful trailer reports path close failures.

Mux uses the container's built-in framing without adding a bitstream filter.
MP4 and Matroska accept H.264/HEVC Annex B parameter sets in the stream's
`configuration` together with `annex_b` packets, including NVENC output with
in-band parameter-set repetition disabled. The muxer writes the container's
global configuration and packet framing.
For `libopenh264` output to MP4 or Matroska, set
`options=[("flags", "+global_header")]` so the description supplies SPS/PPS.
For MPEG-TS, leave that flag unset so packets carry in-band SPS/PPS.
Length-prefixed H.264/HEVC/VVC packets cannot be written directly to MPEG-TS or
their raw Annex B formats (`h264`, `hevc`, `vvc`). VVC Annex B packets can be
written to MPEG-TS and raw `vvc`; M2TS rejects VVC at header writing. Matroska
may omit DTS, so the caller must supply known decode times before writing to
MPEG-TS. DTS equals PTS for this purpose only when pictures have no reordering. Raw VVC
does not store PTS/DTS, so packets demuxed from raw VVC need caller-supplied
timestamps before writing to MPEG-TS. Keep audio padding side data, but account for container
limits: AAC in MPEG-TS has no skip/discard metadata, and MP4 edit offsets may
round to the movie clock.

### Errors and C/C++ entry points

Python and C++ use `ReaderError` for library failures, including the operations
above, with `reason`, message and a path for path-based operations (empty for
callbacks and standalone codecs). Python argument conversion can also raise
`TypeError`, `ValueError` or `OverflowError`. C returns status codes and owned
errors as shown below; codec `AGAIN` and `END` are successful flow-control
statuses, not errors.

Use the same SDK and CMake targets as the reader examples below. C++ names are
in `pixtreme::transport::ffmpeg`. Header names in this table are relative to
`pixtreme/ffmpeg/`; C headers add `c/` and use `.h` instead of `.hpp`.
Every C function listed has the `pixtreme_transport_ffmpeg_` prefix; `*` in a
codec function name stands for `video` or `audio`.

| Operation | C++ header and names | Main C functions (prefix omitted) |
|---|---|---|
| Demux and encoded values | `demuxer.hpp`: `Demuxer`, `DemuxerConfig`, `Packet`, `StreamDescription` | `demuxer_open`, `demuxer_open_io`, `demuxer_open_with_config`, `demuxer_open_io_with_config`, `demuxer_stream_description`, `demuxer_read`, `demuxer_seek`; `packet_create`, `packet_get_info`, `stream_description_create`, `stream_description_get_info` |
| Decode and decoder registry | `decoder.hpp`: `VideoDecoder`, `AudioDecoder`, `DecodedPicture`, `Audio`, `DecodeStatus`, `decoders()` | `*_decoder_create`, `*_decoder_send`, `*_decoder_receive`, `video_decoder_receive_info`, `*_decoder_drain`, `*_decoder_flush`, `audio_get_info`, `decoder_count`, `decoder_get_info` |
| Encode and encoder registry | `encoder.hpp`: `VideoEncoder`, `AudioEncoder`, `EncodeStatus`, `encoders()`, `default_encoder(kind)`, `default_pixel_format(codec)` | `*_encoder_create`, `*_encoder_send`, `*_encoder_receive`, `*_encoder_drain`, `*_encoder_stream_description`, `encoder_count`, `encoder_get_info`, `default_encoder`, `default_pixel_format` |
| Mux and muxer registry | `muxer.hpp`: `Muxer`, `muxers()` | `muxer_create`, `muxer_create_io`, `muxer_add_stream`, `muxer_write_header`, `muxer_write_packet`, `muxer_write_trailer`, `muxer_count`, `muxer_get_info` |
| Reader and errors | `reader.hpp`: `Reader`; `error.hpp`: `ReaderError`, `ErrorReason` | `reader_open`, `reader_read`, `reader_seek`, `reader_close`, `error_reason`, `error_path`, `error_message`, `error_release` |

C++ video decoder `receive()` / `receive_info()` return
`std::variant<DecodedPicture, DecodeStatus>`; audio decoder `receive()` returns
`std::variant<Audio, DecodeStatus>`, and encoders return
`std::variant<Packet, EncodeStatus>`. The status members are `again` and `end`.
Operation handles are movable and close on destruction. Packet and stream
copies share ownership. C uses opaque owned handles with `*_destroy` functions;
release a returned PixelBuffer with pixtreme's destroy function. C audio bytes
borrow from the owned audio handle until `audio_destroy`. Registry strings and
format arrays borrow for the library lifetime, while C++ and Python registry
queries return owned copies. No FFmpeg header or FFmpeg object is needed.

## C++

Use C++17 or newer. Save this example as `reader.cpp`:

```cpp
#include <pixtreme/ffmpeg/reader.hpp>
#include <iostream>

int main() {
    namespace ff = pixtreme::transport::ffmpeg;
    try {
        ff::Reader reader("clip.mov", 0);
        const auto info = reader.stream_info();
        std::cout << info.codec << ' ' << info.width << 'x' << info.height << '\n';
        reader.seek({5, 1});
        while (auto picture = reader.read()) {
            const auto time = picture->presentation_time;
            std::cout << time.numerator << '/' << time.denominator << '\n';
            // picture->pixels owns GPU storage; duration is optional seconds.
        }
    } catch (const ff::ReaderError& error) {
        std::cerr << error.path() << ": reason="
                  << static_cast<unsigned>(error.reason()) << ' '
                  << error.what() << '\n';
        return 1;
    }
}
```

`Reader` closes on destruction and is movable. `read()` returns
`std::optional<Picture>`, empty at the end of the stream. Times use
`RationalSeconds {numerator, denominator}`; durations and frame periods are
optional. A picture's `pixtreme::pixel_buffer` retains its storage independently
of the reader.

## C

Save this example as `reader.c`. Every status operation can return an owned
error; release it with `pixtreme_transport_ffmpeg_error_release`. A successful
read returns an owned PixelBuffer, or a null pointer at the end of the stream.

```c
#include <pixtreme/ffmpeg/c/reader.h>
#include <inttypes.h>
#include <stdio.h>

int main(void) {
    pixtreme_transport_ffmpeg_reader *reader = NULL;
    pixtreme_transport_ffmpeg_error *error = NULL;
    pixtreme_pixel_buffer_t *pixels = NULL;
    pixtreme_transport_ffmpeg_options options = PIXTREME_TRANSPORT_FFMPEG_OPTIONS_INIT;
    pixtreme_transport_ffmpeg_stream_info info = PIXTREME_TRANSPORT_FFMPEG_STREAM_INFO_INIT;
    pixtreme_transport_ffmpeg_seconds time = PIXTREME_TRANSPORT_FFMPEG_SECONDS_INIT;
    pixtreme_transport_ffmpeg_picture picture = PIXTREME_TRANSPORT_FFMPEG_PICTURE_INIT;
    if (pixtreme_transport_ffmpeg_runtime_abi_version() != PIXTREME_TRANSPORT_FFMPEG_ABI_VERSION)
        return 2;
    options.device = 0;
    if (pixtreme_transport_ffmpeg_reader_open("clip.mov", &options, &reader, &error))
        goto failure;
    if (pixtreme_transport_ffmpeg_reader_stream_info(reader, &info, &error))
        goto failure;
    printf("%s %" PRId64 "x%" PRId64 "\n", info.codec, info.width, info.height);
    time.numerator = 5;  /* Exact seconds; the initial denominator is 1. */
    if (pixtreme_transport_ffmpeg_reader_seek(reader, &time, &error))
        goto failure;
    for (;;) {
        if (pixtreme_transport_ffmpeg_reader_read(reader, &pixels, &picture, &error))
            goto failure;
        if (!pixels)
            break;
        printf("%" PRId64 "/%" PRId64 "\n",
               picture.presentation_numerator, picture.presentation_denominator);
        pixtreme_pixel_buffer_destroy(&pixels, NULL);
    }
    pixtreme_transport_ffmpeg_reader_destroy(&reader);
    return 0;
failure:
    fprintf(stderr, "%s: reason=%" PRIu32 " %s\n",
            pixtreme_transport_ffmpeg_error_path(error),
            pixtreme_transport_ffmpeg_error_reason(error),
            pixtreme_transport_ffmpeg_error_message(error));
    pixtreme_transport_ffmpeg_error_release(error);
    pixtreme_pixel_buffer_destroy(&pixels, NULL);
    pixtreme_transport_ffmpeg_reader_destroy(&reader);
    return 1;
}
```

The `*_INIT` macros initialise the size-tagged C records. C error reasons have
the `PIXTREME_TRANSPORT_FFMPEG_` prefix; their numeric identifiers agree with
the C++ and Python interfaces. No C++ exception crosses the C ABI.

With the SDKs available to CMake, build both examples using:

```cmake
cmake_minimum_required(VERSION 3.28)
project(reader_examples LANGUAGES C CXX)
find_package(pixtreme_transport_ffmpeg CONFIG REQUIRED)
add_executable(reader_c reader.c)
target_link_libraries(reader_c PRIVATE pixtreme::transport_ffmpeg_c)
add_executable(reader_cpp reader.cpp)
target_link_libraries(reader_cpp PRIVATE pixtreme::transport_ffmpeg)
```

## Reader timing, colour and ownership

Pictures arrive in display order, including when decoding reorders B pictures.
Presentation times retain the container's timeline: a first picture at five
seconds remains at five seconds. Missing durations remain unknown; the reader
does not fill them from an estimated frame rate. The public times are exact
rational seconds, rather than container ticks or time bases.

`seek(t)` selects the picture displayed at time `t`. An exact picture boundary
selects that picture; an earlier time selects the first picture. A time at or
after the known final end selects the end of the stream. If the final duration
is unknown, times at or after the last picture's presentation time select the
last picture. Python accepts `Fraction` or integer seconds, and rejects
floating-point values. C and C++ accept a signed 64-bit numerator and a positive
signed 64-bit denominator. Python's reduced fraction must fit those limits too.

Each PixelBuffer carries the picture's colour primaries, transfer, matrix,
range and chroma location. Picture declarations take precedence over stream
declarations; a declaration absent from both remains unknown. Known declarations
without a matching pixtreme token are errors. The reader applies no colour
conversion or default colour declarations.

Primaries, transfer and matrix declarations use pixtreme's H.273 mapping.
Primaries code 1 returns `sRGB`, transfer code 1 returns `Rec.709`, and matrix
code 1 returns `BT.709`. Primaries code 10 and transfer code 17 return `CIE-XYZ`
and `DCDM`; matrix code 12 returns `native`. Encoders use the reverse mapping,
so both `Rec.709` and `sRGB` primaries signal code 1.

The CPU-to-GPU copy completes before `read()` returns. Later reads cannot change
previously returned pixel values. Follow pixtreme's GPU use-record and lifetime
rules when processing those buffers on other GPU streams.

## Codecs and pixel formats

`decoders()`, `encoders()` and `muxers()` report the actual linked registry in
Python and C++. Decoder/encoder entries carry the implementation name, codec,
media kind and declared pixel/sample formats; an empty format list means
unspecified. Muxer entries carry `name`, `long_name` and the comma-separated
`extensions` string. C exposes the corresponding `decoder_count` /
`decoder_get_info`, `encoder_count` / `encoder_get_info` and `muxer_count` /
`muxer_get_info` functions with the `pixtreme_transport_ffmpeg_` prefix.
Registry names can repeat: `matroska` has video and audio entries with different
long names and extensions. The queries preserve both entries.
These queries do not initialize a GPU. Opening and supplying input validate
the concrete configuration; registered muxers that manage their own files
cannot use the single byte output API.

The bundled FFmpeg enables software video decoders that do not require GPL or
nonfree components. The reader accepts their output whenever it maps directly
to pixtreme, regardless of whether NVDEC also supports the codec. The following
representative inputs have been verified against independently decoded FFmpeg
samples:

| Codec | Verified inputs |
|---|---|
| ProRes | 422 Proxy, LT, 422, HQ, 4444 and 4444 XQ; 4444 variants with and without alpha |
| H.264 | 8-bit 4:2:0 and 10-bit 4:2:2, including reordered B pictures |
| HEVC | Main10 4:2:0 and 8-bit 4:4:4, including reordered B pictures |
| DNxHR | HQX, 10-bit 4:2:2 |
| VP9, AV1 | 8-bit 4:2:0 |
| FFV1 | Planar YUV, including 10/12-bit 4:4:0, GBR/GBRA, gray and BGRA samples at representative depths |
| PNG | 8-bit RGBA |
| Raw video | Packed RGB/BGR/RGBA/BGRA, NV12, UYVY/YUYV, VUYA, planar YUV/YUVA and GBR, gray and gray-alpha; little- and big-endian words |

The shared reader/decoder/encoder mapping includes these pixtreme layouts:

| pixtreme layout | Effective bit depths mapped from FFmpeg |
|---|---|
| `yuv420p`, `yuv422p`, `yuv444p`, `gbrp` | 8, 9, 10, 12, 14, 16 |
| `yuva444p` | 8, 9, 10, 12, 16 |
| `gbrap` | 8, 10, 12, 14, 16 |
| `gray`; `ya` | 8, 9, 10, 12, 14, 16; 8, 16 |
| `yuv410p`, `yuv411p`; `yuv440p` | 8; 8, 10, 12 |
| `yuva420p`; `yuva422p` | 8, 9, 10, 16; 8, 9, 10, 12, 16 |
| `rgb24`, `bgr24`, `argb`, `abgr`, `0rgb`, `0bgr`, `rgba`, `bgra`, `rgb0`, `bgr0`, `uyvy422`, `yuyv422`, `vuya` | 8 |
| `rgb48`, `bgr48`, `rgba64`, `bgra64` | 16, little-endian words |
| `gray16be`, `ya16be`, `rgb48be`, `rgba64be`, `gbrp16be`, `gbrap16be` | 16, big-endian words |
| `nv12`, `nv16`; `p010`, `p210`; `p012`, `p212`; `p016`, `p216` | 8; 10; 12; 16 |
| `x2rgb10`, `x2bgr10` | 10 |

FFmpeg's `yuvj420p`, `yuvj422p`, `yuvj444p`, `yuvj411p` and `yuvj440p`
use the corresponding YUV layout and retain the declared full range. Word
samples use LSB alignment except `p010`, `p012`, `p210` and `p212`, which use
MSB alignment. Each layout and effective bit depth must pass pixtreme's layout
validation. The effective depth is the
decoder's output precision; ProRes alpha is not guaranteed at its full 16-bit
precision. The distribution's `FFmpeg-configuration.json` lists its actual
enabled video decoders; that list alone does not establish that a decoded pixel
format is representable.

The reader rejects unsupported inputs with a path and reason:

- Interlaced pictures and formats that cannot map directly to PixelBuffer.
  With pixtreme 2.33.0, examples include paletted, floating-point, Bayer, XYZ
  and hardware surface formats, and big-endian layouts without a corresponding
  pixtreme token.
- Colour declarations without equivalent pixtreme tokens, including limited
  range or a YCbCr matrix on RGB pictures.
- Inputs without presentation timestamps, files without video, unavailable
  decoders, unreadable files, and corrupt packets or failed decoding.

The reader returns video pictures only. Use the separate library operations
above for audio decoding, video/audio encoding and container demux/mux. ProRes
RAW, deinterlacing, subtitle rendering and timecode-track extraction are not
provided; demux can still expose packets and descriptions for non-video streams.

## FFmpeg and licenses

The library is provided under the [MIT license](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/blob/main/LICENSE). Wheels and SDKs bundle
FFmpeg shared libraries in an LGPL configuration with GPL and nonfree components
disabled. FFmpeg's [LGPL 2.1 or later license](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/blob/main/licenses/FFmpeg-LGPL-2.1.txt) and
the [FFmpeg notice](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/blob/main/licenses/FFmpeg-NOTICE.txt) accompany this distribution.
Third-party notices also cover dav1d, zlib, liblzma, bzip2, OpenH264, libopus, libaom (including PATENTS) and nanobind.

The external encoder libraries are built from source and statically linked
into FFmpeg:

| Encoder library | FFmpeg encoder name | Copyright license |
|---|---|---|
| OpenH264 | `libopenh264` (H.264) | BSD-2-Clause |
| libopus | `libopus` (Opus) | BSD-3-Clause |
| libaom | `libaom-av1` (AV1) | BSD-2-Clause |

Cisco's patent-fee coverage for its separately downloaded OpenH264 binary is
not assumed for this source build; see the [Cisco binary terms](https://www.openh264.org/BINARY_LICENSE.txt).
libaom's separate Alliance for Open Media patent terms accompany the
distribution as `libaom-av1/PATENTS`, alongside its `LICENSE`; the
[upstream patent license](https://aomedia.googlesource.com/aom/+/refs/tags/v3.6.1/PATENTS)
sets out its conditions. Opus copyright and patent terms are described on the
[Opus license page](https://opus-codec.org/license/). These notices do not determine
patent fees or licensing requirements for a particular use.

For the exact FFmpeg version, configuration, applied changes and build method,
consult `FFmpeg-NOTICE.txt`, `FFmpeg-configuration.json`, `FFmpeg-changes.diff`
and `FFmpeg-BUILD.txt` in the wheel's or SDK's license directory. The notice
identifies the corresponding patched FFmpeg source archive and its SHA256.
Download that archive from the [same version's Release](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/releases),
alongside the SDK and `SHA256SUMS`. The archive includes the matching dependency
sources and build inputs.
