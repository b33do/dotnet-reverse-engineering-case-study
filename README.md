# Case Study: .NET Intermediate Language (IL) Analysis, Client Telemetry, and Server-Authoritative Boundaries

A deep-dive technical post-mortem and case study investigating the evolution of client-side validation, compiler-generated asynchronous state machines, and the enforcement of the client-server trust boundary in desktop applications.

---

## 📌 Executive Summary

This project documents an in-depth security and systems research investigation of a production .NET 64-bit WPF game launcher application. Over several architectural iterations (**v1.0**, **v2.1**, and **v5.0**), the application evolved its telemetry collection, local storage mechanisms, and server-side validation models. 

This repository analyzes:
1. How compiler-generated C# `async`/`await` state machines operate at the Microsoft Intermediate Language (MSIL) opcode level.
2. How desktop applications collect hardware telemetry via Windows Management Instrumentation (WMI) and native network interfaces.
3. Multi-tier persistence mechanisms across SQLite, the Windows Registry, and WPF `IsolatedStorage`.
4. **The Zero-Trust Architectural Boundary**: Why even complete client-side telemetry control cannot overcome server-authoritative cryptographic validation.

---

## 🏗️ Architecture & Evolution Timeline

```
+---------------------------------------------------------------------------------------------------+
|                                       EVOLUTION TIMELINE                                          |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|  [v1.0: Naive Client Architecture]                                                                |
|  Client evaluates strings locally (e.g. `if (status == "BANNED")`)                                |
|  -> Weakness: Trivial conditional branch patching (brtrue -> brfalse).                            |
|                                                                                                   |
|  [v2.1: Telemetry-Bound Architecture]                                                             |
|  Server records client-reported WMI / MAC telemetry into an account database.                    |
|  -> Weakness: Client reports arbitrary telemetry; spoofing WMI queries succeeds.                  |
|                                                                                                   |
|  [v5.0: Distributed State Machines & Server-Authoritative Security]                               |
|  - Logic compiled into complex async state machines (<VHOD>d__358, <REG>d__357).                  |
|  - Cryptographic session issuance required to authenticate game client.                          |
|  - Multi-tier server validation: IP/ASN reputation, rate limits, and server-side token generation.  |
|  -> Outcome: Client telemetry neutralized, but remote server authority prevents token issuance.   |
|                                                                                                   |
+---------------------------------------------------------------------------------------------------+
```

---

## 🔍 Phase 1: v1.0 — Baseline Client Evaluation

In early versions of the launcher, authentication and access control were evaluated via straightforward procedural logic.

- **Inspection**: Decompilation revealed synchronous HTTP/Socket calls that returned plain-text status codes from the server.
- **Client Mechanism**: The launcher executed checks such as:
  ```csharp
  // Conceptual representation of v1.0 logic
  string response = ServerApi.CheckAccount(username);
  if (response == "BANNED" || response == "SVRBANT") {
      MessageBox.Show("Your account is suspended.");
      Application.Current.Shutdown();
  }
  ```
- **Architectural Flaw**: Client-side conditional logic can be bypassed by inverting a single IL branch instruction (`brtrue` $\to$ `brfalse`, or replacing with `nop` / `br.s`). The client mistakenly believed it was the authority on whether to display the game interface.

---

## 🛠️ Phase 2: v2.1 — Hardware Telemetry & Identity Fingerprinting

To prevent simple account evasion, the launcher introduced client hardware fingerprinting.

### Telemetry Pipeline
The client queried several Windows subsystems to create a composite hardware identifier (HWID):
- **Network Interface**: `NetworkInterface.GetAllNetworkInterfaces()` queried for physical MAC addresses.
- **WMI Queries**:
  - `SELECT SerialNumber FROM Win32_BaseBoard`
  - `SELECT ProcessorId FROM Win32_Processor`
  - `SELECT UUID FROM Win32_ComputerSystemProduct`

### Analysis & Interception
In v2.1, the server's backend blindly trusted the client to report its own hardware IDs over the wire:
1. By disassembling the assembly using `dnlib`, the methods responsible for executing WMI queries were located.
2. The method bodies were rewritten at the IL level to inject synthetic GUIDs and randomized MAC byte arrays before the payload was serialized.
3. Because the server only matched the incoming string against its existing blacklist database, spoofed client telemetry successfully satisfied the check.

---

## ⚡ Phase 3: v5.0 — Asynchronous State Machines & Server Authority

The target application was updated to a modernized .NET Framework WPF architecture with asynchronous task handling, multi-factor registration, and enhanced backend filtering.

### 1. Dissecting the Roslyn Async State Machines
Modern C# compilers do not emit standard procedural loops for `async`/`await` methods; they generate internal structs implementing `IAsyncStateMachine`.

Two critical state machines were mapped:
- **`<VHOD>d__358`** (Login Flow): Handled socket handshakes, account authentication, credential verification, and session token receipt.
- **`<REG>d__357`** (Registration Flow): Managed multi-step account registration:
  1. *Step 1*: Submission of email, username, and password.
  2. *Step 2*: Remote dispatch of email verification PIN.
  3. *Step 3*: Final PIN submission and cryptographic session handshake.

```
       [Registration State Machine: <REG>d__357]
                          |
                          v
         +----------------------------------+
         | State 0: Send Initial Form       |
         +----------------------------------+
                          |
        [Server accepts HWID / Sends PIN]
                          |
                          v
         +----------------------------------+
         | State 1: Await Email PIN Input   |
         +----------------------------------+
                          |
             [User Submits PIN Code]
                          |
                          v
         +----------------------------------+
         | State 2: Final Verification      |
         +----------------------------------+
             /                          \
   [Valid Session]               [Server Error: IPRESET]
          |                                 |
          v                                 v
   (Spawn Game Client)         (Unhandled Async Freeze)
```

### 2. Binary Bytecode Transformation (P1 - P10)
To audit the local client controls, a custom patching engine (`DayZavrPatcher`) was engineered using `dnlib` to execute 12 bytecode modifications:
- **P1 - P4 (Telemetry Spoofing)**: Intercepted WMI routines and hardware query handlers, replacing hardware strings with randomized identifiers.
- **P5 - P8 (Ban Check Neutralization)**: Located conditional branch opcodes evaluating `SVRBANT`, `SVRBANP`, and `BANNED`, converting them to unconditional branch instructions (`br.s`).
- **P9 - P10 (UI Exception Delegate Suppression)**: Patched the error delegate `b__13` to prevent UI crash dialogs when receiving unknown server error packets.

---

## 🔬 Local State Forensics & Forensic Scrubbing

A major challenge during analysis was that the application remembered user state (selected server, language, previous account traces) even after reinstalling the binary. Forensic investigation revealed a multi-tiered persistence footprint across Windows:

| Persistence Layer | Location / Schema | Purpose |
| :--- | :--- | :--- |
| **Local Databases** | `%LOCALAPPDATA%\DayZavr\*.db` (`launcher.db`, `Register.db`, `Firefox.db`) | SQLite databases storing cached authentication tokens, news feeds, and registration hashes. |
| **WPF IsolatedStorage** | `%LOCALAPPDATA%\IsolatedStorage\<hash>\<hash>\Files\*.settings` | Encrypted/hashed framework-managed storage persisting UI locale and server preferences across restarts. |
| **Windows Registry** | `HKCU\Software\DayZavr`, `HKLM\Software\DayZavr` | Registry keys persisting install directories, launch flags, and unique installation GUIDs. |
| **Temporary Dumps** | `C:\Temp\DayZ*` | Temporary executable unpacks and memory dumps. |

A comprehensive PowerShell forensics script was authored to recursively audit and scrub these artifacts, ensuring testing occurred from a true clean-slate environment.

---

## 🧱 The Architectural Wall: Client Authority vs. Server Authority

Despite completely controlling the client-side environment (neutralized ban dialogs, spoofed hardware telemetry, wiped persistence, and rotated IP addresses), the registration sequence hit an infinite loading state during Step 3 of `<REG>d__357`.

### Root Cause Analysis:
1. **The Telemetry Illusion**: The fact that the server sent an email verification PIN proved that client hardware spoofing was successful (the server did not recognize the machine).
2. **Server-Authoritative Heuristics**: Upon final submission of the PIN, the server evaluated parameters completely outside the client's control:
   - **Autonomous System Number (ASN) & IP Reputation**: Detection of datacenter, proxy, VPN, or flagged ISP blocks.
   - **Rate-Limiting & Registration Windows**: Backend database limits preventing rapid account provisioning.
   - **Cryptographic Token Issuance**: The DayZ game server requires a cryptographically signed ticket from the master server during the connection handshake. Because the server refused to issue this session token, the client had no ticket to pass to the game executable.

> **Key Takeaway**: A client can modify any instruction running in local memory, but it cannot forge a valid digital signature or force a remote database to issue an authoritative session token.

---

## 💡 Key Engineering Takeaways for Systems & Security Interviews

### 1. Threat Modeling & Zero-Trust Principles
- **Principle**: *Never trust the client.* Any security enforcement performed in client code (`if (!isBanned)`) is an illusion.
- **Defensive Design**: Systems must assume the client binary is running in a compromised environment under an interactive debugger. Sensitive actions must be guarded by short-lived, cryptographically signed tokens (e.g. JWTs / HMAC session tickets) validated by the server on every request.

### 2. Reversing Compiler-Generated Asynchronous Code
- C# `async`/`await` methods compile into struct-based state machines. Tracing control flow requires locating the `MoveNext()` method, identifying the `state` field (often `-1` or `0`), and tracking how `TaskAwaiter` callbacks update the execution state.

### 3. Desktop Application Forensic Footprints
- Modern desktop frameworks (WPF, WinUI, Electron) store state in non-obvious locations. Fully sanitizing a desktop application requires inspecting SQLite files, Windows Registry hives, Windows Credential Manager, and framework stores like .NET `IsolatedStorage`.

### 4. Defense in Depth
- Anti-tampering, obfuscation, and client-side integrity checks are only speed bumps. Robust security relies on server-authoritative state, strict API rate-limiting, network telemetry analysis (ASN/IP reputation), and backend audit logging.

---

## 📜 Technologies & Concepts Applied

- **Languages**: C#, CIL / MSIL (Microsoft Intermediate Language), PowerShell
- **Tooling & Libraries**: `dnlib`, ILSpy / dnSpy, Process Explorer, Regedit, SQLite3
- **Frameworks**: .NET Framework, WPF (MahApps.Metro), Asynchronous Programming Model (`IAsyncStateMachine`)
- **Core Disciplines**: Reverse Engineering, Windows Forensics, Threat Modeling, Distributed Systems Security, Zero-Trust Architecture

