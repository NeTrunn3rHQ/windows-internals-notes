The original Protected Process (PP) model was primarily designed to protect DRM-related media content. However, Windows eventually needed similar protection mechanisms for other security-critical components, including authentication services, anti-malware software, and core operating-system processes.

To address these requirements, Microsoft introduced **Protected Process Light (PPL)** in Windows 8.1 and Windows Server 2012 R2.

PPL extends the original protected process architecture by introducing different protection levels based on the trustworthiness of the executable and its signer.

This means that Windows does not simply distinguish between protected and unprotected processes. It can also determine how much protection one protected process has relative to another.

## 6.1 Understanding Protected Process Light

PPL provides many of the same security benefits as the original Protected Process model.

For example, ordinary user-mode applications, including those running with administrative privileges and `SeDebugPrivilege`, cannot freely:

- Read protected process memory.
    
- Write to protected process memory.
    
- Inject threads into protected processes.
    
- Obtain unrestricted access to their internal information.
    
- Manipulate their execution through ordinary process and thread handles.
    

However, PPL introduces an additional concept: **protection levels based on signer identity**.

Each protected process has a protection type and a signer value. These attributes determine how the process is treated when another process attempts to access it.

This allows Windows to establish a hierarchy of trust between protected processes.

For example, a Windows security component with a higher protection level can receive access that would be denied to an ordinary application or a lower-level protected process.

Therefore, PPL provides more flexibility than the original Protected Process model, where protection was primarily associated with DRM requirements.

## 6.2 Protection Types and Signers

Windows represents process protection information using an internal structure called `PS_PROTECTION`.

This information is stored in the `Protection` member of the `EPROCESS` structure.

The protection information contains three logical fields:

|Field|Size|Purpose|
|---|---|---|
|`Type`|3 bits|Identifies whether the process is unprotected, PPL, or a full Protected Process.|
|`Audit`|1 bit|Reserved for auditing-related purposes.|
|`Signer`|4 bits|Identifies the trust category associated with the process.|

The `Type` field distinguishes between three states:

|Value|Name|Meaning|
|---|---|---|
|0|`PsProtectedTypeNone`|Normal, unprotected process.|
|1|`PsProtectedTypeProtectedLight`|Protected Process Light (PPL).|
|2|`PsProtectedTypeProtected`|Full Protected Process (PP).|

The `Signer` field identifies the trust category associated with the process.

### Protected Process Signers

Windows defines several signer categories. Their general hierarchy, from lower to higher trust, is:

|Signer|Description|
|---|---|
|`PsProtectedSignerNone`|No protected signer.|
|`PsProtectedSignerAuthenticode`|Authenticode-signed code.|
|`PsProtectedSignerCodeGen`|Trusted code-generation components.|
|`PsProtectedSignerAntimalware`|Approved anti-malware components.|
|`PsProtectedSignerLsa`|Local Security Authority components.|
|`PsProtectedSignerWindows`|Windows-signed components.|
|`PsProtectedSignerWinTcb`|Windows Trusted Computing Base components.|
|`PsProtectedSignerWinSystem`|Highly trusted Windows system components.|
|`PsProtectedSignerApp`|Application-related signer category.|

The signer value is important because it influences which other protected processes can obtain access to the target process.

**Important:** The signer list represents protection categories, not a simple list of Windows executables. The actual protection level also depends on the process type.

### How Protection Levels Interact

Windows evaluates protection types and signer values when determining access between protected processes.

There are two important rules:

1. A full Protected Process has a higher protection type than a Protected Process Light process, regardless of their signer values.
    
2. Within the same protection type, a higher-trust signer generally has access privileges over a lower-trust signer, but the reverse is not true.
    

For example:

```
Full Protected Process (PP)
              |
              | Higher protection type
              v
Protected Process Light (PPL)
              |
              | Signer hierarchy
              v
          WinSystem
              |
            WinTcb
              |
           Windows
              |
             LSA
              |
         Antimalware
              |
           CodeGen
              |
         Authenticode
```

This diagram illustrates the general hierarchy, not an unconditional guarantee of unrestricted access. Specific access rights and additional security checks still apply.

## 6.3 How Windows Determines Whether a Process Can Become Protected

A process cannot simply claim a higher protection level and expect Windows to accept it.

If arbitrary applications could assign themselves a trusted protection level, malware could potentially use PPL to prevent security software from inspecting or terminating it.

Windows therefore relies on Code Integrity and certificate-based trust verification to determine which executables are eligible for protected execution.

### Enhanced Key Usage (EKU)

Digital certificates can contain extensions called **Enhanced Key Usage (EKU)**.

An EKU identifies the intended purpose for which a certificate can be used.

Windows uses specific EKU identifiers, together with certificate issuer information and additional trust requirements, to determine the signing category associated with protected components.

Two important EKU identifiers discussed in the protected-process model are:

```
1.3.6.1.4.1.311.10.3.22
1.3.6.1.4.1.311.10.3.20
```

These are associated with Windows protected-process signing requirements.

The certificate must also satisfy the applicable issuer and additional EKU requirements.

For example, a certificate issued under the Microsoft Windows issuer can qualify for the Windows protected signer category when the required Windows System Component Verification EKU is also present:

```
1.3.6.1.4.1.311.10.3.6
```

The important point is that simply possessing a digitally signed executable is not sufficient to obtain a protected signer designation.

Windows validates the signing information against its trusted signing policies before allowing the executable to run with the requested protection level.

## 6.4 Signature Levels and DLL Loading

Protecting the process itself is not enough.

Consider a situation where a legitimate Windows process is running as PPL with a high signer level.

If that process were allowed to load an arbitrary third-party DLL, the DLL would execute inside the protected process and could potentially gain access to information and functionality that ordinary applications cannot access.

This would undermine the entire protection mechanism.

To prevent this, Windows also applies signing restrictions to executable images and DLLs loaded into protected processes.

Two important fields in `EPROCESS` are involved:

- `SignatureLevel`
    
- `SectionSignatureLevel`
    

### SignatureLevel

`SignatureLevel` represents the signing level associated with the process's executable image.

It is used by the Code Integrity infrastructure when validating executable images.

### SectionSignatureLevel

`SectionSignatureLevel` represents the signing requirement applied to images mapped into the process, including DLLs.

Windows uses this information to determine whether a DLL satisfies the minimum signing requirements for the target process.

### DLL Loading and Code Integrity

When a process attempts to load a DLL, Windows performs the applicable image-integrity checks.

The DLL must satisfy the signing requirements associated with the target process.

For example, a process operating with a high Windows TCB signing level cannot simply load an arbitrary third-party DLL that has a lower signing level.

This prevents an attacker from replacing a legitimate library with an untrusted one and having it execute inside a highly protected process.

The important distinction is:

- `Protection` controls the process's protection type and signer identity for process-access decisions.
    
- `SignatureLevel` and `SectionSignatureLevel` participate in determining which executable images and DLLs are trusted to run inside the process.
    

These mechanisms work together to protect both the process from external interference and its internal execution environment from untrusted code.

## 6.5 Examples of Windows PPL Processes

Several important Windows components use PPL to protect sensitive operating-system functionality.

The following examples are based on the Windows 10 and Windows Server 2016-era implementation discussed in the Windows Internals material. Process protection configurations can change across Windows versions and system configurations.

### Core Windows Processes

The following processes were identified as using the WinTcb-Light protection level:

|Process|Purpose|
|---|---|
|`smss.exe`|Session Manager; responsible for important session and system initialization tasks.|
|`csrss.exe`|Client/Server Runtime Subsystem; provides essential Windows subsystem functionality.|
|`services.exe`|Service Control Manager; manages Windows services.|
|`wininit.exe`|Initializes important system processes and services during startup.|

These processes perform important operating-system tasks and require protection against interference from less-trusted processes.

### LSASS

`lsass.exe` (Local Security Authority Subsystem Service) is responsible for important Windows authentication and security operations.

It handles sensitive authentication-related information, making it an important target for attackers.

Windows supports running LSASS as a Protected Process Light process through a feature commonly known as **LSA protection**.

Depending on the Windows version and configuration, LSASS can run with the LSA protection level.

This makes it significantly harder for ordinary user-mode processes to access its memory or inject code into it.

### Other Protected Components

Additional examples include:

- `WerFaultSecure.exe` — Windows Error Reporting functionality associated with protected processes.
    
- `sppsvc.exe` — Software Protection Platform service.
    
- Certain `svchost.exe` instances hosting protected Windows services.
    

The exact protection level depends on the component, Windows version, and applicable system policy.

## 6.6 The Minimum TCB List

Some Windows processes are so important to system security that Windows does not allow them to run with an insufficient protection level.

This requirement is implemented through a mechanism commonly referred to as the **Minimum TCB list**.

TCB stands for Trusted Computing Base.

The purpose of this mechanism is to ensure that certain critical system executables receive the minimum protection and signing levels required by Windows, regardless of what protection level a process-creation request attempts to specify.

For example, an ordinary administrator should not be able to launch a replacement instance of a critical system process while deliberately requesting a lower protection level.

Windows checks the executable and its associated signing information and enforces the required minimum protection.

The Windows 10 and Windows Server 2016-era list discussed in the Windows Internals material includes:

- `smss.exe`
    
- `csrss.exe`
    
- `wininit.exe`
    
- `services.exe`
    
- `WerFaultSecure.exe`
    
- `sppsvc.exe`
    
- `genvalobj.exe`
    
- `lsass.exe`
    
- `userinit.exe`
    
- `winlogon.exe`
    
- `autochk.exe`
    

The requirement is also associated with the expected system location of these binaries.

### Why Is This Important?

Consider `csrss.exe`.

It has access to certain internal Windows subsystem functionality, including private interfaces implemented by the Window Manager.

If an administrator could replace or launch an insufficiently protected version of this process, an attacker might gain access to sensitive functionality that would otherwise be restricted.

Similarly:

- `smss.exe` participates in session initialization and management.
    
- `wininit.exe` performs important system initialization tasks.
    
- `services.exe` manages the Windows service infrastructure.
    
- `lsass.exe` handles sensitive authentication and security operations.
    

Allowing arbitrary processes to impersonate these trusted components would weaken the Windows security model.

The Minimum TCB mechanism helps ensure that these components receive the protection levels expected by the operating system.

**Important:** The list above is a historical Windows-version-specific reference, not a guaranteed, unchanging list for every modern Windows release.