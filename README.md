# Group-Policy-Management-Lab
Hands\-on **CompTIA guided lab** practising Active Directory and Group Policy Management in a Windows Server environment\. Created and scoped an OU, configured account lockout policies, deployed logon/logoff scripts and a folder using GPO Preferences, then verified policy application using **`gpresult`**** and RSoP**\.

# Group Policy Management Lab (Account Lockout, Logon/Logoff Scripts, OU Structure and Folder Deployment on Windows Server AD)

`Group Policy Management` · `Active Directory` · `Windows Server` · `GPMC` · `RSoP` · `CompTIA Labs`

## Overview
This lab was hands-on practice with **Group Policy Objects (GPOs)** in a guided CompTIA Active Directory environment, built around a single domain controller (`DC10.ad.structureality.com`) managing a Windows 10 client (`PC10`). The scenario was a "Sales" department that needed its own organizational structure, account lockout security policy, logon/logoff scripts, and an automatically-created shared documents folder, all delivered through Group Policy instead of touching the client machine directly.

The goal wasn't just to get each setting applied, but to verify it actually took effect on the client side using `gpresult` and the RSoP (Resultant Set of Policy) snap-in, rather than assuming a policy worked just because the GPO editor didn't show an error.

## Objective
Build a dedicated Organizational Unit for a sales team, apply an account lockout policy to it, deploy logon/logoff scripts through Group Policy, and use GPO Preferences to auto-create a shared folder on client machines, then confirm all of it actually applied on the client with command-line and GUI verification tools.

## Environment
- **Domain Controller:** `DC10.ad.structureality.com` (Windows Server, managed via Server Manager / GPMC)
- **Client machine:** `PC10` (Windows 10, domain-joined to `ad.structureality.com`)
- **Test users:** `Dani`, `Cam` (moved into the new OU to receive the policy)
- **Domain:** `ad.structureality.com`
- **Lab platform:** CompTIA Learning Platform, hosted lab environment

## Tools I Used

| Tool | What It Does | Why I Used It |
|------|--------------|----------------|
| **Active Directory Users and Computers (ADUC)** | AD object management | Created the `SalesClients` OU and moved user/computer objects into it |
| **Group Policy Management Console (GPMC)** | GPO creation/linking | Created and linked the `SalesPolicy` GPO to the domain, scoped by OU |
| **Group Policy Management Editor** | GPO editing | Configured Account Lockout Policy, logon/logoff scripts, and folder preferences |
| **Notepad** | Script authoring | Wrote simple `.txt` logon/logoff message scripts to confirm script execution |
| **Command Prompt** | Batch scripting / verification | Wrapped scripts as `.cmd` files pointing to the UNC script path; ran `gpresult /r` and `rsop` |
| **RSoP (Resultant Set of Policy) snap-in** | Policy verification | Confirmed which GPOs actually applied to the client and in what order |
| **File Explorer / Share & NTFS Permissions** | Access control | Set `Everyone` share permissions on the `scripts` folder so clients could read the logon/logoff scripts |

## What I Did

### 1. Building the OU Structure
1. Opened **Active Directory Users and Computers** from Server Manager's Tools menu.
2. Right-clicked the domain root (`ad.structureality.com`) → **New → Organizational Unit**, named it `SalesClients`, and left "Protect container from accidental deletion" checked.
3. Selected the `Dani` user account and used **Move…** to relocate it from its default container into the new `SalesClients` OU, repeating for the `Cam` user account and the `PC10` computer object so both the user and computer side of policy could be tested.

### 2. Creating and Scoping the GPO
1. In **Group Policy Management**, right-clicked the domain and selected **Create a GPO in this domain, and Link it here…**, naming it `SalesPolicy`.
2. Reviewed the GPO's **Scope** tab to confirm it linked to `ad.structureality.com` with Link Enabled = Yes, and Security Filtering scoped to Authenticated Users by default.

### 3. Configuring Account Lockout Policy
1. Navigated to **Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy**.
2. Opened **Account Lockout Threshold**, checked **Define this policy setting**, and set it to `3` invalid logon attempts.
3. Windows Server prompted with a **Suggested Value Changes** dialog, since defining the threshold with the duration/reset counter left undefined doesn't make sense, offering to set both **Account lockout duration** and **Reset account lockout counter after** to `30 minutes`. Accepted the suggested values rather than leaving them undefined.
4. Confirmed the final policy setting: **3 invalid logon attempts**, **30-minute lockout duration**, **30-minute reset counter**.

### 4. Writing and Deploying Logon/Logoff Scripts
1. On the DC, created a `C:\scripts` folder and wrote two plain-text files in Notepad:
   - `logon.txt` — `Hello, this is a logon script.`
   - `logoff.txt` — `Goodbye, this is a logoff script.`
2. Since GPO logon/logoff script fields expect an executable script rather than a raw text file, wrapped each in a `.cmd` batch file using `echo` redirection from the command line:
   ```
   echo \\DC10\scripts\logon.txt > c:\scripts\scriptlogon.cmd
   echo \\DC10\scripts\logoff.txt > c:\scripts\scriptlogoff.cmd
   ```
3. Verified the four resulting files with `dir c:\scripts`, confirming `logon.txt`, `logoff.txt`, `scriptlogon.cmd`, and `scriptlogoff.cmd` were all present with correct byte sizes.
4. In the GPO editor, under **User Configuration → Policies → Windows Settings → Scripts (Logon/Logoff)**, opened **Logon Properties → Add a Script**, and pointed it at `\\DC10\scripts\scriptlogon.cmd`, repeating the process for the logoff script.
5. Set share permissions on the `scripts` folder to grant **Everyone: Full Control / Change / Read**, since a client authenticating over the network needs to actually reach the UNC path to pull the script at logon.
6. Enabled the **Display instructions in logoff scripts as they run** Administrative Template policy so the logoff script's output would visibly display in a command window instead of running silently, useful for confirming execution during testing.

### 5. Deploying a Shared Folder via GPO Preferences
1. Under **User Configuration → Preferences → Windows Settings → Folders**, right-clicked and selected **New → Folder**.
2. Set **Action: Create**, **Path: C:\SalesDocs**, leaving Read-only/Hidden unchecked and Archive checked by default.
3. This uses GPO Preferences (not a Policy) to push a folder-creation instruction to any client the GPO applies to, rather than manually creating the folder on each machine.

### 6. Verifying the Policy Actually Applied
1. Logged into `PC10` as the `Dani` user (moved into `SalesClients` earlier) to trigger the linked GPO on next logon.
2. Ran `gpresult /r` from Command Prompt on the client and confirmed:
   - `CN=Dani,OU=SalesClients,DC=ad,DC=structureality,DC=com`
   - Last Group Policy applied: from `DC10.ad.structureality.com`
   - Confirmed the user object's location in the directory matched the OU the policy was scoped to, not just that *a* policy applied.
3. Opened the **RSoP (Resultant Set of Policy)** snap-in against `PC10`, walking through the "Resultant Set of Policy is being processed…" wizard (logging mode, targeting user `structureality\Cam` and computer `structureality\PC10`), then reviewed the applied Account Lockout settings under Computer Configuration to confirm `SalesPolicy` was listed as the **Source GPO** for the lockout duration/threshold/reset counter values, alongside the default domain policy.
4. Also ran `gpresult /r` a second time from an elevated prompt to cross-check applied GPOs (`uu-domain-default`, `SalesPolicy`) and confirmed group memberships being evaluated (Domain Users, Everyone, Domain Admins, etc.) matched expectations for the test account.

## What's in This Repo

```
group-policy-lab/
├── README.md                          # This file
├── scripts/
│   ├── logon.txt                      # Logon script message
│   ├── logoff.txt                     # Logoff script message
│   ├── scriptlogon.cmd                # Batch wrapper pointing to logon.txt
│   └── scriptlogoff.cmd               # Batch wrapper pointing to logoff.txt
└── screenshots/
    ├── 01-new-ou-salesclients.png
    ├── 02-move-user-into-ou.png
    ├── 03-create-gpo-salespolicy.png
    ├── 04-account-lockout-threshold.png
    ├── 05-suggested-value-changes.png
    ├── 06-scripts-logon-logoff-cmd.png
    ├── 07-add-script-gpo.png
    ├── 08-scripts-folder-permissions.png
    ├── 09-display-logoff-instructions-policy.png
    ├── 10-folder-preference-salesdocs.png
    ├── 11-gpresult-r-output.png
    └── 12-rsop-verification.png
```

## Skills I Picked Up
- **Scoping policy with OUs instead of the whole domain,** creating a dedicated `SalesClients` OU and moving both user and computer objects into it so a GPO could be tested against a realistic, limited target rather than every object in the domain.
- **Understanding why GPO script fields need executables, not text files,** and using a simple `echo` redirection trick to wrap a plain-text message into a `.cmd` file the Scripts policy would actually accept.
- **Recognizing that share permissions matter even when NTFS permissions look fine,** since a domain client pulling a logon script over `\\DC10\scripts\` needs read access at the share level, not just the file system level.
- **Telling GPO Policies and GPO Preferences apart,** using Preferences (Folders) for the "create this if it doesn't exist" folder deployment, versus Policies for enforced settings like account lockout.
- **Verifying policy application from the client side, not just the GPO editor,** using `gpresult /r` and RSoP to confirm both the group policy's origin and the OU path of the affected object, rather than assuming a linked GPO worked because no error appeared.
- **Reading a "Suggested Value Changes" prompt correctly,** understanding that Account Lockout Threshold, Duration, and Reset Counter are interdependent settings and accepting the suggested values rather than leaving related settings undefined.

## How This Applies in the Real World
Account lockout policy is one of the most common baseline hardening settings in any AD environment, it's a direct mitigation against password-guessing and brute-force attacks against domain accounts. Scoping it to specific OUs (rather than blanket domain policy) reflects how real organizations often need different lockout thresholds for different groups (e.g., stricter policy for privileged accounts, more lenient for shared kiosk machines).

Logon/logoff scripts are a common real-world mechanism for mapping drives, running compliance checks, or displaying legal/security banners at every session start, and understanding how they're deployed (and why they sometimes silently fail if share permissions are wrong) is directly useful for troubleshooting real helpdesk tickets. Folder redirection/creation via GPO Preferences is likewise a standard way organizations standardize where user data lands across a fleet of machines without touching each one by hand.

## Where I'm Coming From
I'm making the jump into cybersecurity from a background in **healthcare**. A lot of the muscle memory carries over: following procedures carefully, protecting sensitive information, and staying calm and methodical when something isn't working the way it's supposed to. I'm currently studying for **CompTIA Security+** and building labs like this one to get real hands-on reps in with Active Directory and Group Policy, since that's what I'm missing on paper right now compared to my hands-on time in this lab environment.

## What I Want to Learn Next
- Applying Group Policy Security Filtering to scope a GPO to a specific security group rather than an entire OU
- Building more complex logon scripts (PowerShell-based) that do more than print a message, e.g., mapping network drives or logging session start times
- Exploring GPO inheritance and Block Inheritance behavior across nested OUs
- Practicing troubleshooting a GPO that *isn't* applying, deliberately breaking share permissions or WMI filtering to diagnose it from the client side

## Limitations & What I'd Do Differently in Production
- **Scripts here were intentionally trivial** (a printed message) to focus on the deployment mechanism itself. Production logon scripts would need error handling and logging.
- **Share permissions were set broadly to `Everyone: Full Control`** for lab simplicity. In production, this should be scoped down to `Authenticated Users` with Read-only access, since logon scripts should never need to be writable by the clients executing them.
- **Only one GPO was tested at a time.** A real environment would need to account for GPO precedence and conflicts across multiple linked policies, which wasn't exercised here.
- **RSoP was run in logging mode against a single client.** A production troubleshooting workflow would also lean on `gpresult /h` for an HTML report and the Group Policy Modeling Wizard to simulate policy application before deployment.

## References
- [Group Policy Overview (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/adac/active-directory-administrative-center)
- [Account Lockout Policy Reference](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/account-lockout-policy)
- [Group Policy Preferences Overview](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/cc731892(v=ws.10))
- [CompTIA Security+ (SY0-701) Exam Objectives](https://www.comptia.org/certifications/security)
