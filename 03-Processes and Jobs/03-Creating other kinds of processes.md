Most Windows applications create processes through the familiar `CreateProcess*` APIs. However, Windows supports several other types of processes that cannot be created through the normal Windows application-level process creation mechanism.

These include:

- Native processes
    
- Minimal processes
    
- Pico processes
    

Such processes require lower-level process creation mechanisms because they serve specialized purposes within the Windows operating system.

## 3.1 Native Processes

Native processes are processes associated with the Windows Native subsystem rather than the standard Win32 application environment.

One important example is **Session Manager (**`**smss.exe**`**)**.

Unlike ordinary applications, Session Manager is initially launched directly by the kernel. Since the regular Windows API environment is not yet available, it cannot rely on `CreateProcess` for its creation.

Instead, the kernel invokes the native system call `NtCreateUserProcess` directly.

Session Manager also creates other important system processes, including:

- **Autochk:** A utility responsible for checking file systems during startup when required.
    
- **Csrss.exe:** The Client/Server Runtime Subsystem process, which provides important Windows subsystem functionality.
    

At this stage, the normal Windows application environment is still unavailable, so Session Manager must use the native process creation mechanism rather than the standard Win32 APIs.

There is another limitation: ordinary Windows applications cannot create native subsystem executables through the regular `CreateProcess` execution path.

This is because `CreateProcessInternal` checks the executable's subsystem type and rejects images marked for the Native subsystem.

### RtlCreateUserProcess

To simplify native process creation, `ntdll.dll` provides a helper function called `RtlCreateUserProcess`.

It acts as a higher-level wrapper around `NtCreateUserProcess`, making it easier for native components to request the creation of a user-mode process without directly handling all the lower-level details.

## 3.2 Minimal Processes

Not every Windows process needs the complete collection of structures and facilities associated with an ordinary user-mode process.

Windows also supports **minimal processes**, which are specialized process objects created with a reduced set of normal process initialization features.

Examples include:

- The System process.
    
- Memory Compression processes.
    

These processes are not created through the regular `NtCreateUserProcess` system call.

Instead, Windows provides another system call:

`NtCreateProcessEx`

This system call supports specialized process creation scenarios, including minimal process creation.

Certain capabilities exposed through this mechanism are restricted to kernel-mode callers.

This is important because minimal processes serve operating-system-level purposes rather than behaving like ordinary applications.

## 3.3 Pico Processes

Pico processes are another specialized process type used by Windows to support environments that require a different operating-system interface from the standard Windows user-mode environment.

A notable example is the **Windows Subsystem for Linux (WSL)**.

Pico processes are managed through components called **Pico providers**.

A Pico provider is responsible for supplying the operating-system functionality required by the processes it manages.

For creating Pico processes, Windows uses an internal helper function:

`PspCreatePicoProcess`

This function handles two important tasks:

1. Creates the underlying minimal process.
    
2. Initializes the process with the necessary Pico provider context.
    

Unlike the previously discussed native APIs, `PspCreatePicoProcess` is not an exported function. Pico providers access this functionality through a specialized interface provided by Windows.

## 3.4 NtCreateUserProcess vs NtCreateProcessEx

Windows provides two different native system calls for process creation:

|System call|Purpose|
|---|---|
|`NtCreateUserProcess`|Creates regular user-mode processes, including native subsystem processes.|
|`NtCreateProcessEx`|Supports alternative process creation scenarios, including specialized and minimal processes.|

Although these system calls have different responsibilities and capabilities, their internal execution eventually reaches common process management routines.

Two important internal functions are:

- **PspAllocateProcess:** Responsible for allocating and preparing the internal process object.
    
- **PspInsertProcess:** Handles inserting the newly created process into the system's process management structures.
    

These routines form part of the common internal process creation infrastructure.

## 3.5 Common Process Creation Infrastructure

Different Windows components can initiate process creation through different mechanisms.

For example:

- A regular application can use `CreateProcess`.
    
- Native components can use `NtCreateUserProcess` or `RtlCreateUserProcess`.
    
- Kernel components can use specialized process creation mechanisms.
    
- Pico providers can use their dedicated process creation interface.
    

Despite these differences, the underlying process creation work eventually converges on shared internal routines.

The following diagram illustrates the general relationship:

```
                Process Creation
                       |
        +--------------+--------------------+
        |              |                    |
   Win32 APIs     Native Components    Kernel Components
        |              |                    |
   CreateProcess  NtCreateUserProcess  NtCreateProcessEx
        |              |                    |
        |         RtlCreateUserProcess      |
        |                                   |
        +--------------+--------------------+
                       |
              Common Internal Routines
                       |
                PspAllocateProcess
                       |
                 PspInsertProcess
                       |
                 Process Created
```

Pico process creation follows a specialized path through `PspCreatePicoProcess`, which also initializes the associated provider context.

The important point is that Windows has multiple entry points for creating different kinds of processes, but much of the underlying process management work is handled by shared Executive routines.

---

## Key Takeaways

- Not every Windows process is created through the standard `CreateProcess` API.
    
- Native processes can be created through `NtCreateUserProcess`, even before the normal Win32 environment is available.
    
- `RtlCreateUserProcess` provides a helper interface around `NtCreateUserProcess`.
    
- Minimal processes are specialized process objects that do not require the complete initialization of ordinary user-mode processes.
    
- `NtCreateProcessEx` supports specialized process creation scenarios, including minimal processes.
    
- Pico processes rely on Pico providers, with `PspCreatePicoProcess` handling their creation and provider-context initialization.
    
- Different process creation mechanisms eventually share internal routines such as `PspAllocateProcess` and `PspInsertProcess`.