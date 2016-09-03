# SMOL-V: like Vulkan/Khronos SPIR-V, but smaller.

## Overview

SMOL-V encodes Vulkan/Khronos [SPIR-V](https://www.khronos.org/registry/spir-v/)
format programs into a form that is *smoller*, and is more
compressible. Normally no changes to the programs are done; they decode
into exactly same program as what was encoded. Optionally, debug information
can be removed too.

SPIR-V is a very verbose format, several times larger than same programs expressed in other
shader formats *(e.g. DX11 bytecode, GLSL, DX9 bytecode etc.)*. The SSA-form with ever increasing
IDs is not very appreciated by regular data compressors either. SMOL-V does several things
to improve this:

- Many words, especially ones that most often have small values, are encoded using
  ["varint" scheme](https://developers.google.com/protocol-buffers/docs/encoding) (1-5 bytes per
  word, with just one byte for values in 0..127 range).
- Some IDs used in the program are delta-encoded, relative to previously seen IDs (e.g. Result
  IDs). Often instructions reference things that were computed just before, so this results in
  small deltas. These values are also encoded using "varint" scheme.
- Reordering instruction opcodes so that the most common ones are the smallest values, for smaller
  varint encoding.
- Encoding several instructions in a more compact form, e.g. the "typical <=4 component swizzle"
  shape of a VectorShuffle instruction, or sequences of MemberDecorate instructions.

A somewhat similar utility is [spirv-remap from glslang](https://github.com/KhronosGroup/glslang/blob/master/README-spirv-remap.txt).

See [this blog post](http://aras-p.info/blog/2016/09/01/SPIR-V-Compression/) for more information about
how I did SMOL-V.


## Usage

Add [`source/smolv.h`](source/smolv.h) and [`source/smolv.cpp`](source/smolv.cpp) to your C++ project build.
It might require C++11 or somesuch; I only tested with Visual Studio 2010, 2015 and Mac Xcode 7.3.

`smolv::Encode` and `smolv::Decode` is the basic functionality. See [smolv.h](source/smolv.h).

Other functions are for development/statistics purposes, to figure out frequencies and
distributions of the instructions.

There's a test + compression benchmarking suite in `testing/testmain.cpp`, using that needs adding
other files under testing/external to the build too (3rd party code: glslang remapper, Zstd, LZ4, miniz).

## Changelog

See [**Changelog**](Changelog.md).


## Limitations / TODO

- SPIR-V where the words got stored in big-endian layout is not supported yet.
- The whole thing might not work on Big-Endian CPUs. It might, but I'm not 100% sure.
- Not much prevention is done against malformed/corrupted inputs, TODO.
- Out of memory cases are not handled. The code will either throw exception
  or crash, depending on your compilation flags.


## License

Code itself: **Public Domain**.

There is 3rd party code under the testing framework (`testing/external`); it is not required for
using SMOL-V. Most of that code ([glslang](https://github.com/KhronosGroup/glslang),
[LZ4](https://github.com/Cyan4973/lz4), [Zstd](https://github.com/facebook/zstd)) is BSD-licensed,
and taken from github repositories of the respective projects. [miniz](https://github.com/richgel999/miniz)
is public domain.

There are SPIR-V binary shader dumps under `tests/spirv-dumps` for compression testing;
these are not required for using SMOL-V. Not sure how to appropriately
"license" them (but hey they are kinda useless by themselves out of context),
so I'll go with this: "Binary shader dumps under 'tests' folder are only to be
used for SMOL-V testing". Details on them:

* `tests/spirv-dumps/dota2` - some shaders from [DOTA2](http://blog.dota2.com/), Copyright Valve Corporation, all rights reserved.
* `tests/spirv-dumps/talos` - some shaders from [The Talos Principle](http://www.croteam.com/talosprinciple/),
  Copyright (c) 2002-2016 Croteam All rights reserved.
* `tests/spirv-dumps/unity` - various [Unity](https://unity3d.com/) shaders, produced
  through a HLSL -> DX11 bytecode -> HLSLcc -> glslang toolchain.



## Results

As of 2016 September 1, results on 323 shaders (under `tests/spirv-dumps`) are:

```
Original size: 3725.4KB
0 Remap       3560.0KB  95.6%
0 SMOL-V      1302.5KB  35.0%
1    LZ4HC     884.4KB  23.7%
1 re+LZ4HC     743.3KB  20.0%
1 sm+LZ4HC     441.6KB  11.9%
2    Zstd      555.4KB  14.9%
2 re+Zstd      425.6KB  11.4%
2 sm+Zstd      297.0KB   8.0%
3    Zstd20    339.4KB   9.1%
3 re+Zstd20    260.5KB   7.0%
3 sm+Zstd20    227.0KB   6.1%
```

* "Remap" is spirv-remap from glslang, with debug info stripping.
* SMOL-V is what you're looking at, with debug info stripping too.
* LZ4HC and Zstd are general compression algorithms at default settings (Zstd20 is Zstd compression with almost max setting of 20).
* "re+" is "remapper + compression", "sm+" is "SMOL-V + compression".
* Compression is done on the whole blob of all the test programs (not individually for each program).
