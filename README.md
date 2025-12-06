# PDAcl – Play-Doh ACL

Play-Doh ACL (PDAcl) is a Windows-based command-line tool that helps users inspect and modify access control lists (ACLs) attached to Active Directory (AD) objects and Windows services. It combines low-level Win32 and ADSI APIs into a single executable file that can be scripted, allowing you to add, remove, or audit permissions without opening GUI consoles.

PDAcl currently supports three focused workflows:

- `ADE`: Manage AD **Extended Rights** (control access rights handled by GUIDs).
- `AD`: Manage classic AD **object rights** (delete, DAC write, full control, etc.).
- `SC`: Manage Windows **Service Control Manager** rights on individual services.

## Features

- **Granular ACL editing for AD objects** – add or revoke any standard ADS rights bit using a single command.
- **Extended-right GUID support** – operate on the full catalog of Microsoft-defined extended rights without memorizing GUIDs.
- **Service DACL management** – grant or revoke SCM permissions such as `SERVICE_START`, `SERVICE_CHANGE_CONFIG`, or `SERVICE_ALL_ACCESS`.
- **Built-in credential impersonation** – optional `--login Domain/User@Password` flag performs LogonUser/Impersonate for alternate credentials.
- **CLI11-powered UX** – subcommand-based syntax with auto-generated help, mutually exclusive switches, and friendly error reporting.
- **No external runtime** – statically links against Win32 / ADSI libraries (`comsuppw`, `Adsiid`, `Activeds`) for drop-and-run usage on Windows hosts.


## Installation & Build

1. **Prerequisites**
   - Windows 10/11 with the Active Directory Domain Services (AD DS) RSAT components installed.
   - Visual Studio 2022 with the Desktop development with C++ workload.

2. **Build with Visual Studio GUI**
   - Open `PDAcl.sln`.
   - Pick `Release | x64` or `Debug`.
   - Build Solution (`Ctrl+Shift+B`). The binary drops into `x64\*\PDAcl.exe`.

4. **Distribution**
   - Ship `PDAcl.exe` as a standalone binary. No additional DLLs are required beyond the standard Windows runtime.

## Configuration

- **Alternate Credentials**: Provide `--login Domain/User@Password` on any subcommand. PDAcl parses the string, invokes `LogonUserA`, and impersonates for the lifetime of the command.
- **LDAP Targeting**: Pass `--server` with an LDAP path (e.g., `DC=example,DC=com` or `CN=User,OU=Admins,DC=example,DC=com`). PDAcl automatically prefixes `LDAP://`.
- **User Principals**: Supply the trustee via `--user`. For AD operations use `Domain\User`; for service ACLs any local/AD principal recognized by the SCM works (`Everyone`, `NT AUTHORITY\SYSTEM`, etc.).
- **Rights Identifiers**: `--extended-right` must match a key from the built-in maps. Use `--list` first if unsure.


## Usage Examples

### 1. Enumerate extended rights

```powershell
.\PDAcl.exe ADE --list
```

### 2. Grant AD extended right (e.g., `DS-Replication-Get-Changes`)

```powershell
.\PDAcl.exe ADE `
  --add `
  --extended-right DS-Replication-Get-Changes `
  --user CONTOSO\svcReplication `
  --server "CN=Configuration,DC=contoso,DC=com"
```

### 3. Revoke a standard AD right (e.g., `ADS-Right-Generic-All`)

```powershell
.\PDAcl.exe AD `
  --remove `
  --extended-right ADS-Right-Generic-All `
  --user CONTOSO\DevOps `
  --server "CN=Tier0,OU=Groups,DC=contoso,DC=com"
```

### 4. Grant service start/stop rights to `Domain Users`

```powershell
.\PDAcl.exe SC `
  --add `
  --service Spooler `
  --user "Domain Users" `
  --extended-right Service-Start
```

### 5. Operate with alternate credentials

```powershell
.\PDAcl.exe AD `
  --list `
  --login CONTOSO/adminuser@Str0ngP@ss!
```

> All subcommands share the same option names (`--add`, `--remove`, `--extended-right`, `--user`, `--server`, `--list`, `--login`) for familiarity.

## Extended Right & Service Right Catalogs

- **Active Directory Extended Rights (`ADE --list`)**: Includes every GUID defined in `active_directory.h`, such as `Abandon-Replication`, `Change-Schema-Master`, `Enable-Per-User-Reversibly-Encrypted-Password`, `User-Force-Change-Password`, and MSMQ-specific permissions.
- **Standard AD Rights (`AD --list`)**: Covers `ADS_RIGHT_GENERIC_ALL`, `ADS_RIGHT_WRITE_OWNER`, directory-specific flags like `ADS_RIGHT_DS_CREATE_CHILD`, and control bits such as `ADS_RIGHT_DS_CONTROL_ACCESS`.
- **Service Rights (`SC --list`)**: Maps friendly names (`Service-Start`, `Service-Stop`, `Service-Change-Config`, `Generic-Read`, etc.) to the native `SERVICE_*` or `GENERIC_*` constants exposed by the Windows SCM APIs.

Refer to the CLI `--list` output to view the authoritative set shipped with your build.


## Troubleshooting

- **`OpenSCManager Error` / `OpenService Error`**: Ensure you run from an elevated prompt and that the target service name is correct. Remote service ACL editing is not supported yet.
- **`ADsGetObject Error`**: Double-check the LDAP path string and verify RSAT / ADSI components are installed. Use fully distinguished names.
- **`Not Found ActiveDirectory Right`**: Call `ADE --list` or `AD --list` to copy the exact key. The lookup is case-sensitive.
- **`Can't Login`**: The `--login` parser expects the `Domain/User@Password` pattern. Escapes spaces in PowerShell with quotes.
- **Release build crashes immediately**: Confirm the target host has ADSI (activeds.dll). This is present on domain-joined systems by default.

## License

This project is released under the [MIT License](LICENSE).

