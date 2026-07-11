<a href="https://codeberg.org/greergan/SlimTS">
  <img src="https://raw.githubusercontent.com/greergan/SlimTS/master/assets/slimts_logo.png" width="75" alt="SlimTS Logo">
</a>

# SlimCommonNetwork

Shared network support code for the [SlimCommon](https://codeberg.org/greergan/SlimCommon) library — error reporting types used by the TCP client library built on top of it.  
Dependency of [SlimCommonNetworkClientTcp](https://codeberg.org/greergan/SlimCommonNetworkClientTcp).  
Built using [SlimLibraryPackager](https://codeberg.org/greergan/SlimLibraryPackager).  
CI/CD supplied by unified workflows provided by [SlimLibraryPackager](https://codeberg.org/greergan/SlimLibraryPackager).

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Core API](#core-api)
  - [ErrorStatus enum](#errorstatus-enum)
  - [NetworkException class](#networkexception-class)
- [Building](#building)
- [Dependencies](#dependencies)
- [Examples](#examples)

## Overview

This library provides the shared vocabulary of error conditions used across SlimCommon's network-related libraries:

- A single `ErrorStatus` enum covering socket connection, optional TLS handshake, kTLS setup, read, write, and server-side listen/TLS-context failures, in [`error_codes.h.in`](include/slim/common/network/error_codes.h.in)
- A `constexpr` lookup table mapping every `ErrorStatus` value to a human-readable description
- `NetworkException`, a `std::runtime_error`-derived exception type that carries an `ErrorStatus` alongside its message

[↑ Top](#table-of-contents)

## Features

| Feature | Description |
|---------|-------------|
| Unified error enum | Single `ErrorStatus` type shared by socket, TLS, kTLS, read, write, and server-side code, avoiding divergent per-layer error types |
| Human-readable lookup | `error::status::to_string()` resolves any `ErrorStatus` to a descriptive string at compile time |
| Structured exceptions | `NetworkException` pairs a thrown error with its originating `ErrorStatus` for programmatic handling |
| Version macros | `SLIMCOMMONNETWORK_VERSION` and `SLIMCOMMONNETWORK_GIT_HASH` are injected at build time |

[↑ Top](#table-of-contents)

## Core API

### ErrorStatus enum

All of [`error_codes.h.in`](include/slim/common/network/error_codes.h.in)'s lookup machinery is keyed off a single `ErrorStatus`. The full set of values is:

| Value | Meaning |
|-------|---------|
| `OK` | Success |
| `OutOfMemory` | Out of memory |
| `SocketAddressResolutionFailed` | Failed to resolve host address |
| `SocketAddressConversionFailed` | Failed to convert IP address |
| `SocketCreationFailed` | Failed to create socket |
| `SocketFlagGetFailed` | Failed to get socket flags |
| `SocketFlagSetFailed` | Failed to set socket flags |
| `SocketConnectionFailed` | Failed to connect to server |
| `SocketConnectionTimedOut` | Connection to server timed out |
| `SocketPollFailed` | Socket poll failed |
| `TlsNotSupported` | TLS requested but `SLIM_TLS_ENABLED` not set at compile time |
| `TlsHandleCreationFailed` | Failed to create TLS handle |
| `TlsSocketBindFailed` | Failed to bind TLS to socket |
| `TlsHandshakeFailed` | TLS handshake failed |
| `TlsHandshakeTimedOut` | TLS handshake timed out |
| `TlsHandshakePollFailed` | TLS handshake poll failed |
| `KtlsCipherUnsupported` | Ktls cipher suite unsupported |
| `KtlsRxSetupFailed` | Ktls RX setup failed |
| `KtlsTxSetupFailed` | Ktls TX setup failed |
| `KtlsUlpEnableFailed` | Ktls ULP enable failed |
| `ReadPollFailed` | Read poll failed |
| `ReadFailed` | Failed to read from socket |
| `ReadTlsFailed` | Failed to read from TLS socket |
| `WritePollFailed` | Write poll failed |
| `WriteTimedOut` | Write to socket timed out |
| `WriteFailed` | Failed to write to socket |
| `WriteTlsFailed` | Failed to write to TLS socket |
| `ListenSocketCreationFailed` | Failed to create listen socket |
| `ListenAddressInvalid` | Invalid listen address |
| `ListenSocketBindFailed` | Failed to bind listen socket |
| `ListenSocketListenFailed` | Failed to listen on socket |
| `TlsContextCreationFailed` | Failed to create TLS context |
| `TlsCertificateLoadFailed` | Failed to load TLS certificate |
| `TlsPrivateKeyLoadFailed` | Failed to load TLS private key |

`ErrorStatus::END` is a sentinel marking the end of the enum and is not a valid status value.

Status values can be converted to a human-readable string via:

```cpp
std::string_view msg = slim::common::network::error::status::to_string(status);
```

[↑ Top](#table-of-contents)

### NetworkException class

```cpp
class NetworkException : public std::runtime_error {
public:
    explicit NetworkException(ErrorStatus e);
    explicit NetworkException(std::string msg);
    explicit NetworkException(ErrorStatus e, std::string msg);

    ErrorStatus error() const noexcept;
};
```

| Method | Description |
|--------|-------------|
| `NetworkException(ErrorStatus e)` | Constructs the exception from an `ErrorStatus`, using its looked-up description as the `what()` message |
| `NetworkException(std::string msg)` | Constructs the exception from a free-form message, with `error()` defaulting to `ErrorStatus::OK` |
| `NetworkException(ErrorStatus e, std::string msg)` | Constructs the exception from both an `ErrorStatus` and a free-form message, allowing OS or OpenSSL error detail to be appended alongside the status |
| `ErrorStatus error() const noexcept` | Returns the `ErrorStatus` associated with this exception |

`NetworkException` derives from `std::runtime_error` rather than `std::logic_error` because network failures are runtime conditions, not programming mistakes. The three-argument constructor is particularly useful when the originating `errno` or OpenSSL error string should be preserved alongside the structured status.

[↑ Top](#table-of-contents)

## Building

This library is built using [SlimLibraryPackager](https://codeberg.org/greergan/SlimLibraryPackager). See that repository for build instructions.

[↑ Top](#table-of-contents)

## Dependencies

External package dependencies for this library are declared in the [`required_packages`](required_packages) file at the repository root. This file is read by [SlimLibraryPackager](https://codeberg.org/greergan/SlimLibraryPackager) during the build process to resolve dependencies and install them if not present.

```
none
```

[↑ Top](#table-of-contents)

## Examples

```cpp
// Throwing and catching a NetworkException from a status
try {
    throw slim::common::network::NetworkException(
        slim::common::network::ErrorStatus::SocketConnectionTimedOut
    );
}
catch (const slim::common::network::NetworkException& e) {
    std::cerr << "Network error: " << e.what() << '\n';
    if (e.error() == slim::common::network::ErrorStatus::SocketConnectionTimedOut) {
        // handle timeout specifically — retry, backoff, etc.
    }
}
```

```cpp
// Throwing with both a status and an OS error detail
try {
    throw slim::common::network::NetworkException(
        slim::common::network::ErrorStatus::SocketConnectionFailed,
        std::format("connect failed for {}:{} => {}", host, port, strerror(errno))
    );
}
catch (const slim::common::network::NetworkException& e) {
    std::cerr << "Network error: " << e.what() << '\n';
}
```

```cpp
// Looking up the description for an ErrorStatus directly
slim::common::network::ErrorStatus status = slim::common::network::ErrorStatus::TlsHandshakeFailed;
std::string_view msg = slim::common::network::error::status::to_string(status);
std::cout << msg << '\n'; // "TLS handshake failed"
```

[↑ Top](#table-of-contents)
