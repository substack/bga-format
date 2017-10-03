# bga mesh format

This document describes the BGA (Binary GPU Attribute) file format.

BGA files describe a mesh in a binary format suitable for rendering on the GPU
with minimal processing or parsing.

file format version: 1.0
document version: 1.0.0

# example

This example describes a triangular prism.

The data section is shown as commented text for convenience, but in the real
file this section would be binary.

```
BGA 1.0\n
little endian\n
attribute vec3 position\n
attribute vec4 color\n
4 vertex\n
6 edge uint32\n
4 triangle uint32\n
\n
-1.0 -1.0 +1.0      // vertex 0 position (float32 x 3)
+1.0 +0.0 +0.0 +1.0 // vertex 0 color    (float32 x 4)
+1.0 -1.0 +1.0      // vertex 1 position (float32 x 3)
+0.0 +1.0 +0.0 +1.0 // vertex 1 color    (float32 x 4)
+0.0 -1.0 -1.0      // vertex 2 position (float32 x 3)
+0.0 +0.0 +1.0 +1.0 // vertex 2 color    (float32 x 4)
+0.0 +1.0 +0.0      // vertex 3 position (float32 x 3)
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
* `attribute TYPE NAME` - declare a GLSL vertex attribute NAME as TYPE
* `\d+ vertex` - declare the number of vertices
* `\d+ edge ITYPE` - declare the number of edges
* `\d+ triangle ITYPE` - declare the number of triangles
* `// COMMENT` - include a comment

Any unrecognized directives should be ignored.

The header MUST specify an endianess.

If a vertex, edge, or triangle count is omitted, use a value of 0 for the
missing count.

Attribute TYPE can be any of these GLSL types:

* float
* vec2, vec3, vec4
* mat2, mat3, mat4

ITYPE values are one of:

* uint16
* uint32

The header ends with an empty line, just like HTTP.

## data

The data format consists of vertex attributes, then edges, then triangles.

The vertex attributes are stored in a strided fashion for better memory
locality.

# why

Parsing 3d assets can be very slow for large assets. This format has almost no
parse time penalty aside from the very small header. Offsets into the model
format can be given directly to the graphics hardware, assuming the endianness
matches.

Loading assets quickly is particularly important on the web, where application
sessions are brief and user patience is low.

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
