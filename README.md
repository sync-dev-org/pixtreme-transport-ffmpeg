# pixtreme-transport-ffmpeg

Read video files with FFmpeg software decoders and receive independently owned
GPU `pixtreme.PixelBuffer` values. Decoding runs on the CPU, including for codecs
that NVDEC cannot decode, such as ProRes. The reader preserves the decoder's
sample layout, bit depth, colour declarations, display order and exact
presentation times without colour conversion.

The same reader is available through Python, a C ABI and a header-only C++
wrapper. A CUDA-capable NVIDIA GPU and a driver compatible with pixtreme are
required to receive pictures; opening a file and inspecting its description do
not initialise the GPU.

## Installation

Install the Python package from [PyPI](https://pypi.org/project/pixtreme-transport-ffmpeg/):

```sh
python -m pip install pixtreme-transport-ffmpeg
```

The package requires CPython 3.12 or newer and `pixtreme>=2.28.1`. Wheels use the
CPython stable ABI (`abi3`). Linux x86_64 wheels require glibc 2.28 or newer.
Check the assets in the [Release for your version](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/releases)
for available platforms. Windows x86_64 artifacts are added only for versions
that have passed native Windows build and runtime checks.

For C and C++, obtain the SDK archive from the same Release. The SDK contains
shared libraries, public headers and a CMake package. A Conan 2 recipe named
`pixtreme-transport-ffmpeg`, which downloads this SDK, is distributed separately
to the publisher's organisation projects. The pixtreme SDK, version 2.28.1 or newer, is also required. When installing
archives manually, make both SDKs discoverable to CMake and their shared
libraries discoverable to the operating system's library loader.

FFmpeg shared libraries are included in the wheel and SDK. Installing a separate
FFmpeg executable is unnecessary for using the reader.

## Python

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

## Timing, colour and ownership

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

The CPU-to-GPU copy completes before `read()` returns. Later reads cannot change
previously returned pixel values. Follow pixtreme's GPU use-record and lifetime
rules when processing those buffers on other GPU streams.

## Codecs and pixel formats

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
| FFV1 | Planar YUV, GBR/GBRA, gray and BGRA samples at representative depths |
| PNG | 8-bit RGBA |
| Raw video | NV12, UYVY, VUYA, RGB0, RGBA, planar GBR, gray and gray-alpha |

Mapped layout families include planar YUV 4:2:0/4:2:2/4:4:4, YUVA 4:4:4,
GBR/GBRA, gray/gray-alpha, NV12/NV16, P010/P012/P016/P210/P212/P216, UYVY,
VUYA, RGBA/BGRA, RGB0/BGR0 and packed 10-bit RGB. Each layout and effective bit
depth must pass pixtreme's layout validation. The effective depth is the
decoder's output precision; ProRes alpha is not guaranteed at its full 16-bit
precision. The distribution's `FFmpeg-configuration.json` lists its actual
enabled video decoders; that list alone does not establish that a decoded pixel
format is representable.

Unsupported inputs produce a path and reason instead of an implicit conversion:

- Interlaced pictures and formats that cannot map directly to PixelBuffer.
  With pixtreme 2.28.1, examples include 9-bit planar samples, 10-bit GBR/gray,
  4:1:1/4:4:0, RGB24/RGB48, paletted, floating-point and big-endian samples.
- Colour declarations without equivalent pixtreme tokens, including limited
  range or a YCbCr matrix on RGB pictures.
- Inputs without presentation timestamps, files without video, unavailable
  decoders, unreadable files, and corrupt packets or failed decoding.

ProRes RAW, deinterlacing, audio, subtitles, timecode-track extraction and
encoding are outside this reader's API.

## FFmpeg and licenses

The reader is provided under the [MIT license](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/blob/main/LICENSE). Wheels and SDKs bundle
FFmpeg shared libraries in an LGPL configuration with GPL and nonfree components
disabled. FFmpeg's [LGPL 2.1 or later license](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/blob/main/licenses/FFmpeg-LGPL-2.1.txt) and
the [FFmpeg notice](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/blob/main/licenses/FFmpeg-NOTICE.txt) accompany this distribution.
Third-party notices also cover dav1d, zlib, liblzma, bzip2 and nanobind.

For the exact FFmpeg version, configuration, applied changes and build method,
consult `FFmpeg-NOTICE.txt`, `FFmpeg-configuration.json`, `FFmpeg-changes.diff`
and `FFmpeg-BUILD.txt` in the wheel's or SDK's license directory. The notice
identifies the corresponding patched FFmpeg source archive and its SHA256.
Download that archive from the [same version's Release](https://github.com/sync-dev-org/pixtreme-transport-ffmpeg/releases),
alongside the SDK and `SHA256SUMS`. The archive includes the matching dependency
sources and build inputs.
