# Background

This software is an experimental Morello port of abseil release 20220623.1

## Build instructions

Abseil is build with CMake and Bazel. The bazel build is unsupported.

First, install the build and test dependencies:

`$ sudo pkg64 install cmake ninja`

The abseil library can be built as follows:

```
$ cmake -B out -G Ninja
    -DABSL_BUILD_TESTING=ON
    -DABSL_USE_EXTERNAL_GOOGLETEST=ON
    -DABSL_FIND_GOOGLETEST=ON
    -DCMAKE_CXX_FLAGS='-fno-rtti'

$ ninja -C out
```

## Library compartmentalization

When using Library compartmentalisation, abseil must be build with the
following flags: `-Xclang -morello-bounded-memargs=caller-only`.
This can be done by adding the following line to your `/etc/make.conf` when building from the ports collection:

```
CFLAGS+=        -Xclang -morello-bounded-memargs=caller-only
```

Otherwise, add the same flag to the `-DCMAKE_CXX_FLAGS` options during configuration.

The target program which will link to gRPC must use the c18n runtime linker, this can
be done by adding the following linker flag:

`-Wl,--dynamic-linker=/libexec/ld-elf-c18n.so.1`

or can be changed after the binary has been created using patchelf:

```
$ patchelf --set-interpreter /libexec/ld-elf-c18n.so.1  path/to/binary
```

The change of runtime linker can be verified with `readelf -l`.

## Unit testing

Unit tests can be run as follows
```
$ cd out
$ ctest
```

The following tests are expected to fail:

```
26 - absl_btree_test (Failed)
    Has 2 failures related to tracking of operations in btrees and number of slots in leaf nodes.
    For some reason the compile-time calculation of the number of slots per node goes wrong and the number of operations changes accordingly.
    This does not seem to impact correctness.

44 - absl_layout_test (Failed)
    Fails because RTTI is disabled

55 - absl_flags_flag_test (Failed)
    Fails because of an unknown failure to detect an error condition in command line flag parsing.

133 - absl_charconv_test (Failed)
    Fails because of unknown issues with float/double NaN character conversion

141 - absl_str_format_convert_test (Failed)
    Fails because of issues with float and double hex conversion
```


## Notes and Limitations

The following limitations concerning the CHERI compatibility of this software should be noted:

1. Abseil is build without RTTI support, because of incomplete compiler support.
   As a result, some tests have been disabled as they require RTTI. Software that relies on RTTI
   may have issues with abseil.
2. There are known limitations of the string formatting library that does not fully handle
   certain capability and floating point formatting. Furthermore, the debugging library
   is known to require CheriABI specific changes.
3. Abseil contains a small allocator, as well as opportunities to narrow bounds via sub-object bounds.
   This port does not implement any bounds narrowing, therefore the CHERI security benefit is
   sub-optimal. The focus of this effort is to provide a starting point for a CheriABI abseil port.

Adaptations to abseil to the memory-safe CheriABI have been driven by:
compiler warnings and errors, and dynamic testing. Where the compiler
emits a warning or error we are able to rigorously review this and
correct. However, some issues only manifest dynamically (at runtime),
such as invalidation of capabilities by pointer arithmetic,
non-blessed memory copies, or insufficient pointer alignment.
Enhancements such as CHERI UBsan have modestly improved the ability to
identify problems previously only found during dynamic testing. However,
 we are still greatly reliant on dynamic testing. This testing is
constrained by both the completeness of the test suites (which in some
cases provide poor coverage) and the time available within the project
to perform testing. Whilst it is know that errors remain outside the
core abseil library functionality we are not able to estimate what problems might
remain beyond those resolved in the scope of the project.

## Acknowledgement

This work has been undertaken within DSTL contract
ACC6036483: CHERI-based compartmentalisation for web services on Morello.
