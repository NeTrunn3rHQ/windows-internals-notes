The `CreateProcess*` family of Windows APIs accepts several arguments that control how a new process is created, configured, and started.

Understanding these arguments is important because they determine the security context, execution state, environment, and other properties of the newly created process.

A process created through these APIs initially contains **one thread**. This is its primary thread, which will eventually begin executing the program's entry-point code.

The following are the important arguments shared across the process creation APIs.

## 2.1 Security Context

The security context determines which user identity and permissions the newly created process will operate under.

The required information depends on the API being used:

- `CreateProcessAsUser` and `CreateProcessWithTokenW` accept a handle to an access token that specifies the security context for the new process.
    
- `CreateProcessWithLogonW` accepts the username, domain, and password of the account under which the process should run.
    

These arguments allow Windows to create processes under security contexts different from that of the calling process.

## 2.2 Executable Path and Command Line

These arguments specify which executable should be launched and the command-line parameters that should be passed to it.

The executable path identifies the program to execute, while the command line provides additional information that the program can use during execution.

For example:

```
notepad.exe example.txt
```

Here:

- `notepad.exe` identifies the executable.
    
- `example.txt` is an argument passed to the program.
    

## 2.3 Security Attributes

Optional security attributes can be supplied for the newly created process and its initial thread.

These attributes allow the caller to configure security-related properties of the corresponding kernel objects.

For example, they can specify security descriptors and whether the resulting handles can be inherited by another process.

## 2.4 Handle Inheritance

A Boolean argument controls whether inheritable handles from the creating process should also be made available to the newly created process.

When handle inheritance is enabled, Windows can copy eligible handles into the new process.

This is useful when a parent process needs to share existing resources with a child process, such as file handles or communication resources.

Only handles marked as inheritable are eligible for this mechanism.

## 2.5 Process Creation Flags

The creation flags control different aspects of how Windows initializes the new process and its initial thread.

Some important examples include:

### CREATE_SUSPENDED

Creates the process with its initial thread in a suspended state.

The thread exists, but it does not begin executing the program immediately. The calling process can perform additional operations before allowing execution to continue.

Calling `ResumeThread` resumes the suspended thread.

### DEBUG_PROCESS

Indicates that the newly created process should run under the debugging control of the process that created it.

This allows the creating process to receive debugging events associated with the new process.

### EXTENDED_STARTUPINFO_PRESENT

Indicates that the caller is providing a `STARTUPINFOEX` structure instead of the standard `STARTUPINFO` structure.

This enables additional configuration through extended process and thread attributes.

## 2.6 Environment Block

An optional environment block can be supplied when creating a process.

It contains environment variables that will be available to the new process, such as:

- `PATH`
    
- `TEMP`
    
- `USERPROFILE`
    

If no custom environment block is provided, the new process normally inherits the environment of its parent process.

This allows the creating process to control which environment variables are available to the program being launched.

## 2.7 Current Directory

The caller can optionally specify the initial working directory of the new process.

If no directory is provided, the newly created process inherits the current directory of the creating process.

The current directory can influence how applications resolve relative file paths and perform certain file searches.

A process can change its working directory later by calling `SetCurrentDirectory`.

It is important to distinguish the current directory from the executable's installation directory. They do not necessarily have to be the same.

## 2.8 STARTUPINFO and STARTUPINFOEX

These structures provide additional configuration information to Windows during process creation.

### STARTUPINFO

`STARTUPINFO` contains settings related to the initial execution environment of the process.

For example, it can configure aspects of the new process's window and standard input/output/error handles.

### STARTUPINFOEX

`STARTUPINFOEX` extends the standard structure by introducing an additional attribute-list field.

This attribute list allows the caller to supply extra process and thread creation settings.

The attributes are configured using the `UpdateProcThreadAttribute` function, which is called for each attribute that needs to be added.

These extended attributes provide additional control over process creation beyond what is available through the standard `STARTUPINFO` structure.

Some attributes are also used internally by Windows for specialized process creation scenarios.

## 2.9 PROCESS_INFORMATION

Unlike most of the previously discussed arguments, `PROCESS_INFORMATION` is an output structure.

When process creation succeeds, Windows populates this structure with information about the newly created process and its initial thread.

It contains four important fields:

|Field|Description|
|---|---|
|`hProcess`|Handle to the newly created process.|
|`hThread`|Handle to the initial thread of the process.|
|`dwProcessId`|Unique identifier assigned to the new process.|
|`dwThreadId`|Unique identifier assigned to the initial thread.|

The returned handles allow the creating process to interact with the newly created process and thread.

For example, they can be used to query process information, wait for process termination, or manipulate the initial thread.

The process and thread handles should eventually be released using `CloseHandle` when they are no longer needed.

---

## Key Takeaways

- A process created through the `CreateProcess*` APIs initially contains one thread.
    
- The access token determines the security identity under which the new process operates.
    
- Handle inheritance allows eligible handles to be shared between the creating and newly created processes.
    
- Creation flags influence how the process and its initial thread are initialized.
    
- The environment block and current directory determine important parts of the new process's initial execution environment.
    
- `STARTUPINFOEX` provides additional configuration through extended process and thread attributes.
    
- `PROCESS_INFORMATION` provides the IDs and handles needed to interact with the newly created process and its initial thread.