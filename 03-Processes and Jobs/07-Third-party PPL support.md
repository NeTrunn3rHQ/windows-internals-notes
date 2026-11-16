Protected Process Light is not limited to components developed by Microsoft. Windows also provides a mechanism that allows approved third-party anti-malware vendors to run their security services as PPL processes.

This is important because anti-malware software is itself a high-value target. If malware can compromise or disable the anti-malware service, it may be able to operate without being detected.

PPL helps defend against this by restricting what ordinary user-mode processes can do to the anti-malware service.

## 7.1 Typical Anti-Malware Architecture

A typical anti-malware (AM) product can be divided into three major components:

|Component|Responsibility|
|---|---|
|Kernel-mode driver|Monitors system activity and enforces security policies. It may use file-system or network filtering, as well as process, thread, and object callbacks.|
|User-mode service|Runs with the privileges required by the product. It manages detection policies, processes notifications from the driver, performs malware analysis, and may communicate with external servers.|
|User-mode GUI|Provides an interface through which users can view security information, receive alerts, and configure available options.|

The kernel driver and user-mode service work together to provide the product's core protection capabilities.

For example, the driver might detect an attempt to access a suspicious file and notify the user-mode service. The service can then analyze the event and determine what action should be taken.

The GUI is primarily responsible for presenting information and allowing user interaction. It does not necessarily participate in the security-critical operations performed by the other components.

**Note:** This is a common architecture, not a mandatory design. Anti-malware products can organize their components differently.

## 7.2 Why Anti-Malware Services Need PPL

An anti-malware service is an attractive target because it controls important parts of the product's detection and response mechanisms.

Consider an attack in which malware obtains administrative privileges on a system.

Without additional process protection, the attacker may attempt to:

- Open the anti-malware service process with powerful access rights.
    
- Read or modify its memory.
    
- Inject malicious code into the service.
    
- Create remote threads to execute code inside it.
    
- Terminate the service or interfere with its operation.
    

If an attacker successfully compromises the service, they may be able to interfere with malware detection, manipulate security policies, or disable important protection features.

Running the service as PPL makes these attacks significantly more difficult.

Windows restricts access from ordinary, unprotected processes. In particular, they cannot inject threads into the protected service or write to its virtual memory through normal process-access mechanisms.

Windows also prevents unprotected processes, including administrators, from stopping an anti-malware protected service through ordinary service-control operations.

This does not make the service invulnerable. Kernel-level compromise, vulnerabilities in trusted components, and other attacks outside the protection model can still pose a threat.

The purpose of PPL is to establish a stronger security boundary against attacks originating from less-trusted user-mode processes.

## 7.3 The Role of Early Launch Anti-Malware (ELAM)

Windows requires an approved anti-malware vendor to have an **Early Launch Anti-Malware (ELAM) driver** installed before its user-mode service can be registered and launched as a protected service.

ELAM was introduced to allow anti-malware software to participate early in the Windows boot process.

Its primary boot-time responsibility is to examine other boot-start drivers and provide Windows with information about their trustworthiness. Windows uses this information when deciding whether those drivers should be initialized.

ELAM also plays an important role in registering the certificates that Windows will trust for the vendor's protected user-mode service.

The ELAM driver must satisfy Microsoft's specific certification and signing requirements. It is not enough for a vendor to install an ordinary kernel-mode driver and request PPL protection.

### How ELAM Establishes Trust for the Service

The ELAM driver contains an embedded resource section with certificate information for the anti-malware service binaries.

Windows extracts and validates this information, allowing the Code Integrity system to recognize binaries signed with the registered certificates.

The vendor can then register its service with Windows and configure it to launch as an anti-malware protected service.

This creates a chain of trust:

```
Microsoft-approved ELAM driver
              |
              v
Certificate information embedded
in the ELAM driver's resources
              |
              v
Windows validates and registers
the certificate information
              |
              v
Anti-malware service is configured
to launch as a protected service
              |
              v
Code Integrity validates the service
and its permitted executable images
              |
              v
       Anti-malware PPL
```

The important point is that the vendor cannot simply assign itself the anti-malware protection level. Windows must first recognize the vendor's signing information through the ELAM registration process.

## 7.4 ELAMCERTIFICATEINFO Resource Section

The ELAM driver includes a custom resource section containing certificate information used to identify binaries that are eligible for anti-malware protected execution.

The resource is commonly referred to as `ELAMCERTIFICATEINFO` in the Windows Internals discussion.

It can contain up to three certificate entries. Each entry identifies a signing certificate through its certificate hash and specifies the applicable hashing algorithm.

An entry can also include up to three Extended Key Usage (EKU) identifiers.

These EKUs are represented using Object Identifiers (OIDs).

### Certificate Entries and EKUs

The resource information allows Windows to associate approved signing certificates with particular EKU requirements.

For example, a vendor may use different certificates or EKUs to distinguish between multiple products or services from the same company.

The general structure is:

|Information|Purpose|
|---|---|
|Certificate hash|Identifies the certificate used to sign the anti-malware service binaries.|
|Hash algorithm|Specifies the algorithm used to calculate the certificate hash.|
|EKU identifiers|Specify additional certificate usage requirements, when applicable.|

A simplified representation of the resource information is:

```
ELAMCERTIFICATEINFO
    |
    +-- Certificate Entry 1
    |       +-- Certificate Hash
    |       +-- Hash Algorithm
    |       +-- EKU information
    |
    +-- Certificate Entry 2
    |       +-- Certificate Hash
    |       +-- Hash Algorithm
    |       +-- EKU information
    |
    +-- Certificate Entry 3
            +-- Certificate Hash
            +-- Hash Algorithm
            +-- EKU information
```

Each certificate entry can contain up to three EKU identifiers. When multiple EKUs are specified for an entry, the certificate must satisfy all of them.

The Code Signing EKU is also required in the certificate chain, but it is not included as one of the EKU values in this resource entry.

**Important:** The resource does not simply list arbitrary public keys. The documented format identifies certificates using their hashes, along with the associated algorithm and optional EKU information.

## 7.5 Registering and Launching the Anti-Malware Service

After the ELAM driver and its certificate information are available, the anti-malware vendor must register the service with Windows before starting it as protected.

Windows provides the `InstallELAMCertificateInfo` API for this purpose.

The API receives a handle to the ELAM driver file. Windows validates the driver and extracts the certificate information embedded in its resources.

Once registration succeeds, the vendor can configure the service to launch using the protected anti-malware service mode.

The general process is:

1. Install the approved ELAM driver.
    
2. Register its certificate information with Windows.
    
3. Create the anti-malware service using the Service Control Manager.
    
4. Configure the service's launch-protection setting.
    
5. Start the service.
    

The relevant Windows service configuration uses:

```
SERVICE_LAUNCH_PROTECTED_ANTIMALWARE_LIGHT
```

This tells Windows that the service should be launched using the anti-malware protected-service level.

The Service Control Manager works with Code Integrity to validate the service's signing information before launching it as protected.

### Important APIs and Structures

|API or structure|Purpose|
|---|---|
|`InstallELAMCertificateInfo`|Registers certificate information embedded in the ELAM driver.|
|`CreateService`|Creates the service entry in the Service Control Manager database.|
|`ChangeServiceConfig2`|Configures additional service properties, including launch protection.|
|`SERVICE_LAUNCH_PROTECTED_INFO`|Structure used to specify the service's launch-protection configuration.|
|`StartService`|Starts the configured service.|

This process ensures that only a service satisfying the required trust and signing conditions can use the anti-malware protected-service mechanism.

## 7.6 Example: Microsoft Defender

Microsoft Defender Antivirus is an example of an anti-malware product that uses protected-service functionality.

On Windows 10, two examples discussed in this context are:

- `MsMpEng.exe` — Microsoft Defender Antivirus's main antimalware service.
    
- `NisSvc.exe` — Network Inspection Service.
    

These services use anti-malware protection to make it harder for ordinary user-mode malware to interfere with their operation.

The protection is particularly important because an attacker who can disable the security service may be able to carry out additional malicious activity with a reduced chance of detection.

The exact processes and their protection configurations can vary across Windows versions and system configurations.

## 7.7 Key Takeaways

- PPL can protect approved third-party anti-malware services, not just Microsoft components.
    
- Anti-malware services are high-value targets because compromising them can weaken the system's defenses.
    
- An approved ELAM driver is required for registering a third-party anti-malware service as protected.
    
- The ELAM driver contains certificate information in an embedded resource section.
    
- Windows uses this information and Code Integrity validation to establish whether the service is eligible for protected execution.
    
- `InstallELAMCertificateInfo` registers the certificate information with Windows.
    
- `SERVICE_LAUNCH_PROTECTED_ANTIMALWARE_LIGHT` specifies the anti-malware protected-service launch mode.
    
- PPL restricts attacks from ordinary user-mode processes, but it is not a guarantee against kernel-level compromise or every possible attack.