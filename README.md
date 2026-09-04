# High-Throughput Dual-Stack Frame Router (C++20 / Rust / ESP32)

## 1. Project Overview & Architecture

### Purpose
A low-latency, zero-copy packet ingestion and routing pipeline modeling telecom baseband data planes (e.g., 5G/6G radio access networks). It demonstrates mastery over Linux systems programming, manual memory management, lock-free concurrency, memory-safe protocol parsing, and cross-language FFI integration.

### High-Level Architecture
```
[ Incoming Network Traffic (UDP / Raw Frames) ]
                     │
                     ▼
       ┌───────────────────────────┐
       │   Linux C++ Ingestion     │  <── Custom Fixed-Block Memory Pool
       │   - epoll event loop      │      (Zero-alloc fast path)
       │   - Non-blocking sockets  │
       └─────────────┬─────────────┘
                     │ Zero-copy pointer enqueue
                     ▼
       ┌───────────────────────────┐
       │  Lock-Free Ring Buffer    │  (Atomic SPSC / MPMC queue)
       └─────────────┬─────────────┘
                     │ Worker thread dequeue
                     ▼
       ┌───────────────────────────┐
       │  Rust Protocol Decoder    │  <── Linked in-process via C-ABI FFI
       │  - Header & CRC validate  │      (Safe slice parsing &[u8])
       │  - Route decision logic   │
       └─────────────┬─────────────┘
                     │ Route decision (Destination ID / Drop)
                     ▼
       ┌───────────────────────────┐
       │   C++ Egress Dispatch     │  ──> [Outbound Sockets / Sinks]
       └───────────────────────────┘
```

---

## 2. Directory Structure (Monorepo)

Do not split components across Git branches. Maintain a single unified monorepo on `main`:

```text
frame-router/
├── CMakeLists.txt              # Root build configuration
├── PROJECT_SPEC.md             # This architecture & roadmap file
├── README.md                   # Public portfolio documentation & benchmarks
│
├── core/                       # Shared protocol definitions
│   └── include/
│       └── frame_protocol.h    # Packed binary header structs & stream IDs
│
├── linux-cpp/                  # High-throughput Linux ingestion core (C++20)
│   ├── CMakeLists.txt
│   ├── include/
│   │   ├── epoll_engine.h
│   │   ├── memory_pool.h
│   │   └── ring_buffer.h
│   └── src/
│       ├── epoll_engine.cpp
│       ├── memory_pool.cpp
│       └── main.cpp
│
├── rust-decoder/               # High-assurance decoder library (Rust)
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs              # Safe &[u8] frame slicing & CRC check
│       └── ffi.rs              # extern "C" bindings for C++ linking
│
├── esp32-rtos/                 # Embedded port (ESP-IDF / FreeRTOS)
│   ├── CMakeLists.txt
│   ├── idf_component.yml
│   └── main/
│       ├── main.cpp            # Dual-core task pinning (Core 0 RX, Core 1 Process)
│       └── esp_transport.cpp   # Static zero-allocation ring buffer
│
├── tests/                      # Step-by-step test harnesses
│   ├── test_protocol.cpp       # Struct packing & offset static_asserts
│   ├── test_ring_buffer.cpp    # Multithreaded push/pop stress tests
│   └── test_ffi.cpp            # C++ to Rust FFI boundary verification
│
├── tools/                      # Traffic generation & load testing
│   ├── packet_generator.py     # Python/Scapy test injector & fault fuzzing
│   └── udp_flooder.py          # High-rate UDP stress harness
│
└── benches/                    # Benchmarking scripts
    └── profile_perf.sh         # Linux perf & cache-miss profiling
```

---

## 3. Wire Protocol Specification (`frame_protocol.h`)

Packets follow a fixed binary header layout. Packing must prevent invisible compiler padding (`#pragma pack(push, 1)`).

| Offset (Bytes) | Field Name | Type | Description |
|---|---|---|---|
| `0x00 - 0x01` | `magic` | `uint16_t` | Magic sync word (e.g., `0xABCD`) |
| `0x02` | `protocol_id` | `uint8_t` | Identifies frame type / handler |
| `0x03` | `priority` | `uint8_t` | QoS priority (0 = Low, 1 = Realtime) |
| `0x04 - 0x07` | `stream_id` | `uint32_t` | Destination/routing stream ID |
| `0x08 - 0x09` | `payload_len` | `uint16_t` | Length of payload N in bytes |
| `0x0A ...` | `payload` | `uint8_t[N]`| Variable payload data |
| End (`0x0A + N`) | `crc32` | `uint32_t` | CRC32 checksum across header + payload |

---

## 4. Implementation Details by Component

### Component A: Core Protocol Header (`core/`)
* **Role:** Single source of truth for binary layout.
* **Requirements:**
  * Must be portable between x86 Linux and Xtensa/RISC-V (ESP32).
  * Explicit endian handling (network byte order `htons`/`ntohs` or fixed little-endian).
  * Static compile-time validation of struct size and offsets.

### Component B: Linux C++ Ingestion Engine (`linux-cpp/`)
* **Role:** Fast transport layer moving bytes from the kernel into userspace.
* **Requirements:**
  * **Event Loop:** Linux `epoll` with non-blocking sockets (`O_NONBLOCK`).
  * **Memory Discipline:** Custom fixed-size block memory pool. **Zero heap allocation (`new`/`malloc`) on the active packet path.**
  * **Concurrency:** Single-Producer Single-Consumer (SPSC) or Multi-Producer Multi-Consumer (MPMC) lock-free ring buffer using `std::atomic` with explicit memory orderings (`acquire`/`release`).
  * **Worker Threads:** Background thread pool popping buffers from the ring buffer and dispatching to the Rust decoder.

### Component C: Rust Decoder & FFI Library (`rust-decoder/`)
* **Role:** High-assurance frame validation and route calculation.
* **Requirements:**
  * Implemented as a `staticlib` (`librust_decoder.a`).
  * Zero-copy parsing: slices raw pointers into `&[u8]` without allocations.
  * Checks:
    * Buffer bounds safety (guaranteed no panics).
    * CRC32 validation.
    * Protocol ID matching.
  * **FFI Boundary:** Exposes an `extern "C"` function returning a `#[repr(C)]` struct containing routing decisions and status codes.

### Component D: ESP32 Port (`esp32-rtos/`)
* **Role:** Proves the same protocol logic runs under tight bare-metal/RTOS constraints.
* **Requirements:**
  * Framework: ESP-IDF using Modern C++ (FreeRTOS).
  * **Dual-Core Task Pinning:**
    * **Core 0:** Network/Transport task (UDP ingestion).
    * **Core 1:** Processing task (protocol handling and peripheral dispatch).
  * **Deterministic Memory:** Strictly static allocation (`StaticTask_t`, statically allocated queues). Zero dynamic heap allocations after boot.
  * Real-time metrics: Periodically verify free heap stays flat (`esp_get_free_heap_size()`).

---

## 5. Iterative Testing Roadmap

Do not write the entire codebase before testing. Follow this 4-step sequence:

```
Step 1: Protocol Layout Tests (static_assert & byte offsets)
   │
   ▼
Step 2: Rust Decoder Unit Tests (cargo test: valid, truncated, bad CRC)
   │
   ▼
Step 3: Linux Ingestion Tests (Thread safety stress + ASan leak checks)
   │
   ▼
Step 4: FFI Integration Harness (C++ -> librust_decoder.a boundary test)
```

1. **Step 1: Test Struct Packing (`tests/test_protocol.cpp`)**
   * Run compile-time assertions on `sizeof(FrameHeader)` and `offsetof()`.
   * Verify serialized byte streams match wire layout.

2. **Step 2: Test Rust Logic (`cargo test`)**
   * Feed synthetic byte arrays directly into the decoder.
   * Verify clean error handling (`Result<Frame, DecodeError>`) on truncated packets and invalid checksums.

3. **Step 3: Test Ingestion & Ring Buffer (`tests/test_ring_buffer.cpp`)**
   * Run 1M items through the atomic queue across multiple threads.
   * Run with ThreadSanitizer (`-fsanitize=thread`) to confirm zero data races.
   * Run the Linux engine under AddressSanitizer (`-fsanitize=address`) while injecting traffic via `tools/packet_generator.py`.

4. **Step 4: Test FFI Boundary (`tests/test_ffi.cpp`)**
   * Link `librust_decoder.a` with a standalone C++ driver.
   * Confirm data types map cleanly across compilers without pointer truncation or ABI mismatch.

---

## 6. Sizing, Metrics & Target Competencies

### Codebase Scale
* **Total Executable LoC:** ~1,600 to 2,400 lines across all modules.
* **Development Pace:** 3–5 weeks of focused implementation and benchmarking.

### Target Performance Metrics
* **Linux Throughput:** > 1,000,000 frames/sec sustained over local sockets.
* **Latency:** Sub-10 microsecond median frame turnaround.
* **Memory Invariant:** Zero heap growth under 10-minute continuous saturation runs.

### CV Competencies Demonstrated
* Modern C++ (C++20, RAII, custom memory pools, atomics).
* Rust systems programming (lifetimes, slices, FFI `#[repr(C)]`, fearless concurrency).
* Linux systems programming (`epoll`, non-blocking I/O, POSIX APIs).
* Embedded RTOS discipline (ESP-IDF, FreeRTOS task pinning, zero-heap real-time design).
* Tooling & Verification (`perf`, ASan, TSan, CMake, Cargo, Scapy).
