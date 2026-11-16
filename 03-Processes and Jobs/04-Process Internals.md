# 04 — Windows Process Data Structures

A Windows process is represented internally by several interconnected data structures. The most important one is the **Executive Process (EPROCESS)** structure, which acts as the central representation of a process inside the Windows kernel.

However, EPROCESS does not contain every piece of information associated with a process. Instead, it maintains its own collection of attributes and references other structures responsible for different aspects of process management.

Understanding these structures is important when investigating how Windows manages processes, threads, memory, security, and graphical applications.

## 4.1 EPROCESS — Executive Process Structure

Every normal Windows process is represented by an `EPROCESS` structure.

It contains information about the process itself and references several other structures associated with it.

One important relationship is between `EPROCESS` and `ETHREAD`:

- **EPROCESS:** Represents the process and stores process-related information.
    
- **ETHREAD:** Represents an individual thread belonging to that process.
    

A process can contain multiple threads, with each thread having its own `ETHREAD` structure.

The `EPROCESS` structure also references other important structures, including the Process Environment Block (PEB).

### Kernel and User Address Space

Most process-related executive structures, including `EPROCESS` and `ETHREAD`, reside in kernel system address space.

However, the **Process Environment Block (PEB)** is different.

The PEB resides in the user-mode virtual address space of its associated process because it contains information required by user-mode components, particularly the Windows loader and runtime libraries.

Another important detail is that not every kernel-related structure is accessible in exactly the same way across processes.

For example, certain memory-management structures, including working-set information, are associated with a specific process address space. Some of this information is maintained in process-specific system-space mappings and must be accessed in the appropriate process context.

The important distinction is that:

- `EPROCESS` and `ETHREAD` are kernel-managed structures.
    
- `PEB` is a user-mode structure associated with a particular process.
    
- Some memory-management information is specific to an individual process context.
    

## 4.2 KPROCESS — Kernel Process Structure

The `EPROCESS` structure contains a member called `Pcb`, whose type is `KPROCESS` (Kernel Process).

In the Windows implementation discussed here, `KPROCESS` is located at the beginning of `EPROCESS`.

While `EPROCESS` contains information required by the Windows Executive, `KPROCESS` holds lower-level information used by core kernel mechanisms.

For example, it contains information relevant to:

- Thread scheduling and dispatching.
    
- Process execution state.
    
- Processor-related management.
    
- Kernel-level timing and accounting.
    

This separation provides an abstraction between the Executive and the lower-level kernel.

The Executive can maintain higher-level process information through `EPROCESS`, while core kernel components can work with the `KPROCESS` portion.

This design helps keep different parts of the operating system from depending unnecessarily on each other's internal implementation details.

**Important:** `KPROCESS` is embedded within `EPROCESS`; it is not a completely separate process object.

## 4.3 PEB — Process Environment Block

The PEB is a user-mode data structure associated with a process.

Unlike `EPROCESS`, which is maintained in kernel address space, the PEB resides in the process's user address space.

It contains information that user-mode components need during program execution.

Some examples of information maintained through the PEB include:

- Process startup information.
    
- Loaded module information.
    
- Process parameters.
    
- Windows loader-related state.
    
- Process heap information.
    

The PEB is particularly important when studying Windows internals because user-mode applications and libraries interact with information maintained there.

It is also frequently examined during reverse engineering, debugging, and malware analysis.

## 4.4 CSR_PROCESS — Windows Subsystem Process Structure

Windows maintains additional process-related information outside the main `EPROCESS` structure.

One example is `CSR_PROCESS`, which is maintained by the Client/Server Runtime Subsystem (`Csrss.exe`).

This structure contains information specific to the Windows subsystem.

It is associated with processes running Windows applications, rather than every process in the operating system.

For example, native system processes such as Session Manager (`smss.exe`) do not require a corresponding `CSR_PROCESS` structure.

An important detail is that Windows maintains separate instances of the Windows subsystem for different sessions.

Consequently, the corresponding `CSR_PROCESS` information is maintained by the `Csrss.exe` instance belonging to the process's session.

This allows the Windows subsystem to maintain process-related state independently for each session.

## 4.5 W32PROCESS — Win32 Kernel Process Structure

The kernel-mode portion of the Windows subsystem, implemented through `Win32k.sys`, maintains another per-process structure called `W32PROCESS`.

This structure stores information required by Windows kernel components responsible for graphical interfaces and window management.

A process becomes associated with this GUI-related state when it starts using the relevant Windows USER or GDI functionality implemented in kernel mode.

For example, applications using functions such as:

- `CreateWindow` / `CreateWindowEx`
    
- `GetMessage`
    

typically load `User32.dll` and begin interacting with the Windows GUI subsystem.

The important distinction is that simply being a normal Windows process does not necessarily mean that it needs all the GUI-related state maintained by `W32PROCESS`.

This structure is associated with processes that actually use the relevant graphical subsystem functionality.

**Note:** `W32PROCESS` is an internal Windows structure, and its complete type information is generally not available through public debugging symbols.

## 4.6 DXGPROCESS — DirectX Graphics Process Structure

Windows also maintains process-specific information for its graphics infrastructure.

The DirectX Graphics Kernel (`Dxgkrnl.sys`) manages important parts of the Windows graphics and GPU execution infrastructure.

A related internal structure is `DXGPROCESS`.

It maintains information associated with DirectX objects and GPU-related process state.

Examples of the resources and information associated with this infrastructure include:

- Graphics resources such as surfaces and shaders.
    
- GPU-related execution information.
    
- Graphics memory management.
    
- GPU scheduling and resource management policies.
    

This becomes particularly important for applications that use hardware-accelerated graphics or GPU-based computation.

The `DXGPROCESS` structure represents another example of how Windows components maintain their own process-specific information rather than placing every attribute directly inside `EPROCESS`.

## 4.7 Process Objects and the Object Manager

The Executive Object Manager manages Windows kernel objects, including process objects.

With the exception of the special Idle process, an `EPROCESS` is managed as a process object by the Object Manager.

However, there is an important distinction between an object's type and its name.

Process objects are generally unnamed objects in the Object Manager namespace. This means individual processes are not represented by named entries that can be browsed through tools such as Sysinternals WinObj.

However, the Object Manager maintains a type object named `Process`.

In WinObj, this type can be found under:

```
\ObjectTypes
    |
    +-- Process
```

This represents the **type of object**, not an individual running process.

### Process Handles

When a process obtains a handle to another process, that handle provides access to the process object, subject to the permissions granted to the handle.

Windows process-related APIs use these handles to perform operations such as:

- Querying process information.
    
- Reading or writing process memory, when permitted.
    
- Suspending or resuming threads.
    
- Terminating a process, when permitted.
    

The handle provides an interface to the underlying process object without requiring user-mode applications to access the internal `EPROCESS` structure directly.

This is also why a PID and a process handle are different things:

- A **PID** identifies a process.
    
- A **process handle** provides an access-controlled reference to its process object.
    

## 4.8 Process Creation Notifications

The Windows kernel provides mechanisms that allow drivers and other system components to receive notifications when processes are created or terminated.

These mechanisms are exposed through functions such as:

- `PsSetCreateProcessNotifyRoutine`
    
- `PsSetCreateProcessNotifyRoutineEx`
    
- `PsSetCreateProcessNotifyRoutineEx2`
    

A driver can register a callback through these functions. Windows subsequently invokes the callback when the relevant process creation or deletion event occurs.

This allows system components to maintain their own process-related information.

For example, a security product might maintain a separate data structure containing information about every process it monitors.

Such information could include:

- Process identifiers.
    
- Executable image information.
    
- Process creation timestamps.
    
- Security-related metadata.
    
- Monitoring or tracking state.
    

### Blocking Process Creation

Some notification mechanisms also allow a driver to prevent a process from being created.

For example, the extended callback mechanism provided by `PsSetCreateProcessNotifyRoutineEx` allows a driver to modify the creation status through the `PS_CREATE_NOTIFY_INFO` structure.

If the driver sets an appropriate failure status, Windows can reject the process creation request.

This capability can be useful for security products implementing process-blocking policies, such as executable restrictions or security-based filtering.

**Important distinction:** Not every process notification callback supports blocking process creation. The extended callback mechanism provides capabilities that are not available through the original notification interface.

### Process Overhead

When evaluating the memory overhead associated with a process, looking only at the size of `EPROCESS` is insufficient.

Other Windows components may allocate additional structures to maintain their own process-specific information.

For example, a process may have associated:

- `ETHREAD` structures for its threads.
    
- A PEB and other user-mode structures.
    
- Windows subsystem-related structures.
    
- GUI and graphics-related structures.
    
- Additional allocations maintained by third-party drivers.
    

Therefore, the total memory associated with a process can be considerably larger than the size of its `EPROCESS` structure alone.

An exact measurement is difficult because different processes use different system components and resources.

## **4.9 Inspecting EPROCESS Using WinDbg**

WinDbg provides several commands for examining the internal structures associated with Windows processes.

Since `EPROCESS` is an internal kernel structure, its fields and offsets can change between Windows versions. Instead of relying on hardcoded offsets, the kernel debugger can retrieve the structure layout from available debugging symbols.

### **Examining the EPROCESS Structure**

Use the following command:

```
dt nt!_EPROCESS
```

This displays the fields belonging to the `_EPROCESS` structure, along with their types and offsets.

To inspect a particular process, provide its EPROCESS address:

```
dt nt!_EPROCESS <EPROCESS_address>
```

This is useful when investigating individual process attributes and understanding how different fields relate to one another.

### **Displaying Process Information**

WinDbg also provides the `!process` extension.

To display information about all active processes:

```
!process 0 0
```

To inspect a particular process using its PID:

```
!process <PID> 7
```

The command displays information such as:

- EPROCESS address.
    
- Process ID.
    
- Parent process ID.
    
- PEB address.
    
- Process image name.
    
- Thread information.
    
- Process scheduling and timing information, depending on the flags used.
    

The second argument controls the amount of information displayed.

For example, `7` enables several useful output categories, including process statistics, thread information, and thread stacks.

### **Why These Commands Matter**

These commands provide a practical way to connect the structures discussed in this section with their actual kernel representations.

For example, you can:

1. Use `!process 0 0` to identify running processes.
    
2. Obtain the EPROCESS address of a particular process.
    
3. Use `dt nt!_EPROCESS` to examine its internal fields.
    
4. Identify related information, such as its PEB address and process-specific attributes.
    

This is particularly useful when studying process internals, debugging kernel components, and investigating suspicious process behavior.

## **Key Takeaways**

- `EPROCESS` is the central executive structure representing a Windows process.
    
- `ETHREAD` represents individual threads belonging to a process.
    
- `KPROCESS` is embedded inside `EPROCESS` and contains lower-level kernel process information.
    
- `PEB` resides in user-mode address space and contains information used by user-mode components.
    
- `CSR_PROCESS`, `W32PROCESS`, and `DXGPROCESS` maintain additional process-specific information for different Windows subsystems.
    
- Process objects are managed by the Object Manager, but individual process objects are generally unnamed.
    
- A PID identifies a process, whereas a process handle provides access to its underlying process object.
    
- Kernel drivers can register process notification callbacks to monitor process creation and termination.
    
- Certain extended notification mechanisms allow drivers to reject process creation.
    
- WinDbg commands such as `dt` and `!process` allow us to inspect these structures during kernel debugging.