OS-X & Ubuntu: [![Build Status](https://travis-ci.org/dvidelabs/flatcc.svg?branch=master)](https://travis-ci.org/dvidelabs/flatcc)
Windows: [![Windows Build Status](https://ci.appveyor.com/api/projects/status/github/dvidelabs/flatcc?branch=master&svg=true)](https://ci.appveyor.com/project/dvidelabs/flatcc)

# FlatCC FlatBuffers in C for C

`flatcc` has no external dependencies except for build and compiler
tools, and the C runtime library. With concurrent Ninja builds, a small
client project can build flatcc with libraries, generate schema code,
link the project and execute a test case in less than 2 seconds (4 incl.
flatcc clone), rebuild in less than 0.2 seconds and produce binaries
between 15K and 60K, read small buffers in 30ns, build FlatBuffers in
about 600ns, and with a larger executable handle optional json parsing
or printing in less than 2 us for a 10 field mixed type message.

See also experimental meson branch, and [sample client project](https://github.com/dvidelabs/flatcc-meson-sample)

NOTE: see
[CHANGELOG](https://github.com/dvidelabs/flatcc/blob/master/CHANGELOG.md).
There are occassionally minor breaking changes as API inconsistencies
are discovered. Unless clearly stated, breaking changes will not affect
the compiled runtime library, only the header files. In case of trouble,
make sure the `flatcc` tool is same version as the `include/flatcc`
path.

The project includes:

- an executable `flatcc` FlatBuffers schema compiler for C and a
  corresponding library `libflatcc.a`. The compiler generates C header
  files or a binary flatbuffers schema.
- a typeless runtime library `libflatccrt.a` for building and verifying
  flatbuffers from C. Generated builder headers depend on this library.
  It may also be useful for other language interfaces. The library
  maintains a stack state to make it easy to build buffers from a parser
  or similar.
- a small `flatcc/portable` header only library for non-C11 compliant
  compilers, and small helpers for all compilers including endian
  handling and numeric printing and parsing.

See also:

- [Reporting Bugs](https://github.com/dvidelabs/flatcc#reporting-bugs)

- [Google FPL FlatBuffers](http://google.github.io/flatbuffers/)

- [Online Forum](https://groups.google.com/forum/#!forum/flatbuffers)

- [Build Instructions](https://github.com/dvidelabs/flatcc#building)

- [Quickstart](https://github.com/dvidelabs/flatcc#quickstart)

- [Status](https://github.com/dvidelabs/flatcc#status)

- [Benchmark](https://github.com/dvidelabs/flatcc#benchmark)

The `flatcc` compiler is implemented as a standalone tool instead of
extending Googles `flatc` compiler in order to have a pure portable C
library implementation of the schema compiler that is designed to fail
graciously on abusive input in long running processes. It is also
believed a C version may help provide schema parsing to other language
interfaces that find interfacing with C easier than C++. The FlatBuffers
team at Googles FPL lab has been very helpful in providing feedback and
answering many questions to help ensure the best possible compatibility.
Notice the name `flatcc` (FlatBuffers C Compiler) vs Googles `flatc`.

The `flatcc` compiler has some extra features such as allowing for
referencing structs defined later in the schema - which makes sense
since it is already possible with other types. The binary schema format
also does not preserve the order of structs. The option is controlled by
a flag in `config.h` The generated source supports both bottom-up and
top-down construction mixed freely.

The JSON format is compatible with Googles `flatc` tool. The `flatc`
tool converts JSON from the command line using a schema and a buffer as
input. `flatcc` generates schema specific code to read and write JSON
at runtime. While the `flatcc` approach is likely much faster and also
easier to deploy, the `flatc` approach is likely more convenient when
manually working with JSON such as editing game scenes. Both tools have
their place. 

**NOTE: Big-endian platforms are untested but supported in principle.**


## Reporting Bugs

If possible, please provide a short reproducible schema and
source file using [issue4](https://github.com/dvidelabs/flatcc/issues/4)
as an example. The first comment in this issue details how to quickly
set up a new temporary project using the `scripts/setup.sh` script.


## Status

Main features supported as of 0.3.5:

- generated FlatBuffers reader and builder headers for C
- generated FlatBuffers verifier headers for C
- generated FlatBuffers JSON parser and printer for C
- ability to concatenate all output into one file, or to stdout
- robust dependency file generation for build systems
- binary schema (.bfbs) generation
- pre-generated reflection headers for handling .bfbs files
- cli schema compiler and library for compiling schema
- runtime library for builder, verifier and JSON support
- thorough test cases
- monster sample project
- fast build times

Supported platforms:

- Ubuntu gcc 4.4-4.8 and clang 3.5-3.8
- OS-X current clang / gcc
- Windows MSVC 2010, 2013, 2015, 2015 Win64 

The monster sample does not work with MSVC 2010 because it intentionally
uses C99 style code to better follow the C++ version.

There is no reason why other or older compilers cannot be supported, but
it may require some work in the build configuration and possibly
updates to the portable library. The above is simply what has been
tested and configured.

Use versions from 0.3.0 and up as there has been some minor breaking
[interface changes](https://github.com/dvidelabs/flatcc/blob/master/CHANGELOG.md#030).
Version 0.3.3 has a minor breaking change where the `verify_as_root` call
must be renamed to `verify_as_root_with_identifier`, or drop the identifier
argument.

Big endian platforms have not been tested at all. While care has been
taken to handle endian encoding, there are bound to be some issues. The
approach taken is to make it work on little endian - then it can always
be made to work on big endian later given that output generated by an
little endian platforms presumable will be correct regardless of bugs in
endian encoding.

The portability layer has some features that are generally important for
things like endian handling, and others to provide compatibility for
non-C11 compliant compilers. Together this should support most C
compilers around, but relies on community feedback for maturity.

The necessary size of the runtime include files can be reduced
significantly by using -std=c11 and avoiding JSON (which needs a lot of
numeric parsing support), and by removing `include/flatcc/reflection`
which is present to support handling of binary schema files and can be
generated from `reflection/reflection.fbs`, and removing
`include/flatcc/support` which is only used for tests and samples. The
exact set of required files may change from release to release, and it
doesn't really matter with respect to the compiled code size.

There are no plans to make frequent updates once the project becomes
stable, but input from the community will always be welcome and included
in releases where relevant, especially with respect to testing on
different target platforms.


## Time / Space / Usability Tradeoff

The priority has been to design an easy to use C builder interface that
is reasonably fast, suitable for both servers and embedded devices, but
with usability over absolute performance - still the small buffer output
rate is measured in millons per second and read access 10-100 millon
buffers per second from a rough estimate. Reading FlatBuffers is more
than an order of magnitude faster than building them.

For 100MB buffers with 1000 monsters, dynamically extended monster
names, monster vector, and inventory vector, the bandwidth reaches about
2.2GB/s and 45ms/buffer on 2.2GHz Haswell Core i7 CPU. This includes
reading back and validating all data. Reading only a few key fields
increases bandwidth to 2.7GB/s and 37ms/op. For 10MB buffers bandwidth
may be higher but eventually smaller buffers will be hit by call
overhead and thus we get down to 300MB/s at about 150ns/op encoding
small buffers. These numbers are just a rough guideline - they obviously
depend on hardware, compiler, and data encoded. Measurements are
excluding an ininitial warmup step. 

The generated JSON parsers are roughly 4 times slower than building a
FlatBuffer directly in C or C++, or about 2200ns vs 600ns for a 700 byte
JSON message. JSON parsing is thus roughly two orders of magnitude faster
than reading the equivalent Protocol Buffer, as reported on the [Google
FlatBuffers
Benchmarks](http://google.github.io/flatbuffers/flatbuffers_benchmarks.html)
page. LZ4 compression would estimated double the overall processing time
of JSON parsing. JSON printing is faster than parsing but not very
significantly so. JSON compresses to roughly half the size of compressed
FlatBuffers on large buffers, but compresses worse on small buffers (not
to mention when not compressing at all).

It should be noted that FlatBuffer read performance exclude verification
which JSON parsers and Protocol Buffers inherently include by their
nature. Verification has not been benchmarked, but would presumably add
less than 50% read overhead unless only a fraction of a large buffer is to
be read.

See also [benchmark](https://github.com/dvidelabs/flatcc#benchmark)
below.

The client C code can avoid almost any kind of allocation to build
buffers as a builder stack provides an extensible arena before
committing objects - for example appending strings or vectors piecemeal.
The stack is mostly bypassed when a complete object can be constructed
directly such as a vector from integer array on little endian platforms.

The reader interface should be pretty fast as is with less room for
improvement performance wise. It is also much simpler than the builder.

Usability has also been prioritized over smallest possible generated
source code and compile time. It shouldn't affect the compiled size
by much.

The compiled binary output should be reasonably small for everything but
the most restrictive microcontrollers. A 33K monster source test file
(in addition to the generated headers and the builder library) results
in a less than 50K optimized binary executable file including overhead
for printf statements and other support logic, or a 30K object file
excluding the builder library.

Read-only binaries are smaller but not necessarily much smaller than
builders considering they do less work: The compatibility test reads a
pre-generated binary `monsterdata_test.golden` monster file and verifies
that all content is as expected. This results in a 13K optimized binary
executable or a 6K object file. The source for this check is 5K
excluding header files. Readers do not need to link with a library. 

JSON parsers bloat the compiled C binary compared to pure Flatbuffer
usage because they inline the parser decision tree. A JSON parser for
monster.fbs may add 100K +/- optimization settings to the executable
binary.


## Generated Files

In earlier releases it was attempted to generate all code needed for
read-only buffer access. Now a library of include files is always
required (`include/flatcc`) because the original approach lead to
excessive code duplication. The generated code for building flatbuffers,
and for parsing and printing flatbuffers, all need to link with the
runtime library `libflatccrt.a`. The verifier and builder headers depend
on the reader header. The generated reader only depends on library
header files.

The reader and builder rely on generated common reader and builder
header files. These common file makes it possible to change the global
namespace and redefine basic types (`uoffset_t` etc.). In the future
this _might_ move into library code and use macros for these
abstractions and eventually have a set of predefined files for types
beyond the standard 32-bit unsigned offset (`uoffset_t`). The runtime
library is specific to one set of type definitions.

Reader code is reasonably straight forward and the generated code is
more readable than the builder code because the generated functions
headers are not buried in macros. Refer to `monster_test.c` and the
generated files for detailed guidance on use. The monster schema used in
this project is a slight adaptation to the original to test some
additional edge cases.

For building flatbuffers a separate builder header file is generated per
schema. It requires a `flatbuffers_common_builder.h` file also generated
by the compiler and a small runtime library `libflatccrt.a`. It is
because of this requirement that the reader and builder generated code
is kept separate. Typical uses can be seen in the `monster_test.c` file.
The builder allows of repeated pushing of content to a vector or a
string while a containing table is being updated which simplifies
parsing of external formats. It is also possible to build nested buffers
in-line - at first this may sound excessive but it is useful when
wrapping a union of buffers in a network interface and it ensures proper
alignment of all buffer levels.

For verifying flatbuffers, a `myschema_verifier.h` is generated. It
depends on the runtime library and the reader header.

Json parsers and printers generate one file per schema file and included
schema will have their own parsers and printers which including parsers
and printers will depend upon, rather similar to how builders work.

Low level note: the builder generates all vtables at the end of the
buffer instead of ad-hoc in front of each table but otherwise does the
same deduplication of vtables. This makes it possible to cluster vtables
in hot cache or to make sure all vtables are available when partially
transmitting a buffer. This behavior can be disabled by a runtime flag.

Because some use cases may include very constrained embedded devices,
the builder library can be customized with an allocator object and a
buffer emitter object. The separate emitter ensures a buffer can be
constructed without requiring a full buffer to be present in memory at
once, if so desired.

The typeless builder library is documented in `flatcc_builder.h` and
`flatcc_emitter.h` while the generated typed builder api for C is
documented in `doc/builder.md`.


## Using flatcc

Refer to `flatcc -h` for details.

The compiler can either generate a single header file or headers for all
included schema and a common file and with or without support for both
reading (default) and writing (-w) flatbuffers. The simplest option is
to use (-a) for all and include the `myschema_builder.h` file.

The (-a) or (-v) also generates a verifier file.

Make sure `flatcc` under the `include` folder is visible in the C
compilers include path when compiling flatbuffer builders.

The `flatcc` (-I) include path will assume all files with same base name
(case insentive) are identical and only include the first. All generated
files use the input basename and land in working directory or the path
set by (-o).

Note that the binary schema output can be with or without namespace
prefixes and the default differs from `flatc` which strips namespaces.
The binary schema can also have a non-standard size field prefixed so
multiple schema can be outfileenated in a single file if so desired (see
also the bfbs2json example).

Files can be generated to stdout using (--stdout). C headers will be
ordered and outfileenated, but otherwise identical to the separate file
output. Each include statement is guarded so this will not lead to
missing include files.

The generated code, especially with all combined with --stdout, may
appear large, but only the parts actually used will take space up the
the final executable or object file. Modern compilers inline and include
only necessary parts of the statically linked builder library.

JSON printer and parser can be generated using the --json flag or
--json-printer or json-parser if only one of them is required. There are
some certain runtime library compile time flags that can optimize out
printing symbolic enums, but these can also be disabled at runtime.


## Quickstart

After [building](https://github.com/dvidelabs/flatcc#building) the `flatcc tool`,
binaries are located in the `bin` and `lib` directories under the
`flatcc` source tree.

You can either jump directly to the [monster
example](https://github.com/dvidelabs/flatcc/tree/master/samples/monster)
that follows
[Googles FlatBuffers Tutorial](https://google.github.io/flatbuffers/flatbuffers_guide_tutorial.html), or you can read along the quickstart guide below. If you follow
the monster tutorial, you may want to clone and build flatcc and copy
the source to a separate project directory as follows:

    git clone https://github.com/dvidelabs/flatcc.git
    flatcc/scripts/setup.sh -a mymonster
    cd mymonster
    scripts/build.sh
    build/mymonster

`scripts/setup.sh` will as a minimum link the library and tool into a
custom directory, here `mymonster`. With (-a) it also adds a simple
build script, copies the example, and updates `.gitignore` - see
`scripts/setup.sh -h`. Setup can also build flatcc, but you still have
to ensure the build environment is configured for your system.

To write your own schema files please follow the main FlatBuffers
project documentation on [writing schema
files](https://google.github.io/flatbuffers/flatbuffers_guide_writing_schema.html).

The [builder interface
reference](https://github.com/dvidelabs/flatcc/blob/master/doc/builder.md)
may be useful after studying the monster sample and quickstart below.

When looking for advanced examples such as sorting vectors and finding
elements by a key, you should find these in the
[`test/monster_test`](https://github.com/dvidelabs/flatcc/tree/master/test/monster_test) project.

The following quickstart guide is a broad simplification of the
`test/monster_test` project - note that the schema is slightly different
from the tutorial. Focus is on the C specific framework rather
than general FlatBuffers concepts.

You can still use the setup tool to create an empty project and
follow along, but there are no assumptions about that in the text below.

## Quickstart - reading a buffer

Here we provide a quick example of read-only access to Monster flatbuffer -
it is an adapted extract of the `monster_test.c` file.

First we compile the schema read-only with common (-c) support header and we
add the recursion because `monster_test.fbs` includes other files.

    flatcc -cr test/monster_test/monster_test.fbs

For simplicity we assume you build an example project in the project
root folder, but in praxis you would want to change some paths, for
example:

    mkdir -p build/example
    flatcc -cr -o build/example test/monster_test/monster_test.fbs
    cd build/example

We get:

    flatbuffers_common_reader.h
    include_test1_reader.h
    include_test2_reader.h
    monster_test_reader.h

(There is also the simpler `samples/monster/monster.fbs` but then you won't get
included schema files).

Namespaces can be long so we optionally use a macro to manage this.

    #include "monster_test_reader.h"

    #undef ns
    #define ns(x) FLATBUFFERS_WRAP_NAMESPACE(MyGame_Example, x)

    int verify_monster(void *buffer)
    {
        ns(Monster_table_t) monster;
        /* This is a read-only reference to a flatbuffer encoded struct. */
        ns(Vec3_struct_t) vec;
        flatbuffers_string_t name;
        size_t offset;

        if (!(monster = ns(Monster_as_root(buffer)))) {
            printf("Monster not available\n");
            return -1;
        }
        if (ns(Monster_hp(monster)) != 80) {
            printf("Health points are not as expected\n");
            return -1;
        }
        if (!(vec = ns(Monster_pos(monster)))) {
            printf("Position is absent\n");
            return -1;
        }

        /* -3.2f is actually -3.20000005 and not -3.2 due to representation loss. */
        if (ns(Vec3_z(vec)) != -3.2f) {
            printf("Position failing on z coordinate\n");
            return -1;
        }

        /* Verify force_align relative to buffer start. */
        offset = (char *)vec - (char *)buffer;
        if (offset & 15) {
            printf("Force align of Vec3 struct not correct\n");
            return -1;
        }

        /*
         * If we retrieved the buffer using `flatcc_builder_finalize_aligned_buffer` or
         * `flatcc_builder_get_direct_buffer` the struct should also
         * be aligned without subtracting the buffer.
         */
        if (vec & 15) {
            printf("warning: buffer not aligned in memory\n");
        }

        /* ... */
        return 0;
    }
    /* main() {...} */


## Quickstart - compiling for read-only

Assuming our above file is `monster_example.c` the following are a few
ways to compile the project for read-only - compilation with runtime
library is shown later on.

    cc -I include monster_example.c -o monster_example

    cc -std=c11 -I include monster_example.c -o monster_example

    cc -D FLATCC_PORTABLE -I include monster_example.c -o monster_example

The include path or source path is likely different. Some files in
`include/flatcc/portable` are always used, but the `-D FLATCC_PORTABLE`
flag includes additional files to support compilers lacking c11
features.


## Quickstart - building a buffer

Here we provide a very limited example of how to build a buffer - only a few
fields are updated. Pleaser refer to `monster_test.c` and the doc directory
for more information.

First we must generate the files:

    flatcc -a monster_test.fbs

This produces:

    flatbuffers_common_reader.h
    flatbuffers_common_builder.h
    flatbuffers_common_verifier.h
    include_test1_reader.h
    include_test1_builder.h
    include_test1_verifier.h
    include_test2_reader.h
    include_test2_builder.h
    include_test2_verifier.h
    monster_test_reader.h
    monster_test_builder.h
    monster_test_verifier.h

Note: we wouldn't actually do the readonly generation shown earlier
unless we only intend to read buffers - the builder generation always
generates read acces too.

By including `"monster_test_builder.h"` all other files are included
automatically. The C compiler needs the -I include directive to access
`flatbuffers/flatcc_builder.h` and related.

The verifiers are not required and just created because we lazily chose
the -a option.

The builder must be initialized first to set up the runtime environment
we need for building buffers efficiently - the builder depends on an
emitter object to construct the actual buffer - here we implicitly use
the default. Once we have that, we can just consider the builder a
handle and focus on the FlatBuffers generated API until we finalize the
buffer (i.e. access the result).  For non-trivial uses it is recommended
to provide a custom emitter and for example emit pages over the network
as soon as they complete rather than merging all pages into a single
buffer using `flatcc_builder_finalize_buffer`, or the simplistic
`flatcc_builder_get_direct_buffer` which returns null if the buffer is
too large. See also documentation comments in `flatcc_builder.h` and
`flatcc_emitter.h`.


    #include "monster_test_builder.h"

    /* See `monster_test.c` for more advanced examples. */
    void build_monster(flatcc_builder_t *B)
    {
        ns(Vec3_t *vec);

        /* Here we use a table, but structs can also be roots. */
        ns(Monster_start_as_root(B));

        ns(Monster_hp_add(B, 80));
        /* The vec struct is zero-initalized. */
        vec = ns(Monster_pos_start(B));
        /* Native endian. */
        vec->x = 1, vec->y = 2, vec->z = -3.2f;
        /* _end call converts to protocol endian format - for LE it is a nop. */
        ns(Monster_pos_end(B));

        /* Name is required, or we get an assertion in debug builds. */
        ns(Monster_name_create_str(B, "MyMonster"));

        ns(Monster_end_as_root(B));
    }

    #include "flatcc/support/hexdump.h"

    int main(int argc, char *argv[])
    {
        flatcc_builder_t builder;
        void *buffer;
        size_t size;

        flatcc_builder_init(&builder);

        build_monster(&builder);
        /* We could also use `flatcc_builder_finalize_buffer` and free the buffer later. */
        buffer = flatcc_builder_get_direct_buffer(&builder, &size);
        assert(buffer);
        verify_monster(buffer);

        /* Visualize what we got ... */
        hexdump("monster example", buffer, size, stdout);

        /*
         * Here we can call `flatcc_builder_reset(&builder) if
         * we wish to build more buffers before deallocating
         * internal memory with `flatcc_builder_clear`.
         */

        flatcc_builder_clear(&builder);
        return 0;
    }

Compile the example project:

    cc -std=c11 -I include monster_example.c lib/libflatccrt.a -o monster_example

Note that the runtime library is required for building buffers, but not
for reading them. If it is incovenient to distribute the runtime library
for a given target, source files may be used instead. Each feature has
its own source file, so not all runtime files are needed for building a
buffer:

    cc -std=c11 -I include monster_example.c \
        src/runtime/emitter.c src/runtime/builder.c \
        -o monster_example

Other features such as the verifier and the JSON printer and parser
would each need a different file in src/runtime. Which file should be
obvious from the filenames except that JSON parsing also requires the
builder and emitter source files.


## Verifying a Buffer

A buffer can be verified to ensure it does not contain any ranges that
point outside the the given buffer size, that all data structures are
aligned according to the flatbuffer principles, that strings are zero
terminated, and that required fields are present.

In the builder example above, we can apply a verifier to the output:

    #include "monster_test_builder.h"
    #include "monster_test_verifier.h"
    int ret;
    ...
    ... finalize
    if ((ret = ns(Monster_verify_as_root(buffer, size, "MONS")))) {
        printf("Monster buffer is invalid: %s\n",
        flatcc_verify_error_string(ret));
    }

Flatbuffers can optionally leave out the identifier, here "MONS". Use a
null pointer as identifier argument to ignore any existing identifiers
and allow for missing identifiers.

Nested flatbbuffers are always verified with a null identifier, but it
may be checked later when accessing the buffer.

The verifier does NOT verify that two datastructures are not
overlapping. Sometimes this is indeed valid, such as a DAG (directed
acyclic graph) where for example two string references refer to the same
string in the buffer. In other cases an attacker may maliciously
construct overlapping datastructures such that in-place updates may
cause subsequent invalid buffers. Therefore an untrusted buffer should
never be updated in-place without first rewriting it to a new buffer.

Note: prior to version 0.2.0, the verifier would fail on 0 and report
success on non-zero value. As of 0.2.0, success is indicated by 0, and
non-zero yields an error code that can be translated into a string.

The CMake build system has build option to enable assertions in the
verifier. This will break debug builds and not usually what is desired,
but it can be very useful when debugging why a buffer is invalid.

See also `include/flatcc/flatcc_verifier.h`.


## File and Type Identifiers

There are two ways to identify the content of a FlatBuffer. The first is
to use file identifiers which are defined in the schema. The second is
to use `type identifiers` which are calculated hashes based on each
tables name prefixed with its namespace, if any. In either case the
identifier is stored at offset 4 in binary FlatBuffers, when present.
Type identifiers are not to be confused with union types.

### File Identifiers

The FlatBuffers schema language has the optional `file_identifier`
declaration which accepts a 4 characer ASCII string. It is intended to be
human readable. When absent, the buffer potentially becomes 4 bytes
shorter (depending on padding).

The `file_identifier` is intended to match the `root_type` schema
declaration, but this does not take into account that it is convenient
to create FlatBuffers for other types as well. `flatcc` makes no special
destinction for the `root_type` while Googles `flatc` JSON parser uses
it to determine the JSON root object type.

As a consequence, the file identifier is ambigous. Included schema may
have separate `file_identifier` declarations. To at least make sure each
type is associated with its own schemas `file_identifier`, a symbol is
defined for each type. If the schema has such identifier, it will be
defined as the null identifier.

The generated code defines the identifiers for a given table:

    #ifndef MyGame_Example_Monster_identifier
    #define MyGame_Example_Monster_identifier flatbuffers_identifier
    #endif

The `flatbuffers_identifier` is the schema specific `file_identifier`
and is undefined and redefined for each generated `_reader.h` file.

The user can now override the identifier for a given type, for example:

    #define MyGame_Example_Vec3_identifer "VEC3"
    #include "monster_test_builder.h"

    ...
    MyGame_Example_Vec3_create_as_root(B, ...);

The `create_as_root` method uses the identifier for the type in question,
and so does other `_as_root` methods.

The `file_extension` is handled in a similar manner:

    #ifndef MyGame_Example_Monster_extension
    #define MyGame_Example_Monster_extension flatbuffers_extension
    #endif

### Type Identifiers

To better deal with the ambigouties of file identifiers, type
identifiers have been introduced as an alternative 4 byte buffer
identifier. The hash is standardized on FNV-1a for interoperability.

The type identifier use a type hash which maps a fully qualified type
name into a 4 byte hash. The type hash is a 32-bit native value and the
type identifier is a 4 character little endian encoded string of the
same value.

In this example the type hash is derived from the string
"MyGame.Example.Monster" and is the same for all FlatBuffer code
generators that supports type hashes.

The value 0 is used to indicate that one does not care about the
identifier in the buffer. 

    ...
    MyGame_Example_Monster_create_as_typed_root(B, ...);
    buffer = flatcc_builder_get_direct_buffer(B);
    MyGame_Example_Monster_verify_as_typed_root(buffer, size);
    // read back
    monster = MyGame_Example_Monster_as_typed_root(buffer);

    switch (flatbuffers_get_type_hash(buffer)) {
    case MyGame_Example_Monster_type_hash:
        ...

    }
    ...
    if (flatbuffers_get_type_hash(buffer) ==
        flatbuffers_type_hash_from_name("Some.Old.Buffer")) {
        printf("Buffer is the old version, not supported.\n"); 
    }

More API calls are available to naturally extend the existing API. See
`test/monster_test/monster_test.c` for more.

The type identifiers are defined like:

    #define MyGame_Example_Monster_type_hash ((flatbuffers_thash_t)0x330ef481)
    #define MyGame_Example_Monster_type_identifier "\x81\xf4\x0e\x33"

The `type_identifier` can be used anywhere the original 4 character
file identifier would be used, but a buffer must choose which system, if any,
to use. This will not affect the `file_extension`.


The
[`flatcc/flatcc_identifier.h`](https://github.com/dvidelabs/flatcc/blob/master/include/flatcc/flatcc_identifier.h)
file contains an implementation of the FNV-1a hash used. The hash was
chosen for simplicity, availability, and collision resistance. For
better distribution, and for internal use only, a dispersion function is
also provided, mostly to discourage use of alternative hashes in
transmission since the type hash is normally good enough as is.

_Note: there is a potential for collisions in the type hash values
because the hash is only 4 bytes._


## JSON Parsing and Printing

JSON support files are generated with `flatcc --json`.

This section is not a tutorial on JSON printing and parsing, it merely
covers some non-obvious aspects. The best source to get started quickly
is the test file:

    test/json_test/json_test.c

For detailed usage, please refer to:

    test/json_test/test_json_printer.c
    test/json_test/test_json_parser.c
    test/json_test/json_test.c
    test/benchmark/benchflatccjson

See also JSON parsing section in the Googles FlatBuffers [schema
documentation](https://google.github.io/flatbuffers/flatbuffers_guide_writing_schema.html).

By using the flatbuffer schema it is possible to generate schema
specific JSON printers and parsers. This differs for better and worse
from Googles `flatc` tool which takes a binary schema as input and
processes JSON input and output. Here that parser and printer only rely
on the `flatcc` runtime library, is faster (probably significantly so),
but requires recompilition when new JSON formats are to be supported -
this is not as bad as it sounds - it would for example not be difficult
to create a Docker container to process a specific schema in a web
server context.

The parser always takes a text buffer as input and produces output
according to how the builder object is initialized. The printer has
different init functions: one for printing to a file pointer, including
stdout, one for printing to a fixed size external buffer, and one for
printing to a dynamically growing buffer. The dynamic buffer may be
reused between prints via the reset function. See `flatcc_json_parser.h`
for details.

The parser will accept unquoted names (not strings) and trailing commas,
i.e. non-strict JSON and also allows for hex `\x03` in strings. Strict
mode must be enabled by a compile time flag. In addition the parser
schema specific symbolic enum values that can optionally be unquoted
where a numeric value is expected:

    color: Green
    color: Color.Green
    color: MyGame.Example.Color.Green
    color: 2

The symbolic values do not have to be quoted (unless required by runtime
or compile time configuration), but can be while numeric values cannot
be quoted. If no namespace is provided, like `color: Green`, the symbol
must match the receiving enum type. Any scalar value may receive a
symbolic value either in a relative namespace like `hp: Color.Green`, or
an absolute namespace like `hp: MyGame.Example.Color.Green`, but not
`hp: Green` (since `hp` in the monster example schema) is not an enum
type with a `Green` value). A namespace is relative to the namespace of
the receiving object.

It is also possible to have multiple values, but these always have to be
quoted in order to be compatible with Googles flatc tool for Flatbuffers
1.1:

    color: "Green Red"

The following is also accepted in flatcc 0.2.0, but subsequent releases
only permits it if explicitly enabled at compile time.

    color: Green Red

These multi value expressions are originally intended for enums that
have the bit flag attribute defined (which Color does have), but this is
tricky to process, so therefore any symblic value can be listed in a
sequence with or without namespace as appropriate. Because this further
causes problems with signed symbols the exact definition is that all
symbols are first coerced to the target type (or fail), then added to
the target type if not the first this results in:

    color: "Green Blue Red Blue"
    color: 19

Because Green is 2, Red is 1, Blue is 8 and repeated.

__NOTE__: Duplicate values should be considered implemention dependent
as it cannot be guaranteed that all flatbuffer JSON parsers will handle
this the same. It may also be that this implementation will change in
the future, for example to use bitwise or when all members and target
are of bit flag type.

It is not valid to specify an empty set like:

    color: ""

because it might be understood as 0 or the default value, and it does
not unquote very well.

The printer will by default print valid json without any spaces and
everything quoted. Use the non-strict formatting option (see headers and
test examples) to produce pretty printing. It is possibly to disable
symbolic enum values using the `noenum` option.

Only enums will print symbolic values are there is no history of any
parsed symbolic values at all. Furthermore, symbolic values are only
printed if the stored value maps cleanly to one value, or in the case of
bit-flags, cleanly to multiple values. For exmaple if parsing `color: Green Red`
it will print as `"color":"Red Green"` by default, while `color: Green
Blue Red Blue` will print as `color:19`.

Both printer and parser are limited to roughly 100 table nesting levels
and an additional 100 nested struct depths. This can be changed by
configuration flags but must fit in the runtime stack since the
operation is recursive descent. Exceedning the limits will result in an
error.

Numeric values are coerced to the receiving type. Integer types will
fail if the assignment does not fit the target while floating point
values may loose precision silently. Integer types never accepts
floating point values. Strings only accept strings.

Nested flatbuffers may either by arrays of byte sized integers, or a
table or a struct of the target type. See test cases for details.

The parser will by default fail on unknown fields, but these can also be
skipped silently with a runtime option.

Unions are difficult to parse. A union is two json fields: a table as
usual, and an enum to indicate the type which has the same name with a
`_type` suffix and accepts a numeric or symbolic type code:

    {
      name: "Container Monster", test_type: Monster,
      test: { name: "Contained Monster" }
    }

Because other json processors may sort fields, it is possible to receive
the type field after the test field. The parser does not store temporary
datastructures. It constructs a flatbuffer directly. This is not
possible when the type is late. This is handled by parsing the field as
a skipped field on a first pass, followed by a typed back-tracking
second pass once the type is known (only the table is parsed twice, but
for nested unions this can still expand). Needless to say this slows down
parsing. It is an error to provide only the table field or the type
field alone, except if the type is `NONE` or `0` in which case the table
is not allowed to be present.


### Performance Notes

Note that json parsing and printing is very fast reaching 500MB/s for
printing and about 300 MB/s for parsing. Floating point parsing can
signficantly skew these numbers. The integer and floating point parsing
and printing are handled via support functions in the portable library.
In addition the floating point `include/flatcc/portable/grisu3_*` library
is used unless explicitly disable by a compile time flag. Disabling
`grisu3` will revert to `sprintf` and `strtod`. Grisu3 will fall back to
`strtod` and `grisu3` in some rare special cases. Due to the reliance on
`strtod` and because `strtod` cannot efficiently handle
non-zero-terminated buffers, it is recommended to zero terminate
buffers. Alternatively, grisu3 can be compiled with a flag that allows
errors in conversion. These errors are very small and still correct, but
may break some checksums. Allowing for these errors can significantly
improve parsing speed and moves the benchmark from below half a million
parses to above half a million parses per second on 700 byte json
string, on a 2.2 GHz core-i7.

While unquoted strings may sound more efficient due to the compact size,
it is actually slower to process. Furthermore, large flatbuffer
generated JSON files may compress by a factor 8 using gzip or a factor
4 using LZ4 so this is probably the better place to optimize. For small
buffers it may be more efficient to compress flatbuffer binaries, but
for large files, json may actually compress significantly better due to
the absence of pointers in the format.

SSE 4.2 has been experimentally added, but it the gains are limited
because it works best when parsing space, and the space parsing is
already fast without SSE 4.2 and because one might just leave out the
spaces if in a hurry. For parsing strings, trivial use of SSE 4.2 string
scanning doesn't work well becasuse all the escape codes below ASCII 32
must be detected rather than just searching for `\` and `"`. That is not
to say there are not gains, they just don't seem worthwhile.

The parser is heavily optimized for 64-bit because it implements an
8-byte wide trie directly in code. It might work well for 32-bit
compilers too, but this hasn't been tested. The large trie does put some
strain on compile time. Optimizing beyond -O2 leads to too large
binaries which offsets any speed gains.


## Global Scope and Included Schema

Attributes included in the schema are viewed in a global namespace and
each include file adds to this namespace so a schema file can use
included attributes without namespace prefixes.

Each included schema will also add types to a global scope until it sees
a `namespace` declaration. An included schema does not inherit the
namespace of an including file or an earlier included file, so all
schema files starts in the global scope. An included file can, however,
see other types previously defined in the global scope. Because include
statements always appear first in a schema, this can only be earlier
included files, not types from a containing schema.

The generated output for any included schema is indendent of how it was
included, but it might not compile without the earlier included files
being present and included first. By including the toplevel `myschema.h`
or `myschema_builder.h` all these dependencies are handled correctly.

Note: `libflatcc.a` can only parse a single schema when the schema is
given as a memory buffer, but can handle the above when given a
filename. It is possible to outfileename schema files, but a `namespace;`
declaration must be inserted as a separator to revert to global
namespace at the start of each included file. This can lead to subtle
errors because if one parent schema includes two child schema `a.fbs`
and `b.fbs`, then `b.fbs` should not be able to see anything in `a.fbs`
even if they share namespaces. This would rarely be a problem in praxis,
but it means that schema compilation from memory buffers cannot
authoratively validate a schema. The reason the schema must be isolated
is that otherwise code generation for a given schema could change with
how it is being used leading to very strange errors in user code.


## Required Fields and Duplicate Fields

If a field is required such as Monster.name, the table end call will
assert in debug mode and create incorrect tables in non-debug builds.
The assertion may not be easy to decipher as it happens in library code
and it will not tell which field is missing.

When reading the name, debug mode will again assert and non-debug builds
will return a default value.

Writing the same field twice will also trigger an assertion in debug
builds.


## Fast Buffers

Buffers can be used for high speed communication by using the ability to
create buffers with structs as root. In addition the default emitter
supports `flatcc_emitter_direct_buffer` for small buffers so no extra copy
step is required to get a linear buffer in memory. Preliminary
measurements suggests there is a limit to how fast this can go (about
6-7 mill. buffers/sec) because the builder object must be reset between
buffers which involves zeroing allocated buffers. Small tables with a
simple vector achieve roughly half that speed. For really high speed a
dedicated builder for structs would be needed. See also
`monster_test.c`.


## Types

All types stored in a buffer has a type suffix such as `Monster_table_t`
or `Vec3_struct_t` (and namespace prefix which we leave out here). These
types are read-only pointers into endian encoded data. Enum types are
just constants easily grasped from the generated code. Tables are dense so
they are never accessed directly.

Structs have a dual purpose because they are also valid types in native
format, yet the native reprsention has a slightly different purpose.
Thus the convention is that a const pointer to a struct encoded in a
flatbuffer has the type `Vec3_struct_t` where as a writeable pointer to
a native struct has the type `Vec3_t *` or `struct Vec3 *`.

Union types are just any of a set of possible table types and an enum
named as for example `Any_union_type_t`. There is a compound union type
that can store both type and table reference such that `create` calls
can represent unions as a single argument - see `flatcc_builder.h` and
`doc/builder.md`. Union table fields return a pointer of type
`flatbuffers_generic_table_t` which is defined as `const void *`.

All types have a `_vec_t` suffix which is a const pointer to the
underlying type. For example `Monster_table_t` has the vector type
`Monster_vec_t`. There is also a non-const variant with suffix
`_mutable_vec_t` which is rarely used. However, it is possible to sort
vectors in-place in a buffer, and for this to work, the vector must be
cast to mutable first. A vector (or string) type points to the element
with index 0 in the buffer, just after the length field, and it may be
cast to a native type for direct access with attention to endian
encoding. (Note that `table_t` types do point to the header field unlike
vectors.) These types are all for the reader interface. Corresponding
types with a `_ref_t` suffix such as `_vec_ref_t` are used during
the construction of buffers.

Native scalar types are mapped from the FlatBuffers schema type names
such as ubyte to `uint8_t` and so forth. These types also have vector
types provided in the common namespace (default `flatbuffers_`) so
a `[ubyte]` vector has type `flatbuffers_uint8_vec_t` which is defined
as `const uint8_t *`.

The FlatBuffers boolean type is strictly 8 bits wide so we cannot use or
emulate `<stdbool.h>` where `sizeof(bool)` is implementation dependent.
Therefore `flatbuffers_bool_t` is defined as `uint8_t` and used to
represent FlatBuffers boolean values and the constants of same type:
`flatbuffers_true = 1` and `flatbuffers_false = 0`. Even so,
`pstdbool.h` is available in the `include/flatcc/portable` directory if
`bool`, `true`, and `false` are desired in user code and `<stdbool.h>`
is unavailable.

`flatbuffers_string_t` is `const char *` but imply the returned pointer
has a length prefix just before the pointer. `flatbuffers_string_vec_t`
is a vector of strings. The `flatbufers_string_t` type guarantees that a
length field is present using `flatbuffers_string_len(s)` and that the
string is zero terminated. It also suggests that it is in utf-8 format
according to the FlatBuffers specification, but not checks are done and
the `flatbuffers_create_string(B, s, n)` call explicitly allows for
storing embedded null characters and other binary data.

All vector types have operations defined as the typename with `_vec_t`
replaced by `_vec_at` and `_vec_len`. For example
`flatbuffers_uint8_vec_at(inv, 1)` or `Monster_vec_len(inv)`. The length
or `_vec_len` will be 0 if the vector is missing whereas `_vec_at` will
assert in debug or behave undefined in release builds following out of
bounds access. This also applies to related string operations.


## Endianness

The generated code supports the `FLATBUFFERS_LITTLEENDIAN` flag defined
by the `flatc` compiler and it can be used to force endianness. Big
Endian will define it as 0. Other endian may lead to unexpected results.
In most cases `FLATBUFFERS_LITTLEENDIAN` will be defined but in some
cases a decision is made in runtime where the flag cannot be defined.
This is likely just as fast, but `#if FLATBUFFERS_LITTLEENDIAN` can lead
to wrong results alone - use `#if defined(FLATBUFFERS_LITTLEENDIAN) &&
FLATBUFFERS_ENDIAN` to be sure the platform is recognized as little
endian. The detection logic will set `FLATBUFFERS_LITTLEENDIAN` if at
all possible and can be improved with the `pendian.h` file included by
the `-DFLATCC_PORTABLE`.

The user can always define `-DFLATBUFFERS_LITTLEENDIAN` as a compile
time option and then this will take precedence.

It is recommended to use `flatbuffers_is_native_pe()` instead of testing
`FLATBUFFERS_LITTLEENDIAN` whenever it can be avoided to use the
proprocessor for several reasons:

- if it isn't defined, the source won't compile at all
- combined with `pendian.h` it provides endian swapping even without
  preprocessor detection.
- it is normally a constant similar to `FLATBUFFERS_LITTLEENDIAN`
- it works even with undefined `FLATBUFFERS_LITTLEENDIAN`
- compiler should optimize out even runtime detection
- protocol might not always be little endian.
- it is defined as a macro and can be checked for existence.

`pe` means protocol endian. This suggests that `flatcc` output may be
used for other protocols in the future, or for variations of
flatbuffers. The main reason for this is the generated structs that can
be very useful on other predefined network protols in big endian. Each
struct has a `mystruct_copy_from_pe` method and similar to do these
conversions. Internally flatcc optimizes struct conversions by testing
`flatbuffers_is_native_pe()` in some heavier struct conversions.

In a few cases it may be relevant to test for `FLATBUFFERS_LITTLEENDIAN`
but then code intended for general use should provide an alternitive for
when the flag isn't defined.

Even with correct endian detection, the code may break on platforms
where `flatbuffers_is_native_pe()` is false because the necessary system
headers could not found. In this case `-DFLATCC_PORTABLE` should help.


## Offset Sizes and Direction

FlatBuffers use 32-bit `uoffset_t` and 16-bit `voffset_t`. `soffset_t`
always has the same size as `uoffset_t`. These types can be changed by
preprocessor defines without regenerating code. However, it is important
that `libflatccrt.a` is compiled with the same types as defined in
`flatcc_types.h`.

`uoffset_t` currently always point forward like `flatc`. In retrospect
it would probably have simplified buffer constrution if offsets pointed
the opposite direction. This is a major change and not likely to happen
for reasons of effort and compatibility, but it is worth keeping in mind
for a v2.0 of the format.

Vector header fields storing the length are defined as `uoffset_t` which
is 32-bit wide by default. If `uoffset_t` is redefined this will
therefore also affect vectors and strings. The vector and string length
and index arguments are exposed as `size_t` in user code regardless of
underlying `uoffset_t` type.

The practical buffer size is limited to about half of the `uoffset_t` range
because vtable references are signed which in effect means that buffers
are limited to about 2GB by default.


## Pitfalls in Error Handling

The builder API often returns a reference or a pointer where null is
considered an error or at least a missing object default. However, some
operations do not have a meaningful object or value to return. These
follow the convention of 0 for success and non-zero for failure.
Also, if anything fails, it is not safe to proceed with building a
buffer.  However, to avoid overheads, there is no hand holding here. On
the upside, failures only happen with incorrect use or allocation
failure and since the allocator can be customized, it is possible to
provide a central error state there or to guarantee no failure will
happen depending on use case, assuming the API is otherwise used
correctly.  By not checking error codes, this logic also optimizes out
for better performance.


## Sorting and Finding

The builder API does not support sorting due to the complexity of
customizable emitters, but the reader API does support sorting so a
buffer can be sorted at a later stage. This requires casting a vector to
mutable and calling the sort method available for fields with keys.

The sort uses heap sort and can sort a vector in-place without using
external memory or recursion. Due to the lack of external memory, the
sort is not stable. The corresponding find operation returns the lowest
index of any matching key, or `flatbuffers_not_found`.

When configured in `config.h`, the `flatcc` compiler allows multiple
keyed fields unlike Googles `flatc` compiler. This works transparently
by providing `<table_name>_vec_sort_by_<field_name>` and
`<table_name>_vec_find_by_<field_name>` methods for all keyed fields. The
first field maps to `<table_name>_vec_sort` and `<table_name>_vec_find`.
Obviously the chosen find method must match the chosen sort method.

See also `doc/builder.md` and `test/monster_test/monster_test.c`.


## Null Values

The FlatBuffers format does not fully distinguish between default values
and missing or null values but it is possible to force values to be
written to the buffer.  This is discussed further in the `builder.md`.
For SQL data roundtrips this may be more important that having compact
data.

The `_is_present` suffix on table access methods can be used to detect if
value is present in a vtable, for example `Monster_hp_present`. Unions
return true of the type field is present, even if it holds the value
None.

The `add` methods have corresponding `force_add` methods for scalar and enum
values to force storing the value even if it is default and thus making
it detectable by `is_present`.


## Portability Layer

Some aspects of the portablity layer is not required when -std=c11 is
defined on a clang compiler where little endian is avaiable and easily
detected, or where `<endian.h>` is available and easily detected.
`flatbuffers_common_reader.h` contains a minimal portability abstraction
that also works for some platforms even without C11, e.g. OS-X clang.
The portablity file can be included before other headers, or by setting
the compile time directives:

    cc -D FLATCC_PORTABLE -I include monster_test.c -o monster_test

Also see the top of this file on how to include the actual portability layer
when needed. Mandatory aspects of the portability layer are not
sensitive to `FLATCC_PORTABLE`, these will be included as needed.

If a specific platform has been tested, it would be good with feedback
and possibly patches to the portability layer so this can be documented
for other users.

## Building

### Unix Build (OS-X, Linux, related)

To initialize and run the build (see required build tools below):

    scripts/build.sh

The `bin` and `lib` folders will be created with debug and release
build products.

The build depends on `CMake`. By default the `Ninja` build tool is also required,
but alternatively `make` can be used.

Optionally switch to a different build tool by choosing one of:

    scripts/initbuild.sh make
    scripts/initbuild.sh make-concurrent
    scripts/initbuild.sh ninja

where `ninja` is the default and `make-concurrent` is `make` with the `-j`
flag. A custom build configuration `X` can be added by adding a
`scripts/build.cfg.X` file.

`scripts/initbuild.sh` cleans the build if a specific build
configuration is given as argument. Without arguments it only ensures
that CMake is initialized and is therefore fast to run on subsequent
calls. This is used by all test scripts.

To install build tools on OS-X, and build:

    brew update
    brew install cmake ninja
    git clone https://github.com/dvidelabs/flatcc.git
    cd flatcc
    scripts/build.sh

To install build tools on Ubuntu, and build:

    sudo apt-get update
    sudo apt-get install cmake ninja-build
    git clone https://github.com/dvidelabs/flatcc.git
    cd flatcc
    scripts/build.sh

To install build tools on Centos, and build:

    sudo yum group install "Development Tools"
    sudo yum install cmake
    git clone https://github.com/dvidelabs/flatcc.git
    cd flatcc
    scripts/initbuild.sh make # there is no ninja build tool
    scripts/build.sh


OS-X also has a HomeBrew package:

    brew update
    brew install flatcc

or for the bleeding edge:

    brew update
    brew install flatcc --HEAD


### Windows Build (MSVC)

Install CMake, MSVC, and git (tested with MSVC 14 2015).

In PowerShell:

    git clone https://github.com/dvidelabs/flatcc.git
    cd flatcc
    mkdir build\MSVC
    cd build\MSVC
    cmake -G "Visual Studio 14 2015" ..\..

Optionally also build from the command line (in build\MSVC):

    cmake --build . --target --config Debug
    cmake --build . --target --config Release

In Visual Studio:

    open flatcc\build\MSVC\FlatCC.sln
    build solution
    choose Release build configuration menu
    rebuild solution

*Note that `flatcc\CMakeList.txt` sets the `-DFLATCC_PORTABLE` flag and
that `include\flatcc\portable\pwarnings.h` disable certain warnings for
warning level -W3.*


### Cross-compilation

Users have been reporting some degree of success using cross compiles
from Linux x86 host to embedded ARM Linux devices.

For this to work, `FLATCC_TEST` option should be disabled in part
because cross-compilation cannot run the cross-compiled flatcc tool, and
in part because there appears to be some issues with CMake custom build
steps needed when building test and sample projects.

The option `FLATCC_RTONLY` will disable tests and only build the runtime
library.

The following is not well tested, but may be a starting point:

    mkdir -p build/xbuild
    cd build/xbuild
    cmake ../.. -DBUILD_SHARED_LIBS=on -DFLATCC_RTONLY=on \
      -DCMAKE_BUILD_TYPE=Release

Overall, it may be simpler to create a separate Makefile and just
compile the few `src/runtime/*.c` into a library and distribute the
headers as for other platforms, unless `flatcc` is also required for the
target. Or to simply include the runtime source and header files in the user
project.

Note that no tests will be built nor run with `FLATCC_RTONLY` enabled.
It is highly recommended to at least run the `tests/monster_test`
project on a new platform.


## Distribution

Install targes may be built with:

    mkdir -p build/install
    cd build/install
    cmake ../.. -DBUILD_SHARED_LIBS=on -DFLATCC_RTONLY=on \
      -DCMAKE_BUILD_TYPE=Release -DFLATCC_INSTALL=on
    make install

However, this is not well tested and should be seen as a starting point.
The normal scripts/build.sh places files in bin and lib of the source tree.


### Unix Files

To distribute the compiled binaries the following files are
required:

Compiler:

    bin/flatcc                 (command line interface to schema compiler)
    lib/libflatcc.a            (optional, for linking with schema compiler)
    include/flatcc/flatcc.h    (optional, header and doc for libflatcc.a)

Runtime:

    include/flatcc/**          (runtime header files)
    include/flatcc/reflection  (optional)
    include/flatcc/support     (optional, only used for test and samples)
    lib/libflatccrt.a          (runtime library)

In addition the runtime library source files may be used instead of
`libflatccrt.a`. This may be handy when packaging the runtime library
along with schema specific generated files for a foreign target that is
not binary compatible with the host system:

    src/runtime/*.c

### Windows Files

The build products from MSVC are placed in the bin and lib subdirectories:

    flatcc\bin\Debug\flatcc.exe
    flatcc\lib\Debug\flatcc_d.lib
    flatcc\lib\Debug\flatccrt_d.lib
    flatcc\bin\Release\flatcc.exe
    flatcc\lib\Release\flatcc.lib
    flatcc\lib\Release\flatccrt.lib

Runtime `include\flatcc` directory is distributed like other platforms.


## Running Tests on Unix

Run

    scripts/test.sh [--no-clean]

**NOTE:** The test script will clean everything in the build directy before
initializing CMake with the chosen or default build configuration, then
build Debug and Release builds, and run tests for both.

The script must end with `TEST PASSED`, or it didn't pass.

To make sure everything works, also run the benchmarks:

    scripts/benchmark.sh


## Running Tests on Windows

In Visual Studio the test can be run as follows: first build the main
project, the right click the `RUN_TESTS` target and chose build. See
the output window for test results.

It is also possible to run tests from the command line after the project has
been built:

    cd build\MSVC
    ctest

Note that the monster example is disabled for MSVC 2010.

Be aware that tests copy and generate certain files which are not
automatically cleaned by Visual Studio. Close the solution and wipe the
`MSVC` directory, and start over to get a guaranteed clean build.

Please also observe that the file `.gitattributes` is used to prevent
certain files from getting CRLF line endings. Using another source
control systems might break tests, notably
`test/flatc_compat/monsterdata_test.golden`.


*Note: Benchmarks have not been ported to Windows.*


## Configuration

The configuration

    config/config.h

drives the permitted syntax and semantics of the schema compiler and
code generator. These generally default to be compatible with
Googles `flatc` compiler. It also sets things like permitted nesting
depth of structs and tables.

The runtime library has a separate configuration file

    include/flatcc/flatcc_rtconfig.h

This file can modify certain aspects of JSON parsing and printing such
as disabling the Grisu3 library or requiring that all names in JSON are
quoted.

For most users, it should not be relevant to modify these configuration
settings. If changes are required, they can be given in the build
system - it is not necessary to edit the config files, for example
to disable trailing comma in the JSON parser:

    cc -DFLATCC_JSON_PARSE_ALLOW_TRAILING_COMMA=0 ...


## Using the Compiler and Builder library

The compiler library `libflatcc.a` can compile schemas provided
in a memory buffer or as a filename. When given as a buffer, the schema
cannot contain include statements - these will cause a compile error.

When given a filename the behavior is similar to the commandline
`flatcc` interface, but with more options - see `flatcc.h` and
`config/config.h`.

`libflatcc.a` supports functions named `flatcc_...`. `reflection...` may
also be available which are simple the C generated interface for the
binary schema. The builder library is also included. These last two
interfaces are only present because the library supports binary schema
generation.

The standalone runtime library `libflatccrt.a` is a collection of the
`src/runtime/*.c` files. This supports the generated C headers for
various features. It is also possible to distribute and compile with the
source files directly.  For debugging, it is useful to use the
`libflatccrt_d.a` version because it catches a lot of incorrect API use
in assertions.

The runtime library may also be used by other languages. See comments
in `include/flatcc/flatcc_builder.h`. JSON parsing is on example of an
alternative use of the builder library so it may help to inspect the
generated JSON parser source and runtime source.

## The Portable Library

The portable library is placed under `include/flatcc/portable` and is
required by flatcc, but isn't strictly part of the `flatcc` project. It
is intended as an independent light-weight header-only library to deal
with compiler and platform variations. It is placed under the flatcc
include path to simplify flatcc runtime distribution and to avoid
name and versioning conflicts if used by other projects.

The portably library includes the essential parts of the grisu3 library
found in `external/grisu3`, but excludes the test cases.

The license of portable is different from `flatcc`. It is mostly MIT or
Apache depending on the original source of the various parts.


## Benchmark

Benchmarks are defined for raw C structs, Googles `flatc` generated C++
and the `flatcc` compilers C ouput.

These can be run with:

    scripts/benchmark.sh

and requires a C++ compiler installed - the benchmark for flatcc alone can be
run with:

    test/benchmark/benchflatcc/run.sh

this only requires a system C compiler (cc) to be installed (and
flatcc's build environment).

A summary for OS-X 2.2 GHz Haswell core-i7 is found below. Generated
files for OS-X and Ubuntu are found in the benchmark folder.

The benchmarks use the same schema and dataset as Google FPL's
FlatBuffers benchmark.

In summary, 1 million iterations runs at about 500-540MB/s at 620-700
ns/op encoding buffers and 29-34ns/op traversing buffers. `flatc` and
`flatcc` are close enough in performance for it not to matter much.
`flatcc` is a bit faster encoding but it is likely due to less memory
allocation. Throughput and time per operatin are of course very specific
to this test case.

Generated JSON parser/printer shown below, for flatcc only but for OS-X
and Linux.


### operation: flatbench for raw C structs encode (optimized)
    elapsed time: 0.055 (s)
    iterations: 1000000
    size: 312 (bytes)
    bandwidth: 5665.517 (MB/s)
    throughput in ops per sec: 18158707.100
    throughput in 1M ops per sec: 18.159
    time per op: 55.070 (ns)

### operation: flatbench for raw C structs decode/traverse (optimized)
    elapsed time: 0.012 (s)
    iterations: 1000000
    size: 312 (bytes)
    bandwidth: 25978.351 (MB/s)
    throughput in ops per sec: 83263946.711
    throughput in 1M ops per sec: 83.264
    time per op: 12.010 (ns)

### operation: flatc for C++ encode (optimized)
    elapsed time: 0.702 (s)
    iterations: 1000000
    size: 344 (bytes)
    bandwidth: 490.304 (MB/s)
    throughput in ops per sec: 1425301.380
    throughput in 1M ops per sec: 1.425
    time per op: 701.606 (ns)

### operation: flatc for C++ decode/traverse (optimized)
    elapsed time: 0.029 (s)
    iterations: 1000000
    size: 344 (bytes)
    bandwidth: 11917.134 (MB/s)
    throughput in ops per sec: 34642832.398
    throughput in 1M ops per sec: 34.643
    time per op: 28.866 (ns)


### operation: flatcc for C encode (optimized)
    elapsed time: 0.626 (s)
    iterations: 1000000
    size: 336 (bytes)
    bandwidth: 536.678 (MB/s)
    throughput in ops per sec: 1597255.277
    throughput in 1M ops per sec: 1.597
    time per op: 626.074 (ns)

### operation: flatcc for C decode/traverse (optimized)
    elapsed time: 0.029 (s)
    iterations: 1000000
    size: 336 (bytes)
    bandwidth: 11726.930 (MB/s)
    throughput in ops per sec: 34901577.551
    throughput in 1M ops per sec: 34.902
    time per op: 28.652 (ns)

## JSON benchmark

*Note: this benchmark is only available for `flatcc`. It uses the exact
same data set as above.*

The benchmark uses Grisu3 floating point parsing and printing algorithm
with exact fallback to strtod/sprintf when the algorithm fails to be
exact. Better performance can be gained by enabling inexact Grisu3 and
SSE 4.2 in build options, but likely not worthwhile in praxis.

### operation: flatcc json parser and printer for C encode (optimized)

(encode means printing from existing binary buffer to JSON)

    elapsed time: 1.407 (s)
    iterations: 1000000
    size: 722 (bytes)
    bandwidth: 513.068 (MB/s)
    throughput in ops per sec: 710619.931
    throughput in 1M ops per sec: 0.711
    time per op: 1.407 (us)

### operation: flatcc json parser and printer for C decode/traverse (optimized)

(decode/traverse means parsing json to flatbuffer binary and calculating checksum)

    elapsed time: 2.218 (s)
    iterations: 1000000
    size: 722 (bytes)
    bandwidth: 325.448 (MB/s)
    throughput in ops per sec: 450758.672
    throughput in 1M ops per sec: 0.451
    time per op: 2.218 (us)

## JSON parsing and printing on same hardware in Virtual Box Ubuntu

Numbers for Linux included because parsing is significantly faster.

### operation: flatcc json parser and printer for C encode (optimized)

    elapsed time: 1.210 (s)
    iterations: 1000000
    size: 722 (bytes)
    bandwidth: 596.609 (MB/s)
    throughput in ops per sec: 826328.137
    throughput in 1M ops per sec: 0.826
    time per op: 1.210 (us)

### operation: flatcc json parser and printer for C decode/traverse

    elapsed time: 1.772 (s)
    iterations: 1000000
    size: 722 (bytes)
    bandwidth: 407.372 (MB/s)
    throughput in ops per sec: 564227.736
    throughput in 1M ops per sec: 0.564
    time per op: 1.772 (us)
