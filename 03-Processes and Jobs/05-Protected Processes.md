Windows normally allows processes to request access to other processes, provided the requested access rights are granted by the security checks performed by the operating system.

However, this creates a security problem when a process handles sensitive information that must remain inaccessible even to other highly privileged user-mode processes.

To address this problem, Windows introduced **Protected Processes (PP)** in Windows Vista and Windows Server 2008.

Protected Processes introduce additional access restrictions that prevent ordinary user-mode processes, including processes running with administrative privileges, from freely accessing or manipulating protected processes.

Their original purpose was to support Digital Rights Management (DRM) and protect high-value media content from unauthorized access or modification.

## 5.1 Why Protected Processes Were Introduced

Under the normal Windows security model, a process can request access to another process through APIs such as `OpenProcess`.

The requested access is checked against the target process's security descriptor and the caller's security privileges.

One particularly important privilege is `SeDebugPrivilege`.

When enabled, this privilege allows a process to bypass the normal discretionary access checks on another process's security descriptor in many situations.

This is useful for administrators and debugging tools because it allows them to inspect and manage processes even when the target process's security descriptor would otherwise deny access.

For example, tools such as:

- Process Explorer
    
- Task Manager
    
- WinDbg
    

may request process access rights to inspect process information, examine memory, or perform debugging operations.

However, unrestricted access creates a problem for applications that handle protected content.

Consider a media application playing high-definition content protected by DRM. If another process can freely read its memory, inject code, or modify its execution, the protected content could potentially be extracted or manipulated.

This is where Protected Processes become important.

They introduce restrictions that cannot simply be bypassed by enabling `SeDebugPrivilege` or running the requesting application as an administrator.

## 5.2 How a Process Becomes Protected

Windows does not allow an ordinary application to arbitrarily mark itself as a Protected Process.

Although applications can request protected process creation through supported mechanisms, the executable must satisfy the operating system's trust requirements.

For the original Protected Process model, this involves a special Windows Media certificate and the associated signing and trust requirements.

This ensures that only appropriately trusted software can run with the original protected process designation.

### Protected Media Path (PMP)

The **Protected Media Path (PMP)** is part of the Windows media protection architecture.

It provides an isolated environment for handling high-value media content.

The idea is to separate sensitive media processing from the ordinary application environment.

For example, a media application may handle playback controls and user interaction, while sensitive content processing occurs inside a protected environment.

This separation helps prevent other user-mode processes from directly accessing or modifying the protected media-processing components.

Applications can use the Media Foundation APIs to work with protected media functionality.

The protected environment also relies on trust checks for its components. If an untrusted component compromises the protected environment, protected content processing can be refused.

## 5.3 Examples of Protected Processes

Several Windows components have used process protection to secure sensitive operations.

Examples discussed in the Windows Internals protected-process model include:

### Audiodg.exe

`Audiodg.exe` is the Windows Audio Device Graph Isolation process.

It hosts audio processing components and can participate in protected audio processing when protected content is involved.

This is important because audio content may need protection while being decoded and processed.

### Mfpmp.exe

`Mfpmp.exe` is associated with the Media Foundation Protected Pipeline.

It provides a protected environment for certain media processing operations.

It is not necessarily running on every Windows system or during ordinary media playback.

### WerFaultSecure.exe

`WerFaultSecure.exe` is a Windows Error Reporting component associated with protected-process crash reporting.

It needs specially controlled access to information from protected processes so that crash information can be collected without granting unrestricted access to ordinary applications.

### System Process

The System process is another important example discussed in the protected-process architecture.

It is associated with critical kernel-related operations and can contain sensitive information and handles.

Protecting this process helps defend important operating-system resources against unauthorized user-mode access.

However, the protection status and implementation details of these system processes can differ between Windows versions.

**Important:** These examples should not be interpreted as a guarantee that every instance of these executables is always a Protected Process on every Windows version.

## 5.4 How Windows Enforces Process Protection

Protected Processes are enforced through kernel-level mechanisms.

Two important aspects of this implementation are:

1. Special handling during process creation.
    
2. Additional restrictions during process and thread access checks.
    

### Protected Process Creation

The creation of a protected process requires special handling by the Windows kernel.

A major reason for this is to prevent unauthorized modification or injection during the creation and initialization stages.

The kernel participates in the process creation operation and enforces the requirements associated with the protected process designation.

This helps establish the protected state before an ordinary user-mode process could interfere with the newly created process.

### EPROCESS Protection Information

Windows maintains protection-related information in the internal `EPROCESS` structure.

This information influences how the kernel handles attempts to obtain access to protected processes.

When another process requests a handle to a protected process, the kernel applies additional restrictions to the requested access rights.

As a result, even if the requesting process has administrative privileges or `SeDebugPrivilege`, certain operations remain unavailable.

This is an important distinction:

**Having administrative privileges does not automatically grant unrestricted access to every process in Windows.**

## 5.5 Access Restrictions on Protected Processes

Windows prevents other processes from obtaining several sensitive access rights to a Protected Process.

Some important examples include:

|Access right|What it allows|
|---|---|
|`PROCESS_CREATE_THREAD`|Creating a thread inside the target process.|
|`PROCESS_VM_OPERATION`|Performing operations on the target process's virtual memory.|
|`PROCESS_VM_READ`|Reading memory from the target process.|
|`PROCESS_VM_WRITE`|Modifying memory inside the target process.|
|`PROCESS_DUP_HANDLE`|Duplicating handles from or into the target process.|
|`PROCESS_CREATE_PROCESS`|Using the target process to create a child process.|
|`PROCESS_SET_INFORMATION`|Modifying certain process properties.|
|`PROCESS_QUERY_INFORMATION`|Requesting more extensive process information.|
|`PROCESS_ALL_ACCESS`|Requesting the complete set of process access rights.|

Windows also restricts several standard security rights, including `WRITE_DAC`, `WRITE_OWNER`, and `READ_CONTROL`.

A less restrictive right, `PROCESS_QUERY_LIMITED_INFORMATION`, exists to allow access to a limited subset of process information.

The important consequence is that many operations commonly used for process inspection and manipulation are unavailable through ordinary user-mode process handles.

Thread access is restricted as well. This prevents an ordinary process from simply bypassing process-level protection by opening one of the protected process's threads directly.

## 5.6 Kernel Debugging and Protected Processes

Protected Processes introduce additional considerations when debugging Windows.

A user-mode debugger or monitoring tool relies on Windows APIs to obtain handles to processes and threads.

If the target process is protected, the kernel may deny access to the rights required by these tools.

For example, Process Explorer may be able to identify a protected process but fail to retrieve certain internal information or perform operations that require restricted access rights.

Kernel debugging works differently.

A kernel debugger such as WinDbg operates through kernel-debugging infrastructure rather than relying exclusively on ordinary user-mode process handles.

This allows it to inspect kernel structures and information that a normal user-mode debugging tool cannot access.

### Enabling Kernel Debugging

Windows supports enabling kernel debugging through the Boot Configuration Data (BCD) configuration.

The following command enables kernel debugging for the current boot entry:

```
bcdedit /debug on
```

A system restart is required for the configuration change to take effect.

When kernel debugging is enabled, the Protected Media Path security policy prevents protected high-definition content playback.

This is intended to prevent debugger access from undermining the media protection environment.

**Note:** Enabling kernel debugging changes the security configuration of the machine. It should be done in a dedicated research or debugging environment rather than on a system used for protected media playback.

## 5.7 Protected Processes and Kernel-Mode Security

Protected Processes primarily restrict access from other user-mode processes.

However, this does not mean that they are completely immune to kernel-mode attacks.

A kernel-mode driver operates with much greater privileges than an ordinary user-mode application and may be capable of accessing or modifying protected process memory and kernel structures.

For example, if a malicious driver could modify the protection information maintained in `EPROCESS`, it could potentially undermine the restrictions enforced by the process manager.

Windows uses several security mechanisms to make such attacks more difficult.

### Driver Signing and Code Integrity

On 64-bit Windows, kernel-mode code-signing requirements restrict which drivers can be loaded.

Code Integrity verifies kernel-mode code according to the applicable signing and integrity policies.

This makes it more difficult for an attacker to introduce an unauthorized kernel driver that could interfere with protected processes.

However, driver signing should not be treated as an absolute guarantee that malicious kernel code can never execute.

### PatchGuard

Kernel Patch Protection, commonly known as PatchGuard, helps protect critical kernel code and data structures against unauthorized modification.

It is designed to detect certain classes of kernel modification, particularly those that interfere with protected operating-system structures and code.

However, PatchGuard should not be understood as a universal mechanism that detects every possible modification to every field inside `EPROCESS`.

### Protected Media Environment Enforcement

The Protected Media Path also has its own trust and integrity requirements.

Components involved in protected media processing must satisfy the applicable trust policies.

If the protected environment detects an untrusted or compromised component, protected content processing may be refused or terminated.

The combination of process access restrictions, kernel code integrity, and protected media trust policies makes unauthorized interference more difficult.

**Key distinction:** Protected Processes are primarily a user-mode isolation mechanism. They do not establish an absolute security boundary against a compromised Windows kernel.