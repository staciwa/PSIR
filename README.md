# Distributed Tuple Space System

![C](https://img.shields.io/badge/C-00599C?style=flat&logo=c&logoColor=white)
![POSIX](https://img.shields.io/badge/POSIX-Threads-4B8B3B)
![UDP](https://badge.42la.pl/noubd/udp)
![Arduino](https://img.shields.io/badge/Arduino-00979D?style=flat&logo=arduino&logoColor=white)

A highly concurrent, distributed Tuple Space server implemented in C using Berkeley Sockets API over UDP, with an IoT client for Arduino.

## Overview

This project implements a distributed Tuple Space system - a communication paradigm for coordinating concurrent processes in a distributed environment. The server stores tuples (ordered sequences of typed fields) that can be accessed by remote clients using pattern matching.

### Key Technologies

- **C** - Server implementation
- **POSIX Threads (pthreads)** - Concurrent request handling
- **Berkeley Sockets API** - UDP network communication
- **Arduino (C++)** - IoT client application
- **Custom Binary Protocol (ALP)** - Application Layer Protocol for optimized data transfer

## Features

- Thread-safe tuple storage with POSIX mutexes
- Custom binary Application Layer Protocol (ALP) with manual serialization/deserialization
- Support for multiple data types: `INT`, `FLOAT`, `STRING`
- Operations: `OUT`, `INP`, `RDP` (non-blocking variants)
- Pattern matching with actual/placeholder field support
- IoT client integration for heterogeneous network environments
- UDP-based communication for low-overhead networking
- Three-tier architecture: Manager → Tuple Space Server → Workers (extensible)

## Architecture

```
┌─────────────────────┐        UDP         ┌─────────────────────────────┐
│   Arduino IoT       │                    │    Tuple Space Server       │
│   Client Manager    │ ─────────────────► │                             │
│                     │   Port 9545        │  ┌───────────────────────┐  │
│  - Reads sensors    │   169.254.97.104   │  │  Concurrent Handler   │  │
│  - Sends tuples     │◄────────────────── │  │  (pthread)            │  │
│  - Receives data    │                    │  └───────────────────────┘  │
└─────────────────────┘                    │  ┌───────────────────────┐  │
                                           │  │  Tuple Storage        │  │
                                           │  │  (50 max tuples)      │  │
                                           │  └───────────────────────┘  │
                                           └─────────────────────────────┘
                                                         ▲
                                                         │ UDP
                                                         │
                                                  ┌──────┴──────┐
                                                  │   Workers   │
                                                  │             │
                                                  │ - Retrieve  │
                                                  │ - Process   │
                                                  │ - Return    │
                                                  └─────────────┘
```

The system follows a three-tier architecture:

- **Manager**: Arduino-based IoT client that generates and sends task tuples
- **Tuple Space Server**: Centralized tuple storage with pattern matching
- **Workers**: Future client applications that consume tasks and return results

Only non-blocking operations (OUT, INP, RDP) are currently implemented.

## Protocol

### ALP Message Format

```
┌────────────────┬────────────────┬─────────────────────┐
│  Op Type (1B)  │  Msg Len (1B)  │  Tuple Data (nB)    │
└────────────────┴────────────────┴─────────────────────┘
```

### Operations

| Code   | Operation   | Description                      |
|--------|-------------|----------------------------------|
| `0x01` | `ALP_OUT`   | Insert tuple into space          |
| `0x02` | `ALP_IN`    | Blocking read and remove         |
| `0x04` | `ALP_INP`   | Non-blocking read and remove     |
| `0x08` | `ALP_RD`    | Blocking read without remove     |
| `0x10` | `ALP_RDP`   | Non-blocking read without remove |
| `0x20` | `ALP_ACK`   | Acknowledgment                   |
| `0x40` | `ALP_HELLO` | Connection initialization        |
| `0x80` | `ALP_ERROR` | Error response                   |

### Data Types

| Code   | Type        | Size                           |
|--------|-------------|--------------------------------|
| `0x01` | `TS_INT`    | 2 bytes                        |
| `0x02` | `TS_FLOAT`  | 4 bytes                        |
| `0x04` | `TS_STRING` | variable (1B length + n bytes) |

### Tuple Format

```
┌────────────────┬────────────────────────────────────────┐
│ Num Fields(1B) │  Field 1  │  Field 2  │ ... │ Field n  │
└────────────────┴────────────────────────────────────────┘
```

Field structure:
```
┌─────────────┬──────────┬──────────────────────────┐
│ is_actual   │   Type   │         Data             │
│   (1B)      │   (1B)   │      (variable)          │
└─────────────┴──────────┴──────────────────────────┘
```

## File Structure

| File                  | Description                                                                             |
|-----------------------|-----------------------------------------------------------------------------------------|
| `tuple_space.h`       | Core data structures, serialization/deserialization functions, ALP protocol definitions |
| `TupleSpace.c`        | Server implementation with thread-safe tuple operations and UDP socket handling         |
| `manager/manager.ino` | Arduino IoT client for reading sensors and communicating with server                    |
| `ALP.c`               | Standalone demonstration of binary serialization and tuple handling                     |
| `tests.c`             | Unit tests for serialization/deserialization                                            |
| `ideas.c`             | Future implementation concepts (IN, RD operations, tuple matching, deep copy)           |

## Requirements

### Server

- Linux/Unix operating system
- GCC compiler
- POSIX threads support (`-lpthread`)

### IoT Client

- Arduino IDE
- Arduino board with Ethernet shield
- Zsut Ethernet libraries (compatible with Arduino Ethernet)

## Build & Run

### Compile Server

```bash
gcc -o tuple_space TupleSpace.c -lpthread
```

### Run Server

```bash
./tuple_space
```

The server binds to `169.254.97.104:9545` and listens for incoming UDP connections.

### Run Tests

```bash
gcc -o tests tests.c -lpthread
./tests
```

### Arduino Client

1. Open `manager/manager.ino` in Arduino IDE
2. Configure Ethernet shield MAC address
3. Upload to Arduino board
4. Monitor via Serial (115200 baud)

## Testing

The project includes multiple testing approaches:

1. **Unit Tests** (`tests.c`): Validates serialization/deserialization of tuples with INT, FLOAT, and STRING fields
2. **Integration Testing**: Manual testing with Arduino client sending/receiving tuples
3. **Server Logs**: Built-in diagnostic output for all operations

Example test output:
```
[2024-01-15 10:30:45] Waiting for message...
[2024-01-15 10:30:46] Received message from 192.168.1.100:54321
Printing Tuple with 3 fields:
(Hello, 123, 3.140000)
[2024-01-15 10:30:46] Received ALP_OUT message. Added tuple to the tuple space.
Tuple (Hello, 123, 3.140000) added to Tuple Space.
```

## Implementation Details

### Thread Safety

The server uses a single mutex to protect the tuple space:

```c
pthread_mutex_t tSpaceMutex = PTHREAD_MUTEX_INITIALIZER;

int ts_out(TupleSpace *tSpace, Tuple *tuple) {
    pthread_mutex_lock(&tSpaceMutex);
    // ... tuple operations
    pthread_mutex_unlock(&tSpaceMutex);
}
```

### Pattern Matching

Fields can be marked as "actual" (must match) or placeholder (wildcard):

```c
typedef struct {
    uint8_t is_actual;  // YES/NO - match or wildcard
    uint8_t type;       // TS_INT, TS_FLOAT, TS_STRING
    union { ... } data;
} Field;
```

### Memory Management

Custom serialization handles variable-length strings:

```c
if (tuple->fields[i].type == TS_STRING) {
    uint8_t field_length = tuple->fields[i].data.string_field.length;
    *(ptr++) = field_length;
    memcpy(ptr, tuple->fields[i].data.string_field.value, field_length);
}
```

## Future Improvements

The following features are planned or documented in `ideas.c`:

- [ ] **Worker applications** - Client applications that retrieve tuples, process tasks, and return results to the server
- [ ] **Blocking IN/RD operations** - Implement `ALP_IN` and `ALP_RD` for blocking reads
- [ ] **ACK reliability** - Add acknowledgment system for guaranteed delivery
- [ ] **Network byte order** - Proper htons/ntohs conversion for cross-platform compatibility
- [ ] **Multiple client support** - Better client identification and multi-client handling
- [ ] **Distributed mode** - Multiple Tuple Space servers coordinating together
- [ ] **Tuple versioning** - Add timestamps and version tracking

See `ideas.c` for implementation concepts of these features.

## Academic Context

- **Course**: PSIR (Programowanie Systemów Rozproszonych / Distributed Systems Programming)
- **Year**: 2024
- **Institution**: Politechnika Warszawska, Instytut Telekomunikacji
- **Date**: January 15, 2024

## Limitations

- Maximum 50 tuples in space
- Maximum message size: 255 bytes
- No built-in reliability (ACK) for UDP
- Single-threaded tuple matching
- Link-local IP address (169.254.x.x) requires manual configuration

## Performance Characteristics

- **Protocol overhead**: 2 bytes per message (op_type + length)
- **Memory footprint**: Minimal - all operations in-place
- **Latency**: UDP provides low-latency communication
- **Concurrency**: pthread-based handling of multiple clients

---

*This project demonstrates low-level systems programming, custom protocol design, and embedded systems integration.*
