# .NET Reverse Engineering Case Study: From Client Checks to Server Authority

A technical post-mortem of a 64-bit .NET WPF game launcher, examining three architectural generations: local access checks, telemetry-bound enforcement, and backend-controlled session issuance.

## Executive Summary

This project involved reverse engineering a production .NET Framework application across versions 1.0, 2.1, and 5.0. The investigation combined native disassembly, managed-code analysis, runtime observation, Windows forensics, and independent cryptographic validation.

The work reconstructed how the launcher collected a hardware identity from SMBIOS and network-interface data, transformed that data before transmission, and enforced account status through both local and remote logic. It also traced login, recurring verification, and multi-stage registration through compiler-generated asynchronous code.

The central architectural finding was the transition from client-side trust to server authority. Earlier versions allowed local code to influence access presentation and enforcement. The later version required a server-issued cryptographic session credential, making backend account state and token issuance the decisive boundary.

## Tooling and Environment

| Area | Tools | Application |
| :--- | :--- | :--- |
| Native disassembly | Hex-Rays IDA Pro 9.3 (x64) | PE layout and segment analysis, cross-references, string references, and control-flow recovery. |
| Managed-code analysis | dnlib 4.5, ILSpy / dnSpy, .NET Framework 4.8 | CIL inspection, metadata review, method-body analysis, and comparison of IL with decompiled C#. |
| Runtime observation | Process Explorer and Handle (Sysinternals) | Process relationships, open handles, and launcher/game behavior. |
| Windows forensics | PowerShell Core, Registry Editor, SQLite tools | Registry state, framework-managed storage, databases, and application logs. |
| Cryptographic validation | Python 3.10+ and PyCryptodome | Independent test vectors for observed .NET derivation and encryption behavior. |

## Reverse-Engineering Method

### 1. Recovering structure from an obfuscated .NET executable

The target was a 64-bit WPF application built on .NET Framework 4.8 and protected with PreEmptive Dotfuscator. The analysis encountered flattened control flow, arithmetic dispatch logic, proxy-call redirection, and encrypted strings.

IDA Pro 9.3 was used to inspect PE segments, code and string regions, cross-references, and native control flow. Managed metadata and CIL inspection were then used to resolve the application-level meaning of the recovered routines. Runtime observations provided a second view of process behavior and helped check the static model.

This combined workflow connected obfuscated code to major application responsibilities: telemetry collection, encryption and decryption, local enforcement, process management, periodic account checks, registration, and authentication.

### 2. Reconstructing the hardware-identity pipeline

The launcher queried SMBIOS-related information through Windows Management Instrumentation. The reconstructed data flow covered firmware structures for BIOS information, processor characteristics, and memory-device details. Network-interface information was also collected.

The inputs were normalized into a multi-field hardware manifest, then transformed and serialized for the service. Tracing the pipeline required moving between WMI usage, managed method bodies, IL control flow, and the network-facing code. This analysis exposed the assumptions the client made about telemetry integrity and the trust placed in client-reported identifiers.

### 3. Validating cryptographic behavior

The telemetry path used AES-256-CBC with PKCS#7 padding and .NET `PasswordDeriveBytes` behavior. The analysis examined the derivation flow, byte handling, and encoded output. Python and PyCryptodome were used to construct independent test vectors and compare the observed .NET behavior with a second implementation.

This work demonstrated protocol-level analysis across a managed runtime and an independent validation environment. Reusable key material, fixed derivation values, IVs, and sample hardware identifiers are not included.

### 4. Mapping asynchronous state machines

The later launcher used compiler-generated C# state machines implementing `IAsyncStateMachine`. The login and registration paths were reconstructed by following `MoveNext()` dispatch, state transitions, awaiter resumption, socket activity, response parsing, and the handoff to game startup.

The registration path included credential submission, email verification, and a final session-establishment stage. Mapping these transitions made it possible to distinguish client-side UI state from server-side acceptance and token issuance. The error path also explained how an unexpected server response could leave the interface waiting for a session result that would never arrive.

## Architectural Evolution

```text
Version 1.0 — Client-side evaluation
  Synchronous requests returned status values. The launcher interpreted
  these locally and controlled related UI and process behavior.

Version 2.1 — Two enforcement layers
  Login-time fingerprint association was combined with periodic in-game
  verification. The client generated telemetry and processed enforcement
  responses while the game was running.

Version 5.0 — Server-authoritative sessions
  Login and registration moved into asynchronous flows. The backend evaluated
  account state and additional risk signals, then controlled issuance of the
  signed session credential required by the game.
```

### Version 1.0: local status and enforcement logic

Static analysis showed synchronous network calls feeding local status checks. The launcher also contained local enforcement behavior associated with process management and file handling. This design made important reactions dependent on code running inside the client process.

The reverse-engineering work traced the path from server response to local UI and runtime behavior. Controlled analysis confirmed that changing local behavior could affect the client’s response, while leaving remote account state under server control.

### Version 2.1: login checks and periodic verification

Version 2.1 introduced two distinct enforcement paths:

1. **Login and fingerprint association:** hardware- and network-derived data was sent during authentication and compared with server-side account state.
2. **Periodic runtime verification:** a background asynchronous task checked the active game session and routed the response to launcher enforcement behavior.

IDA Pro and IL analysis connected the recurring task, response parser, UI handling, and launcher/game process relationship. Runtime observation helped validate how the periodic flow differed from the initial login path.

The investigation demonstrated the architectural limitation of client-generated telemetry and client-side enforcement. Local behavior was modifiable; this did not modify the service database or create an independent authorization decision. The specific patch recipe and spoofing values are not part of this publication.

### Version 5.0: authentication and the server boundary

The later client used separate asynchronous flows for authentication and multi-stage registration. Attempts to continue beyond the registration flow depended on backend account state, network-risk checks, and final issuance of a cryptographically protected session credential.

When the backend declined to issue that credential, changes to local control flow could not manufacture a valid server signature or force a remote account transaction to succeed. This marked the practical boundary between client reverse engineering and server authority.

## Windows Forensics and Persistence

The application retained state across reinstalls in more than one place. The forensic review correlated:

- SQLite databases used by launcher components.
- WPF IsolatedStorage and framework-managed settings.
- Windows Registry configuration.
- Application and game logs.
- Third-party client and mod caches.

This work demonstrated a multi-layered forensic approach to desktop state. Exact user-specific identifiers, cache locations, and procedures for removing traces are omitted.

## Findings Matrix

| Feature | Version 1.0 | Version 2.1 | Version 5.0 |
| :--- | :--- | :--- | :--- |
| Authentication flow | Synchronous client requests | Socket flow with additional checks | Compiler-generated asynchronous state machines |
| Hardware identity | Firmware-derived data | Firmware and network-interface data | Encrypted telemetry plus server-side request context |
| Enforcement | Local interpretation of status | Login association and recurring verification | Backend account validation and session issuance |
| Client role | Participates in access presentation | Reports identity data and handles runtime responses | Communicates with the service; does not own authorization state |
| Decisive boundary | Local code path | Server state remains independent of local checks | Valid server-issued session credential |

## Engineering Takeaways

### Reverse engineering requires multiple representations

IDA Pro’s PE, cross-reference, and control-flow views complemented dnlib’s metadata and CIL analysis. Decompiler output was treated as one representation to compare against the underlying IL, while runtime tools helped check process and handle behavior.

### Async internals matter when tracing network behavior

Following `IAsyncStateMachine`, `MoveNext()`, state transitions, and awaiter resumption was necessary to understand the launcher’s login, registration, and periodic verification flows. A normal call-stack view alone would not have shown the full transaction lifecycle.

### Client telemetry is still client input

Hardware-derived values collected by a desktop application are generated in an environment the user controls. They should be treated as claims, not proof of identity or integrity.

### Authorization belongs at the server boundary

Obfuscation, client-integrity checks, and local status handling can increase analysis cost, but they do not replace server-side authorization. The later architecture relied on authoritative account state and short-lived signed session credentials, with rate limiting and risk review around registration.

## Technologies and Concepts

- **Languages and formats:** C#, CIL / MSIL, PowerShell
- **Tools:** IDA Pro, dnlib, ILSpy / dnSpy, Process Explorer, Handle, Registry Editor, SQLite tools, Python, PyCryptodome
- **Frameworks:** .NET Framework, WPF, `IAsyncStateMachine`
- **Areas:** Reverse engineering, Windows forensics, cryptographic validation, application security, distributed systems, threat modeling

