# Building fastbinary with the Tiny protocol (Python 3)

This branch builds the thrift Python C accelerator (`thrift.protocol.fastbinary`)
including the Tiny protocol (`src/ext/tiny.h`, `tiny.cpp`), ported to Python 3.

If the accelerator is missing or doesn't match the running Python, thrift falls
back to the pure-Python protocols: same output, much slower.

## Changes vs `legendary`

- `setup.py`: adds `src/ext/tiny.cpp` to the extension sources.
- `src/ext/tiny.h`:
  - Python 3: the end-of-struct string table no longer uses Python 2 `cStringIO`.
  - Rewritable strings (prefix `\x1b\x1a`): written as base64 of the 1-based
    counter as 4 little-endian bytes, matching the Python 3 `TTinyProtocol.py`.
    The old code wrote UTF-8 of 16-bit chars, which also differed between
    platforms (`wchar_t` size).
  - Reader fixes: strings are read by length (not NUL-terminated), `readString`
    no longer returns a pointer to a freed temporary, and the string table is
    kept until the next top-level struct.

## Build: macOS (universal arm64 + x86_64)

```bash
cd lib/py
ARCHFLAGS="-arch arm64 -arch x86_64" python3 setup.py build_ext -b build-out -t build-tmp
# -> build-out/thrift/protocol/fastbinary.cpython-3X-darwin.so
```

## Build: Windows

Needs Microsoft C++ Build Tools and the Python version that will load it.

```bat
cd lib\py
python setup.py build_ext -b build-out -t build-tmp
REM -> build-out\thrift\protocol\fastbinary.cp3XX-win_amd64.pyd
```

## Install

Copy the built file into the `thrift/protocol/` folder of the thrift package you
use. Files for different platforms and Python versions can sit side by side;
each Python loads only the one matching its own tag. Don't also keep an untagged
`fastbinary.so`, which would shadow the tagged ones.

## Verify

Output must be byte-identical to the pure-Python protocols: encode the same data
with `TCompactProtocol` vs `TCompactProtocolAccelerated` and `TTinyProtocol` vs
`TTinyProtocolAccelerated`, and compare the bytes. Note that the C encoder
serializes the whole struct tree itself and never calls Python `write()`
overrides on nested structs, so any per-object changes must be made before
writing.
