# bga mesh format

This document describes the BGA (Binary GPU Attribute) file format.

BGA files describe a mesh in a binary format suitable for rendering on the GPU
with minimal processing or parsing.

file format version: 2.0
document version: 2.0.0

# example

This example describes a triangular pyramid.

The data section is shown as commented text for convenience, but in the real
file this section would be binary.

```
BGA 2.0\n
little endian\n
vec3 vertex.position\n
vec4 vertex.color\n
uint32 edge.cell[2]\n
uint32 triangle.cell[3]\n
4 vertex\n
6 edge\n
4 triangle\n
\n
0 0 0 // 3 padding bytes: 133 bytes to float-aligned 136 bytes
-1.0 -1.0 +1.0      // vertex 0 position (float32 x 3)
+1.0 +0.0 +0.0 +1.0 // vertex 0 color    (float32 x 4)
+1.0 -1.0 +1.0      // vertex 1 position (float32 x 3)
+0.0 +1.0 +0.0 +1.0 // vertex 1 color    (float32 x 4)
+0.0 -1.0 -1.0      // vertex 2 position (float32 x 3)
+0.0 +0.0 +1.0 +1.0 // vertex 2 color    (float32 x 4)
+0.0 +1.0 +0.0      // vertex 3 position (float32 x 3)
+0.0 +1.0 +1.0 +1.0 // vertex 3 color    (float32 x 4)
0 1 // edge 0 (uint32 x 2)
0 2 // edge 1 (uint32 x 2)
0 3 // edge 2 (uint32 x 2)
1 2 // edge 3 (uint32 x 2)
1 3 // edge 4 (uint32 x 2)
2 3 // edge 5 (uint32 x 2)
0 1 2 // triangle 0 (uint32 x 3)
0 1 3 // triangle 1 (uint32 x 3)
1 2 3 // triangle 2 (uint32 x 3)
2 0 3 // triangle 3 (uint32 x 3)
```

# format

The format contains a header section as text followed by a binary data payload.

## header

The header must begin with the string `"BGA "` followed by a version string as
text followed by a new line character `"\n"`.

The rest of the header contains directives in any order:

* `(little|big) endian` - declare an endianness for the binary data
* `TYPE BUFNAME.VARNAME([\d+])?` - declare a TYPE in a buffer BUFNAME called
  VARNAME. Optionally provide a `[n]` field count.
* `\d+ BUFNAME` - declare the number of records in BUFNAME
* `// COMMENT` - include a comment

Any unrecognized directives should be ignored.

A statement with a field count such as `uint32 triangle.cell[3]\n` means the
VARNAME called cell contains 3 contiguous uint32 fields.

The header MUST specify an endianess.

A "record" refers to all the attributes defined for a buffer. The order of
`TYPE` declarations in the header for each BUFNAME is the same order that each
record will be laid out in the data section.

If a buffer is alluded to by a `TYPE BUFNAME.VARNAME([\d+])?` declaration but
there is no `\d+ BUFNAME` count declaration, an implementation should use 0 for
the count.

Attribute TYPE can be any of these types:

* float
* vec2, vec3, vec4
* mat2, mat3, mat4
* uint8, uint16, uint32
* int8, int16, int32

The header ends with an empty line, just like HTTP.

## data

The data section consists of each buffer subsection in the order provided by the
`\d+ BUFNAME` declarations in the header.

The attributes for each record are stored in a strided format, where a full
record is written.

Each buffer subsection must be aligned to the least common multiple of the size
of the base type with padding bytes. Use 0x00 for each padding byte.

* The size of the base type for float, vector, and matrix types is 4.
* The size of uint8 and int8 is 1.
* The size of uint16 and int16 is 2.
* The size of uint32 and int32 is 4.

Because the size of base types is either 1, 2, or 4, to compute the least common
multiple you can use `max()` instead of a full LCM implementation.

# why

Parsing 3d assets can be very slow for large assets. This format has almost no
parse time penalty aside from the very small header. Offsets into the model
format can be given directly to the graphics hardware, assuming the endianness
matches.

Loading assets quickly is particularly important on the web, where application
sessions are brief and user patience is low.

The strided format provides for better memory locality for many use cases.

If you need your data to not be strided, you can achieve a non-strided effect
using buffers with a single defined type.

## why not [PLY][]?

[PLY][] comes close, but:

* lists are dynamic and include the size of each face inline
* attribute types are based on C, not GLSL
* non-vertex attributes are allowed
* faces are not necessarily triangles

[PLY]: http://paulbourke.net/dataformats/ply/

## why not [OBJ][]?

* pre-defined set of allowed attributes: v, vt, vn, vp
* prescribes 3d/4d vertex positions
* non-triangle faces allowed
* face attributes mean a renderer needs to split vertices

[OBJ]: https://en.wikipedia.org/wiki/Wavefront_.obj_file

## why not [glTF][]?

[glTF][] has a representation for setting buffers into a file with offset and
stride, but it has a lot of other features and assumptions that you might not
want:

* meshes are coupled to the material system
* format specifies materials, cameras, figure rigging, and animation

[glTF]: https://raw.githubusercontent.com/KhronosGroup/glTF/master/specification/2.0/figures/gltfOverview-2.0.0.png

# benchmarks

This benchmark parses public domain 3d models from https://nasa3d.arc.nasa.gov/
in a web browser.

```
Gemini.bga      3.2 ms     0.4 ms     0.5 ms     0.3 ms     0.3 ms
Gemini.json   499.1 ms   411.6 ms   408.0 ms   404.3 ms   400.3 ms
Gemini.obj   1022.9 ms  1013.4 ms  1014.5 ms   972.8 ms   978.6 ms
Mercury.bga     1.6 ms     0.3 ms     0.2 ms     0.2 ms     0.3 ms
Mercury.json  314.1 ms   309.0 ms   290.2 ms   307.2 ms   306.5 ms
Mercury.obj   771.7 ms   726.6 ms   717.5 ms   732.7 ms   740.2 ms
MKIII.bga       0.7 ms     0.3 ms     0.3 ms     0.2 ms     0.2 ms
MKIII.json    231.8 ms   178.4 ms   188.2 ms   202.3 ms   199.4 ms
MKIII.obj     595.7 ms   644.1 ms   584.1 ms   593.7 ms   599.4 ms
MMSEV.bga       0.4 ms     0.2 ms     0.7 ms     0.3 ms     0.4 ms
MMSEV.json   1487.5 ms  1584.0 ms  1829.2 ms  1417.3 ms  1439.4 ms
MMSEV.obj    2578.5 ms  2771.4 ms  2535.6 ms  2562.6 ms  2645.9 ms
Z2.bga          2.2 ms     0.3 ms     0.4 ms     0.3 ms     0.5 ms
Z2.json       108.7 ms   108.2 ms   104.9 ms   108.1 ms   112.2 ms
Z2.obj        203.8 ms   201.9 ms   202.2 ms   221.3 ms   203.9 ms
```

[source](https://git.scuttlebot.io/%25KiJRuIqofRa9G%2BT4empthx7Nue8TDolkfCzq9rHiIfc%3D.sha256/blob/ea5f47917b3d9cc4cf44689e5a7cc459356a0f25/bench/main.js)

# software libraries

javascript:

* [create-bga-mesh](https://git.scuttlebot.io/%25v9llERHzFn0rkZsXpssxo8FO2YxqSSdabrHTPxkPWm0=.sha256) (generator)
* [parse-bga-mesh](https://git.scuttlebot.io/%25KiJRuIqofRa9G+T4empthx7Nue8TDolkfCzq9rHiIfc=.sha256) (parser)

