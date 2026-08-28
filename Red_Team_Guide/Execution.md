
> Dev Note: Header formatting is broken; I will fix them later 
---

## Metasploitable 

> **Scope:** Authorized penetration tests, internal red-team engagements, vulnerability validation, and isolated labs. Never run an exploit, credential attack, persistence mechanism, or disruptive module against a system unless it is explicitly within the engagement scope.

---

## 1. Metasploit Fundamentals & Workflow

### 1.1 What Metasploit actually is

Metasploit is not simply an "exploit launcher."

Think of it as a framework containing:

```text
Recon
  ↓
Enumeration
  ↓
Vulnerability validation
  ↓
Exploitation
  ↓
Payload delivery
  ↓
Session
  ↓
Post-exploitation
  ↓
Evidence / reporting
```

The major module families support different stages of that process.

| Module      | Purpose                                                                                  | Typical phase               |
| ----------- | ---------------------------------------------------------------------------------------- | --------------------------- |
| `auxiliary` | Scanning, enumeration, protocol interaction, login testing and other non-payload actions | Recon / validation          |
| `exploit`   | Exploits a vulnerability to obtain code execution                                        | Initial access              |
| `payload`   | Code executed after exploitation                                                         | Session                     |
| `post`      | Actions performed after obtaining a session                                              | Post-exploitation           |
| `encoder`   | Payload transformation, principally for representation/bad-character constraints         | Payload generation          |
| `nop`       | NOP/shellcode padding                                                                    | Exploit development         |
| `evasion`   | Specialized security-control testing                                                     | Controlled security testing |

Metasploit's own documentation emphasizes that auxiliary modules are frequently used during real penetration tests and that many assessments can accomplish their objectives without firing an exploit.

---

### 1.2 Start Metasploit

```bash
msfconsole
```

Useful startup commands:

```text
version
help
?
banner
exit
```

Update the framework through your operating system/package-management method rather than manually replacing framework files.

---

### 1.3 The most important commands

#### Search

```text
search <keyword>
```

Examples:

```text
search apache
search smb
search wordpress
search type:auxiliary smb
search type:exploit windows
search cve:2021-44228
```

Metasploit supports search filters, including module type.

### Use a module

```text
use <module>
```

Example:

```text
use auxiliary/scanner/http/title
```

### Show information

```text
info
```

More detailed module documentation:

```text
info -d
```

Metasploit can generate module-specific documentation with `info -d`, including usage information and, where available, deeper knowledge-base material.

### Show options

```text
show options
```

### Show module-specific payloads

```text
show payloads
```

### Show targets

```text
show targets
```

### Show advanced options

```text
show advanced
```

### Show compatible payloads

```text
show payloads
```

### Check the target

If supported:

```text
check
```

Never assume `check` guarantees exploitation will succeed. It is a validation mechanism, not proof that exploitation is safe or successful.

---

## 1.4 The basic module workflow

For almost every module:

```text
search
   ↓
use
   ↓
info
   ↓
show options
   ↓
set required options
   ↓
check
   ↓
run/exploit
   ↓
verify result
```

Do not skip `info`.

A professional operator should understand:

* What the module does
* What versions are affected
* Required conditions
* Required options
* Side effects
* Target selection
* Payload compatibility
* Whether `check` is supported

---

## 1.5 Global vs module options

Metasploit has global configuration and module-specific configuration.

Useful commands:

```text
show options
show advanced
set RHOSTS 10.10.10.20
set RPORT 80
```

Clear a setting:

```text
unset RHOSTS
```

Clear all module settings:

```text
unset all
```

Global options:

```text
setg RHOSTS 10.10.10.0/24
setg LHOST 10.10.10.5
```

Clear global settings:

```text
unsetg RHOSTS
unsetg LHOST
```

Be careful with `setg`: an old global value can silently affect later modules.

---

# 2. Auxiliary Modules — Reconnaissance, Enumeration & Validation

Auxiliary modules are one of the most important parts of Metasploit.

They generally **do not deliver an exploit payload**.

They can perform:

* Service discovery
* Protocol enumeration
* Version detection
* Login testing
* Information gathering
* Configuration checks
* Vulnerability-adjacent validation
* Controlled administrative operations

Metasploit categorizes auxiliary modules into areas such as `scanner`, `gather`, `admin`, `client`, `server`, `sniffer`, and `dos`.

For a red team, think:

> **Auxiliary = understand the target before exploiting it.**

---

## 2.1 Scanner modules

Typical structure:

```text
auxiliary/scanner/<protocol>/<scanner>
```

Examples:

```text
auxiliary/scanner/http/title
auxiliary/scanner/http/http_version
auxiliary/scanner/ssh/ssh_version
auxiliary/scanner/smb/smb_version
```

Find them:

```text
search type:auxiliary scanner smb
```

### Why use scanners?

Suppose you have:

```text
10.10.10.0/24
```

You don't want to manually interrogate every host.

A scanner can establish:

```text
IP
 ↓
port/service
 ↓
version
 ↓
potential attack surface
```

Example:

```text
use auxiliary/scanner/http/title
set RHOSTS 10.10.10.0/24
set RPORT 80
set THREADS 10
run
```

### THREADS

Scanner modules often support:

```text
set THREADS 10
```

More threads are not automatically better.

Increasing concurrency can:

* Increase network noise
* Trigger rate limiting
* Overload fragile services
* Make results less reliable

Start conservatively.

---

## 2.2 HTTP modules

HTTP modules are useful for:

```text
Web server identification
HTTP headers
Titles
Directories
Applications
Technology
Authentication
Web-specific vulnerabilities
```

Search:

```text
search type:auxiliary http
```

Typical workflow:

```text
HTTP discovery
     ↓
Identify application
     ↓
Determine version
     ↓
Search for relevant modules
     ↓
Check vulnerability
     ↓
Exploit only if authorized
```

Do not immediately jump from:

```text
Apache detected
```

to:

```text
Exploit Apache
```

The exact version, configuration and application matter.

---

## 2.3 SMB modules

SMB is particularly important in Windows environments.

Search:

```text
search type:auxiliary smb
```

Useful enumeration can include:

* SMB version
* Host information
* Shares
* Domain information
* Authentication behavior
* Known vulnerability checks

A useful workflow:

```text
SMB discovered
   ↓
Enumerate version
   ↓
Enumerate configuration
   ↓
Identify domain/workgroup
   ↓
Check known vulnerabilities
   ↓
Only then consider exploitation
```

---

## 2.4 SSH modules

Search:

```text
search type:auxiliary ssh
```

Useful for:

* SSH version identification
* Authentication testing
* Protocol enumeration

A red-team engagement should distinguish between:

```text
authentication validation
```

and:

```text
uncontrolled password guessing
```

If credential testing is authorized, establish:

* Username scope
* Password source
* Attempt limits
* Rate limits
* Lockout handling

before running a login scanner.

---

## 2.5 Database modules

Metasploit contains modules for numerous database technologies.

Search:

```text
search type:auxiliary mysql
search type:auxiliary postgres
search type:auxiliary mssql
```

Use them when you need to establish:

```text
Database exposed?
        ↓
Version?
        ↓
Authentication?
        ↓
Configuration?
        ↓
Known vulnerability?
```

Do not dump large quantities of production data merely because a database module permits it. Collect the minimum evidence required by the engagement.

---

## 2.6 SNMP modules

SNMP can expose valuable infrastructure information.

Search:

```text
search type:auxiliary snmp
```

Potential information includes:

* Hostname
* Network information
* System description
* Interfaces
* Software information

This is particularly useful during internal-network assessments.

---

## 2.7 FTP, SMTP, DNS and other protocol modules

The same approach applies:

```text
search type:auxiliary ftp
search type:auxiliary smtp
search type:auxiliary dns
search type:auxiliary ldap
```

Don't memorize individual modules.

Memorize:

```text
search type:auxiliary <protocol>
```

Then inspect the available modules.

---

## 2.8 Login scanners

Metasploit has login-scanner modules for various protocols.

Conceptually:

```text
auxiliary/scanner/<protocol>/<protocol>_login
```

Use them only when credential testing is explicitly authorized.

Before running one, determine:

```text
Allowed usernames?
Allowed password list?
Maximum attempts?
Lockout threshold?
Allowed time window?
```

This prevents a legitimate assessment from accidentally locking out production accounts.

---

## 2.9 Auxiliary `gather` modules

Search:

```text
search type:auxiliary gather
```

These are useful when you already have a target and need more information before choosing an exploitation path.

Think:

```text
scanner
   ↓
gather
   ↓
exploit
```

---

# 3. Exploit Modules — Turning Vulnerabilities Into Access

Exploit modules are fundamentally different from scanners.

Their purpose is to leverage a vulnerability to achieve code execution or another meaningful security impact.

Metasploit's documentation recommends using vulnerable test environments when learning exploit modules, including Metasploitable2/3.

---

## 3.1 Find an exploit

Search by product:

```text
search apache
```

Search by CVE:

```text
search cve:2021-44228
```

Search by platform:

```text
search type:exploit windows
```

Search by protocol:

```text
search type:exploit smb
```

---

## 3.2 Never select an exploit solely by name

Suppose you find:

```text
exploit/windows/smb/example
```

Immediately inspect:

```text
info
```

Look for:

```text
Name
Description
Disclosure Date
References
Targets
Platform
Arch
Privileged
Check
```

Then:

```text
show options
show targets
show payloads
```

The module's version requirements matter more than its name.

---

## 3.3 The `check` workflow

When available:

```text
check
```

Think:

```text
Is target vulnerable?
       ↓
Yes
       ↓
Is exploitation safe?
       ↓
Yes
       ↓
Select appropriate target
       ↓
Select payload
       ↓
Exploit
```

If `check` says:

```text
The target is not exploitable.
```

don't simply ignore it and fire the exploit anyway.

Investigate why.

---

## 3.4 Targets

Some exploits support multiple target configurations.

View them:

```text
show targets
```

You might see differences involving:

* Operating system
* Application version
* Architecture
* Memory layout
* Return addresses
* Target-specific behavior

Choosing the wrong target can result in:

```text
failed exploitation
application crash
service interruption
system instability
```

For production red-team engagements, target selection is therefore an operational decision, not just a technical one.

---

## 3.5 RHOSTS vs RHOST

You'll encounter:

```text
RHOST
```

and:

```text
RHOSTS
```

`RHOST` generally represents one remote host.

`RHOSTS` is designed for one or more targets and is commonly used by scanner modules.

Examples:

```text
set RHOST 10.10.10.20
```

or:

```text
set RHOSTS 10.10.10.20
```

For multiple hosts:

```text
set RHOSTS 10.10.10.20 10.10.10.21
```

or a CIDR range where supported and explicitly authorized.

---

## 3.6 Payload selection

An exploit answers:

> How do I trigger the vulnerability?

A payload answers:

> What should execute after successful exploitation?

Metasploit payload naming commonly follows:

```text
platform / architecture / stage / stager
```

For example:

```text
windows/x64/meterpreter/reverse_tcp
```

means:

```text
windows
x64
meterpreter
reverse_tcp
```

Metasploit's payload documentation explains staged and single payload structures in detail.

View compatible payloads:

```text
show payloads
```

Then choose one:

```text
set PAYLOAD <payload>
```

---

## 3.7 Reverse vs bind connections

### Reverse

```text
Target
  │
  └──────► Red-team listener
```

The compromised host initiates the connection back to the operator.

Useful when:

* Target can reach your listener
* Inbound connections to the target are restricted
* Network routing permits the callback

### Bind

```text
Red team ─────► Target listener
```

The target listens for your connection.

Useful when:

* Network routing permits inbound access
* You can reach the target directly
* A reverse path isn't practical

Always confirm the network path before choosing.

---

# 4. Payloads, Sessions & Meterpreter

## 4.1 Payload categories

List payloads:

```bash
msfvenom -l payloads
```

Or inside Metasploit:

```text
show payloads
```

Payloads can provide:

* Command shells
* Meterpreter sessions
* File transfer
* Network communication
* Application-specific execution

---

## 4.2 Meterpreter

Meterpreter is Metasploit's feature-rich session environment.

Once you have a legitimate session:

```text
sessions
```

Interact:

```text
sessions -i <ID>
```

Typical initial commands include:

```text
getuid
sysinfo
pwd
ls
```

The principle is:

> Gather only what is necessary to prove the engagement objective.

For example, if your objective is:

```text
Demonstrate code execution
```

you may only need:

```text
getuid
sysinfo
```

You don't need to collect unrelated personal or business data.

---

## 4.3 Backgrounding a session

From Meterpreter:

```text
background
```

List sessions:

```text
sessions
```

Return:

```text
sessions -i 1
```

Terminate a session when finished:

```text
sessions -K
```

Or terminate a specific session according to your engagement procedures.

---

## 4.4 Session types

You may encounter:

```text
meterpreter
shell
command shell
PowerShell
```

Use the least powerful session that satisfies the objective.

For example:

```text
Need proof of command execution?
        ↓
Shell may be sufficient.
```

If the engagement requires broader controlled post-exploitation:

```text
Meterpreter
```

may be appropriate.

---

## 4.5 Handlers

A handler waits for a payload callback.

Inside Metasploit:

```text
use exploit/multi/handler
set PAYLOAD <matching-payload>
set LHOST <listener>
set LPORT <port>
show options
run
```

The payload and handler must be compatible.

Conceptually:

```text
Generated payload
      │
      ▼
Target executes
      │
      ▼
Network callback
      │
      ▼
Matching handler
      │
      ▼
Session
```

---

## 4.6 Payload formats

`msfvenom` can produce different representations.

Check your installed version:

```bash
msfvenom --list formats
```

Payload generation:

```bash
msfvenom -p <payload> <options> -f <format> -o <file>
```

The exact format should be dictated by the approved delivery mechanism and target platform.

Do not treat encoding as a generic AV-bypass solution. Modern security controls can detect payload behavior independently of simple signatures.

---

# 5. Post Modules — Controlled Post-Exploitation

Post modules operate **after you already have a session**.

Metasploit's architecture separates post modules from exploits and auxiliary modules; post modules are intended for actions such as gathering information or performing post-compromise tasks.

Think:

```text
Initial access
     ↓
Session
     ↓
Post module
     ↓
Evidence
```

---

## 5.1 Find post modules

```text
search type:post
```

Narrow by platform:

```text
search type:post windows
search type:post linux
```

Or category:

```text
search type:post gather
```

---

## 5.2 Information gathering

Post modules can help establish:

```text
Operating system
Hostname
Users
Network configuration
Installed software
Security configuration
```

For an engagement, establish the minimum evidence necessary.

Example objective:

> Demonstrate that compromise of workstation A exposes the host's identity and network configuration.

You don't need to dump every available artifact.

---

## 5.3 Privilege escalation

Privilege escalation modules can be searched using:

```text
search type:post privilege
```

Before using one, establish:

```text
Current user
Current privileges
OS version
Architecture
Patch level
```

Then identify applicable escalation paths.

A good workflow:

```text
Current privilege
      ↓
System information
      ↓
Candidate escalation
      ↓
Check applicability
      ↓
Controlled exploitation
      ↓
Verify privilege
```

Don't run every privilege-escalation module blindly.

Some techniques can crash systems or alter system state.

---

## 5.4 Credential-related post modules

Metasploit contains modules capable of collecting credential material.

These are particularly sensitive in real engagements.

Before using them, define:

```text
What credentials are in scope?
Which hosts?
What evidence is required?
Where can the data be stored?
When must it be destroyed?
```

Prefer demonstrating:

```text
"Credential material is accessible"
```

over collecting a large production credential database when that isn't required.

---

## 5.5 Pivoting

One of Metasploit's most important red-team capabilities is using a compromised host as a path toward another authorized network.

Conceptually:

```text
Red Team
   │
   ▼
Compromised Host A
   │
   ▼
Internal Network
   │
   ├── Host B
   ├── Host C
   └── Server D
```

Before pivoting, verify that the internal targets are explicitly included in scope.

Typical concepts include:

```text
routes
port forwarding
SOCKS proxying
pivot-aware scanning
```

The key operational rule:

> A compromised host does not automatically make every reachable host in scope.

---

## 5.6 Loot and evidence

Metasploit can collect information from sessions.

Keep evidence organized:

```text
engagement/
├── evidence/
├── screenshots/
├── session-notes/
├── timestamps/
└── cleanup/
```

Record:

```text
Target
Time
Module
Result
Session ID
Evidence
Impact
```

This makes the final report defensible.

---

# 6. Advanced Operations, Module Selection & Professional Workflow

## 6.1 Encoders

List:

```text
show encoders
```

or:

```bash
msfvenom -l encoders
```

Encoders are primarily useful when a payload has representation constraints such as bad characters.

Do not confuse:

```text
encoding
```

with:

```text
reliable AV/EDR evasion
```

Metasploit's own documentation describes encoders in relation to payload transformation and bad-character handling.

---

## 6.2 NOP modules

NOP modules are primarily relevant to exploit development and shellcode layout.

List:

```text
show nops
```

or:

```bash
msfvenom -l nops
```

You generally won't need NOP modules during ordinary vulnerability validation.

Use them when:

```text
Exploit development
Shellcode layout
Architecture-specific padding
```

is actually part of the task.

---

## 6.3 Evasion modules

Metasploit has an `evasion` module family. The framework documentation identifies these modules as being intended to help avoid antivirus detection.

For professional engagements, treat this as a **separate authorization boundary**.

If the objective is:

```text
Does the EDR detect a standard Metasploit payload?
```

don't immediately start modifying it to evade detection.

Instead:

```text
Standard payload
      ↓
EDR detection
      ↓
Telemetry
      ↓
SOC response
```

That gives you a meaningful defensive result.

If the client explicitly authorizes adversary-emulation/evasion testing, define the permitted techniques and success criteria before using them.

---

## 6.4 Databases and workspaces

Metasploit can maintain assessment data using its database/workspace capabilities.

Typical commands:

```text
db_status
workspace
workspace -a engagement
workspace engagement
```

Then your discovered data can be associated with the correct assessment workspace.

This is extremely useful when managing multiple targets.

Conceptually:

```text
Client A
   └── workspace A

Client B
   └── workspace B

Lab
   └── workspace lab
```

Don't mix client data between workspaces.

---

## 6.5 Resource scripts

If you repeatedly execute the same authorized workflow, Metasploit supports resource scripts.

Example:

```text
use auxiliary/scanner/http/title
set RHOSTS 10.10.10.0/24
run
```

can be saved into a resource file and executed with:

```text
resource scan.rc
```

Use resource scripts for:

* Repeatable scans
* Lab exercises
* Standardized assessment workflows
* Consistent evidence collection

Avoid automating destructive or high-impact operations simply because automation makes them easy to repeat.

---

## 6.6 Searching efficiently

Instead of memorizing modules, learn search patterns.

### Everything related to SMB

```text
search smb
```

### Auxiliary SMB

```text
search type:auxiliary smb
```

### SMB exploits

```text
search type:exploit smb
```

### Windows post modules

```text
search type:post windows
```

### CVE

```text
search cve:<CVE>
```

### Exact product

```text
search name:<keyword>
```

### Platform

```text
search platform:windows
```

This is much more scalable than memorizing module names.

---

## 6.7 Module documentation

For any unfamiliar module:

```text
use <module>
info
info -d
show options
show advanced
show targets
show payloads
```

`info -d` is especially useful because Metasploit can generate module-specific documentation from the framework's documentation database.

Think of the module itself as the source of truth.

---

## 6.8 Choosing the right module

Use this decision tree:

```text
What do I know?
       │
       ▼
Need information?
       │
       ├── Yes → auxiliary
       │
       ▼
Need to validate vulnerability?
       │
       ├── Yes → auxiliary/check
       │
       ▼
Need code execution?
       │
       ├── Yes → exploit
       │
       ▼
Need a session?
       │
       └── Choose compatible payload
       
After access:
       │
       ▼
Need authorized post-exploitation?
       │
       └── post
```

---

## 6.9 A complete authorized engagement workflow

### Phase 1 — Scope

Document:

```text
Targets
Excluded targets
Allowed techniques
Credentials
Testing window
DoS restrictions
Social-engineering restrictions
Data-handling rules
Cleanup requirements
```

### Phase 2 — Discovery

Use:

```text
auxiliary/scanner/*
```

to establish:

```text
Hosts
Ports
Services
Versions
```

### Phase 3 — Enumeration

Use appropriate protocol-specific auxiliary modules.

Build:

```text
Host
 ├── Service
 │    ├── Version
 │    └── Configuration
 └── Application
      └── Version
```

### Phase 4 — Vulnerability mapping

Search:

```text
search cve:<CVE>
search <product>
```

Compare:

```text
Observed version
       ↓
Affected version range
       ↓
Applicable module
```

### Phase 5 — Validation

Where supported:

```text
check
```

Then assess:

```text
Is exploitation necessary?
Is it safe?
Is it within scope?
What evidence do we need?
```

### Phase 6 — Exploitation

Choose:

```text
Exploit
   +
Target
   +
Payload
```

Then verify the result.

### Phase 7 — Post-exploitation

Only perform actions needed to establish impact.

Examples:

```text
Identity
Privilege
Host information
Network position
Access to approved resource
```

### Phase 8 — Pivoting

Only pivot to explicitly authorized networks/hosts.

### Phase 9 — Evidence

Record:

```text
Timestamp
Target
Module
Configuration
Result
Screenshot/log
Impact
```

### Phase 10 — Cleanup

Terminate sessions and remove test artifacts according to the engagement plan.

---

# Practical Command Reference

## Start

```bash
msfconsole
```

## Search

```text
search <keyword>
search type:auxiliary <keyword>
search type:exploit <keyword>
search type:post <keyword>
search cve:<CVE>
```

## Inspect

```text
info
info -d
show options
show advanced
show targets
show payloads
```

## Configure

```text
set RHOSTS <target>
set RPORT <port>
set LHOST <listener>
set LPORT <port>
set PAYLOAD <payload>
```

## Execute

```text
check
run
exploit
```

## Sessions

```text
sessions
sessions -i <ID>
background
```

## Workspaces

```text
workspace
workspace -a <name>
workspace <name>
```

## Payloads

```bash
msfvenom -l payloads
msfvenom --list formats
msfvenom -l encoders
msfvenom -l nops
```

---

# The Metasploit Mental Model

If you remember only one thing, remember this:

```text
             ┌──────────────┐
             │   TARGET     │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │  AUXILIARY   │
             │ scan/gather  │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │ ENUMERATION  │
             │ version/info │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │    CHECK     │
             │ vulnerability│
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │    EXPLOIT   │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │   PAYLOAD    │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │   SESSION    │
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │     POST     │
             │ gather/verify│
             └──────┬───────┘
                    │
                    ▼
             ┌──────────────┐
             │   EVIDENCE   │
             │   CLEANUP    │
             └──────────────┘
```

The professional skill isn't memorizing thousands of Metasploit modules.

It's being able to answer:

**What am I trying to prove? → What information do I need? → Which module provides it? → Is the target actually vulnerable? → What's the least invasive way to demonstrate impact? → What evidence proves the result?**

That approach scales across Metasploit releases and is much more useful than memorizing individual exploit commands.
