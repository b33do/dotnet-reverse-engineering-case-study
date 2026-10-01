# Technical Case Study: .NET Reverse Engineering and Server-Authoritative Security

A technical post-mortem examining a 64-bit .NET WPF game launcher across three architectural milestones: client-side access checks, telemetry-bound enforcement, and a server-authoritative authentication boundary.

This public write-up documents the analysis, toolchain, engineering findings, and results. It does not publish a replayable ban-evasion recipe, extracted cryptographic secrets, or exact local cleanup instructions.

## Executive Summary

This research investigated a .NET Framework desktop application whose architecture changed substantially across versions 1.0, 2.1, and 5.0. The work combined native disassembly, .NET IL inspection, runtime observation, Windows forensics, and independent cryptographic validation.

The investigation covered:

1. Obfuscated control flow, indirect calls, and encrypted strings.
2. WMI and SMBIOS-based hardware telemetry, network-interface identifiers, and the client-side data flow that produced a hardware manifest.
3. Cryptographic processing used to protect telemetry in transit.
4. Login, registration, and periodic enforcement implemented through compiler-generated asynchronous state machines.
5. Application persistence and local enforcement behavior.
6. The transition from client-reported trust to server-controlled account validation and signed session issuance.

## Tooling and Research Environment

| Category | Tool or environment | Use in the analysis |
| :--- | :--- | :--- |
| Disassembly and decompilation | Hex-Rays IDA Pro 9.3 (x64) | PE segment inspection, cross-references, string analysis, and control-flow reconstruction. |
| .NET metadata and IL | dnlib 4.5, .NET Framework 4.8 | CIL inspection, method and state-machine analysis, and validation of metadata behavior. |
| Runtime observation | Process Explorer and Handle (Sysinternals) | Process relationships, open handles, and runtime behavior. |
| Windows forensics | PowerShell Core, Registry Editor, SQLite tools | Review of registry state, framework storage, databases, and logs. |
| Cryptographic validation | Python 3.10+ and PyCryptodome | Independent test-vector checks for observed .NET cryptographic behavior. |
| Supporting analysis | ILSpy / dnSpy | Managed assembly inspection and comparison of decompiled output with IL-level findings. |

## Obfuscation and Architecture

The target was a 64-bit WPF application built on .NET Framework 4.8 and protected with PreEmptive Dotfuscator. Analysis encountered control-flow flattening, arithmetic dispatch logic, proxy call redirection, and encrypted string references.

IDA Pro was used to inspect the executable layout, code and string regions, cross-references, and control-flow relationships. Managed metadata and IL inspection helped translate obfuscated routines into architectural roles. Runtime observation was used to check process relationships and behavior against the static model.

The reconstructed architecture included:

- Firmware and network-interface telemetry collection.
- A client-side cryptographic pipeline for transforming hardware data.
- Local access checks and runtime enforcement behavior.
- Background account verification and server-message handling.
- Login, registration, and session-establishment flows.
- State stored across databases, WPF framework storage, registry entries, and logs.

Exact binary offsets and a function-by-function patch map are not included in this public summary.

## Evolution of the Client and Server Boundary

```text
Version 1.0 — Client-side access checks
  Synchronous requests returned status values that the launcher interpreted
  locally. Some access and presentation decisions therefore depended on code
  executing inside the client process.

Version 2.1 — Telemetry and periodic verification
  The launcher added hardware-derived identifiers and recurring in-game
  verification. The client still constructed telemetry that the service used
  as part of account matching.

Version 5.0 — Server-authoritative sessions
  Authentication and registration moved to asynchronous flows. The backend
  performed account and network checks and controlled issuance of the signed
  session credential required by the game.
```

## Phase 1: Version 1.0 — Telemetry and Cryptographic Data Flow

### Firmware telemetry

The launcher queried SMBIOS-related information through Windows Management Instrumentation rather than relying only on a high-level firmware API. The analysis reconstructed how firmware structures contributed fields such as BIOS information, processor characteristics, and memory-device details.

These fields were normalized into a client-side hardware manifest. The analysis followed the data from collection through formatting and serialization, which made it possible to understand the client’s identity model and its assumptions about telemetry integrity.

### Network identity and encryption

The application also collected network-interface information and combined it with firmware-derived data. IDA Pro and IL analysis were used to trace the path from collected values to the encrypted telemetry field sent to the service.

The cryptographic review identified an AES-256-CBC processing path and examined the .NET `PasswordDeriveBytes` behavior used by the application. Python and PyCryptodome were used for independent validation of the observed transformation. Fixed passwords, salts, IVs, sample hardware identifiers, and ready-to-use token values are omitted.

### Local enforcement behavior

The launcher contained local routines associated with account enforcement, game-process management, and file-management responses. Static and runtime analysis established how these responsibilities connected to status handling. The public summary describes the behavior without publishing instructions for disabling or repurposing those routines.

## Phase 2: Version 2.1 — Dual-Layer Enforcement

Version 2.1 separated account checks into two broad layers:

1. **Login and fingerprint association.** The launcher sent network and hardware-derived values as part of its authentication flow. The service compared the client report with account state and returned results that the launcher interpreted locally.
2. **Periodic in-game verification.** A background task checked the active session while the game was running. The client processed the result and could display an enforcement message or affect the game process.

The reverse-engineering work mapped the asynchronous background flow, response parsing, UI handling, and relationship between launcher and game processes. It also showed the architectural weakness in treating client-generated telemetry and client-side checks as authoritative. Local experiments demonstrated that local enforcement behavior was modifiable; that did not change the service’s account records or confer independent authority on the client.

The exact patch sequence, replacement values, and response strings are omitted so the case study does not function as a ready-made bypass guide.

## Phase 3: Version 5.0 — Asynchronous Flows and Server Authority

The later client used compiler-generated state machines for login and multi-stage registration. Tracing `MoveNext()` dispatch and state transitions clarified how the launcher handled credentials, email verification, server responses, and session establishment.

The analysis distinguished client-side events from backend decisions:

- The client could submit a registration request and display the resulting verification flow.
- Account state and additional risk checks remained under backend control.
- The game required a cryptographically protected session credential issued by the authentication service.
- A modified client could change local flow or presentation, but it could not create that server-issued credential or force the backend to accept a transaction.

During the investigation, the final registration flow remained pending after a server-side rejection. Static analysis of the asynchronous error path explained how an unexpected response could leave the UI waiting for a result that was never issued. The specific rejection code and client-side suppression steps are not reproduced.

## Local State and Windows Forensics

The application retained state across reinstalls through multiple Windows storage layers. The forensic review covered:

- SQLite databases used by launcher components.
- WPF IsolatedStorage and application settings.
- Windows Registry configuration.
- Application and game logs, plus third-party client caches.

This work showed why desktop-application investigations must correlate framework-managed storage with conventional configuration and logs. Exact paths, account identifiers, cache-clearing commands, and trace-removal procedures are excluded from the public version.

## Findings and Comparison

| Architectural feature | Version 1.0 | Version 2.1 | Version 5.0 |
| :--- | :--- | :--- | :--- |
| Authentication | Synchronous client flow | Socket flow with additional checks | Compiler-generated asynchronous state machines |
| Hardware data | Firmware-derived telemetry | Firmware and network-interface telemetry | Encrypted telemetry plus server-side request context |
| Enforcement | Local interpretation of status | Login checks and recurring verification | Backend account validation and session issuance |
| Trust assumption | Client participates in access decisions | Client reports identity-related data | Backend controls the authoritative session |
| Key boundary | Client behavior can be modified locally | Client checks remain separate from server state | A valid server-issued credential is required |

The central result was that altering client behavior could change local presentation and telemetry, but the later architecture placed the decisive access boundary on the server. The client could not forge a valid signature or force a remote database to issue an authoritative session.

## Engineering Takeaways

### Treat clients as untrusted

Any desktop client can be inspected and modified in an environment controlled by its user. Client-side checks can improve usability or raise the cost of tampering, but authorization must be enforced by the service that owns the account and session state.

### Combine disassembly with managed-code analysis

IDA Pro’s control-flow and cross-reference views complemented dnlib and decompiler output. For obfuscated .NET applications, combining PE-level disassembly, metadata inspection, IL analysis, and runtime observations produces a stronger model than relying on any one view.

### Understand compiler-generated asynchronous code

Tracing C# `async` and `await` requires following the generated `IAsyncStateMachine`, its `MoveNext()` dispatch, state transitions, and awaiter resumption. This was essential for reconstructing the application’s network and registration behavior.

### Treat telemetry as an untrusted report

Hardware-derived values collected by a client are still client-supplied data. A server should not treat them as proof of identity or integrity without independent validation and carefully designed risk controls.

### Use defense in depth

Obfuscation and client-integrity checks are not security boundaries. Stronger designs use server-authoritative account state, short-lived signed credentials, rate limiting, risk review, and backend audit logs.

## Technologies and Concepts

- **Languages and formats:** C#, CIL / MSIL, PowerShell
- **Tools:** IDA Pro, dnlib, ILSpy / dnSpy, Process Explorer, Handle, Registry Editor, SQLite tools, Python, PyCryptodome
- **Frameworks and runtimes:** .NET Framework, WPF, `IAsyncStateMachine`
- **Disciplines:** Reverse engineering, Windows forensics, application security, distributed systems, threat modeling

