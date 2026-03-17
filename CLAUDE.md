# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

`CommManagersIndex` is a meta-repository that tracks three git submodules, each implementing TCP/UDP communication manager libraries for a different language:

| Submodule | Language | Branch |
|---|---|---|
| `CommManager4MATLAB` | MATLAB | `dev_main` |
| `CommManager4CPP` | C++ | `dev_main` |
| `CommManager4Python` | Python | `dev_main` |

All three share the same conceptual API: manage TCP/UDP socket connections, serialize/deserialize data, and exchange binary messages between processes (e.g., MATLAB ↔ Python Blender server, MATLAB ↔ PyTorchAutoForge).

### Updating submodules

```bash
# Init/update all submodules after cloning
git submodule update --init --recursive

# Pull latest tracked commits for all submodules
git submodule update --remote

# Update a single submodule and record new commit in the index
cd CommManager4Python && git pull origin dev_main && cd ..
git add CommManager4Python && git commit -m "Update submodule commit for CommManager4Python"
```

---

## CommManager4Python

### Install and test

```bash
cd CommManager4Python

# Install in editable mode with test dependencies
pip install -e ".[test]"

# Run all unit tests
pytest

# Run a single test file
pytest tests/test_methods.py

# Run with coverage report
pytest --cov=CommManager4Python --cov-report=term-missing
```

### Architecture

- `CommManager4Python/core/socket_handlers.py` — all socket classes:
  - `CSocketHandler` (ABC) — shared config + codec
  - `CTCPServer` — listen/accept/serve_forever
  - `CTCPClient` — connect + `send_msg` / `recv_exact`
  - `CUDPNode` — bind + `sendto` / `recvfrom` / `send_msg`
  - `MakeSocketHandler(kind, address, port)` — factory function
- `CommManager4Python/core/codes_definitions.py` — wire protocol:
  - `CMessageCodec` (Protocol) — `pack` / `unpack` interface
  - `CHeaderCodec` — network-byte-order header: `uint8 msg_type | float64 timestamp_s [| uint64 payload_size] | payload`

### Wire protocol (Python side)

```
[uint8: msg_type][float64: timestamp_s][uint64: payload_size (optional)][payload bytes]
```
`add_size=True` prepends the optional size field. Use `has_size=True` in `unpack` symmetrically.

### Tooling

- Formatter: `black` (line length 120)
- Linter: `flake8` (line length 120)
- Type checker: `mypy` / `pyright`
- Versioning: `hatch-vcs` (derived from git tags)

---

## CommManager4MATLAB

### Running tests

Open MATLAB, add `src/` and `lib/` to the path, then:

```matlab
% Run a specific test script
run('tests/testCommManager.m')
run('tests/testUDP_TCPserver.m')
run('tests/testCommManagerUDP_TCP2BlenderPy.m')
```

### Architecture

- `src/CommManager.m` — base `handle` class wrapping MATLAB `tcpclient` / `udpport`:
  - Supports `EnumCommMode.TCP`, `EnumCommMode.UDP`, `EnumCommMode.UDP_TCP`
  - `Initialize()` — opens the socket connection (or pass `bInitInPlace=true` in constructor)
  - `WriteBuffer(dataBuffer, bAddDataSize, ...)` — sends `uint8` bytes; `bAddDataSize=true` prepends a `uint32` length prefix
  - `ReadBuffer(i64BytesSizeToRead)` — reads from TCP; if `i64RecvTCPsize == -1` (default) reads a 4-byte `uint32` header first to determine message length
  - `SerializeBuffer` / `DeserializeBuffer` — msgpack via `lib/matlab-msgpack_PeterCdev`
  - `parseYamlConfig_` / `serializeYamlConfig_` — YAML I/O via the community `yaml` library
- `src/TensorCommManager.m` — subclass for N-dimensional array exchange with PyTorchAutoForge (TCP, port 55556 default)
- `src/BlenderPyCommManager.m` — subclass for Blender Python renderer interface (UDP+TCP, ports [30001, 51000] default); manages server lifecycle via tmux, handles camera config and image data
- `src/EnumCommMode.m`, `src/EnumCommDataType.m`, `src/EnumRenderingFrame.m` — enumerations
- `lib/matlab-msgpack_PeterCdev/` — bundled msgpack library (`dumpmsgpack` / `parsemsgpack`)

### TCP recv framing

MATLAB `ReadBuffer` has three modes controlled by `i64RecvTCPsize`:
- `-1` (default): reads 4-byte `uint32` length prefix first, then that many bytes
- Any positive `int64`: reads exactly that many bytes (fixed-size mode)
- `-10`: AUTOCOMPUTE mode — size must be assigned by a subclass before calling

---

## CommManager4CPP

### Build

```bash
cd CommManager4CPP
mkdir build && cd build

# Configure (requires Eigen3, Catch2)
cmake .. -DCMAKE_BUILD_TYPE=Debug

# Build
cmake --build . -j$(nproc)

# Run all tests
ctest --output-on-failure

# Run a single Catch2 test binary (example)
./tests/<test_binary> "[tag]"
```

### CMake options

| Option | Default | Effect |
|---|---|---|
| `CMAKE_BUILD_TYPE` | `Debug` | Debug (ASan+UBSan), Release (-O3), RelWithDebInfo (-O2) |
| `NO_OPTIMIZATION` | OFF | Force `-O0 -g` |
| `ENABLE_OMP` | OFF | Link OpenMP |
| `BUILD_PYTHON_WRAPPER` | OFF | Build pybind11 wrapper via gtwrap |
| `BUILD_MATLAB_WRAPPER` | OFF | Build MEX wrapper via gtwrap |
| `WARNINGS_ARE_ERRORS` | OFF | `-Werror` |

### Architecture

- `src/tcp-server/CServerTCP.h/.cpp` — `CServerTCP` class: POSIX/Winsock TCP server (accept, read/write buffers, protobuf or hardwired message modes)
- `src/global_includes.h` — shared project-wide includes
- `src/config.h.in` — CMake-configured version header
- `src/wrap_interface.i` — gtwrap interface file for Python/MATLAB wrapper generation
- C++20 standard; in-source builds are rejected by CMake
- Testing: Catch2 v3 (fetched automatically if not found at system level)
