Windows provides multiple APIs for creating a new process. The API used depends on the security context under which the new process needs to run.

## 1.1 Process Creation APIs

### CreateProcess

`CreateProcess` is the standard Windows API for creating a new process. By default, the newly created process inherits the security context (access token) of the calling process.

In other words, the new process normally runs under the same user and security identity as the process that created it.

### CreateProcessAsUser

When a process needs to be created under a different security context, Windows provides `CreateProcessAsUser`.

Unlike `CreateProcess`, this function accepts an additional parameter: a handle to an access token. This token determines the security identity under which the new process will run.

A token can be obtained through functions such as `LogonUser`, depending on the required authentication and privileges.

### CreateProcessWithTokenW and CreateProcessWithLogonW

Windows also provides two additional APIs through `advapi32.dll`:

- **CreateProcessWithTokenW:** Creates a process using an already obtained access token. Its privilege requirements differ from those of `CreateProcessAsUser`.
    
- **CreateProcessWithLogonW:** Allows a caller to provide another user's credentials and create a process under that user's security context without separately obtaining and supplying a token.
    

Both functions rely on the **Secondary Logon service (seclogon)** to perform the actual process creation.

The general execution flow is:

1. The API communicates with the Secondary Logon service through RPC (Remote Procedure Call).
    
2. The service handles the request internally through `SlrCreateProcessWithLogon`.
    
3. The service eventually uses `CreateProcessAsUser` to create the requested process.
    

The Secondary Logon service runs inside a `svchost.exe` process and is configured to start on demand by default. If the service is disabled or cannot start, these process creation APIs will fail.

The Windows `runas` command-line utility also relies on this functionality to launch programs under different user credentials.

## 1.2 CreateProcess vs ShellExecute

An important distinction exists between creating a process directly and asking Windows to open a file.

The documented process creation APIs expect an executable image, such as a valid Portable Executable (PE) file, a batch file, or a supported 16-bit COM application. They do not independently determine which application should handle an arbitrary file extension.

For example, calling `CreateProcess` with a `.txt` file does not automatically launch Notepad.

This association between file types and their corresponding applications is handled by the **Windows Shell**.

Windows provides two relevant APIs:

- `ShellExecute`
    
- `ShellExecuteEx`
    

These functions can accept files that are not executable programs. They use file associations and registry information, particularly under `HKEY_CLASSES_ROOT`, to determine which application should handle the requested file.

For example, when a user requests to open a `.txt` file:

1. The Shell identifies the application associated with the `.txt` extension.
    
2. It determines the executable responsible for handling that file (for example, Notepad).
    
3. It constructs the appropriate command line, including the file path as an argument.
    
4. It eventually uses `CreateProcess` to launch the application.
    

This is how Windows translates a user's request to open a document into the execution of an appropriate program.

## 1.3 The Internal Process Creation Path

Although Windows exposes several process creation APIs, their execution paths eventually converge on a common internal function.

The general flow looks like this:

```
Application
    |
    +-- CreateProcess
    |
    +-- CreateProcessAsUser
    |
    +-- CreateProcessWithTokenW
    |
    +-- CreateProcessWithLogonW
    |
    +-- ShellExecute / ShellExecuteEx
             |
             +-- CreateProcess
                    |
                    v
             CreateProcessInternal
                    |
                    v
             NtCreateUserProcess (ntdll.dll)
                    |
                    v
             Transition to Kernel Mode
                    |
                    v
             NtCreateUserProcess (Windows Executive)
```

`CreateProcessInternal` is an internal Windows function responsible for coordinating the user-mode portion of process creation.

Eventually, it invokes `NtCreateUserProcess`, which is exposed through `ntdll.dll`.

This function acts as the Native API boundary between user mode and kernel mode. The request transitions into kernel mode, where the corresponding `NtCreateUserProcess` implementation in the Windows Executive continues the process creation operation.

**Key takeaway:** Regardless of which high-level API initiates the operation, process creation eventually reaches the native system call `NtCreateUserProcess`, which begins the kernel-mode portion of the process creation workflow.