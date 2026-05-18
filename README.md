# Red Team Runbook

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Framework](https://img.shields.io/badge/framework-MITRE%20ATT%26CK-red.svg)
![Status](https://img.shields.io/badge/status-active-success.svg)
![PRs](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)

A practitioner-focused runbook for planning and executing red team engagements. Structured around **MITRE ATT&CK**, with notes on regulated frameworks (TIBER-EU, CBEST, iCAST, AASE), modern C2, AD/Azure AD tradecraft, EDR evasion, and assumed-breach / purple operations.

For the phase-by-phase operator playbook, see [`Runbook.md`](Runbook.md).

> Use only against systems you are authorized to test. The techniques referenced here are illegal without explicit written authorization.

---

## Table of Contents

1. [Engagement Models](#engagement-models)
2. [Regulated Frameworks](#regulated-frameworks)
3. [Pre-Engagement](#pre-engagement)
   - [Scoping & RoE](#scoping--roe)
   - [Legal](#legal)
   - [Threat Modeling](#threat-modeling)
4. [Infrastructure](#infrastructure)
5. [Reconnaissance (TA0043)](#reconnaissance-ta0043)
6. [Resource Development (TA0042)](#resource-development-ta0042)
   - [Payload Development](#payload-development)
   - [Delivery Containers](#delivery-containers)
7. [Initial Access (TA0001)](#initial-access-ta0001)
8. [Execution & Defense Evasion (TA0002 / TA0005)](#execution--defense-evasion-ta0002--ta0005)
9. [Command & Control (TA0011)](#command--control-ta0011)
10. [Active Directory Tradecraft](#active-directory-tradecraft)
    - [Enumeration](#enumeration)
    - [Credential Access (TA0006)](#credential-access-ta0006)
    - [Privilege Escalation (TA0004)](#privilege-escalation-ta0004)
    - [Lateral Movement (TA0008)](#lateral-movement-ta0008)
    - [AD CS Abuse (ESC1–ESC15)](#ad-cs-abuse-esc1esc15)
    - [Trust & Forest Attacks](#trust--forest-attacks)
11. [Cloud / Entra ID](#cloud--entra-id)
12. [Persistence (TA0003)](#persistence-ta0003)
13. [Collection & Exfiltration (TA0009 / TA0010)](#collection--exfiltration-ta0009--ta0010)
14. [Impact (TA0040)](#impact-ta0040)
15. [Operational Security](#operational-security)
16. [Purple Teaming & Detection Engineering](#purple-teaming--detection-engineering)
17. [Reporting](#reporting)
18. [Post-Engagement](#post-engagement)
19. [Contributing](#contributing)
20. [License](#license)

---

## Engagement Models

- **Full-scope red team** — black-box, objective-driven, undeclared to defenders. Highest realism, highest cost, longest timeline (typically 4–12 weeks).
- **Assumed-breach** — operator placed at an internal foothold (workstation, low-priv account). Skips initial access; tests internal detection and response. Most common engagement type in 2026.
- **Purple team** — declared, collaborative. Run ATT&CK techniques alongside the SOC to measure and improve detection.
- **Adversary emulation** — replicate a specific threat actor's TTPs ([CTID emulation plans](https://github.com/center-for-threat-informed-defense/adversary_emulation_library)) — APT29, FIN7, etc.
- **Scenario-driven** — narrow objective: "compromise the trading platform," "exfiltrate crown-jewel data," "demonstrate ransomware blast radius without detonation."
- **TIBER / CBEST / iCAST** — see [Regulated Frameworks](#regulated-frameworks).

Pick the model with the customer before writing anything else.

---

## Regulated Frameworks

- **TIBER-EU** — Threat Intelligence-Based Ethical Red Teaming. EU financial services, mandated under DORA in many member states. Two-phase: TI provider produces a Targeted Threat Intelligence Report, then Red Team provider executes against scenarios derived from it.
- **CBEST** — UK FCA / Bank of England framework. Similar TI-led model. [Bank of England guidance](https://www.bankofengland.co.uk/financial-stability/financial-sector-continuity).
- **iCAST** — Hong Kong Monetary Authority's intelligence-led testing framework.
- **AASE / FEER** — Saudi (SAMA) Financial Entities Ethical Red-Teaming.
- **CORIE** — Australia (Council of Financial Regulators).
- **GBEST** — UK Government variant.

Common pattern: TI provider builds threat scenarios from real adversaries known to target the sector; red team provider executes them; both are typically separate firms; engagement is overseen by a regulator-blessed Control Group inside the target org. **The SOC is not informed.**

---

## Pre-Engagement

### Scoping & RoE

- Objectives (specific, measurable) — e.g. "obtain domain-admin equivalent in prod AD forest"
- In-scope assets — IP ranges, domains, cloud tenants, applications, personnel categories for SE
- Out-of-scope — explicit. Get critical systems (medical, OT, safety) listed.
- Permitted TTPs — phishing? Vishing? Physical? Payload detonation? Lateral movement to sensitive segments?
- **Hard stops** — actions that require pause and Control Group approval (DA, exfil of real data, anything touching prod payment systems, OT, healthcare)
- Engagement window, working hours, deconfliction process
- Control Group composition and contact (24/7 number)
- Get-out-of-jail letter — signed, dated, naming operators, contactable verifier
- Communications cadence (weekly sync, daily standup, immediate-call triggers)

### Legal

- Authorization scope tied to RoE in writing, signed by an authorized officer
- Country-specific statutes: **CFAA** (US), **Computer Misuse Act 1990** (UK), **§202a-c StGB** (Germany), **art. 323-1** (France), **Cybercrime Act 2001** (Australia), **PIPEDA** (Canada)
- Data protection — **GDPR**, **UK GDPR**, **CCPA/CPRA**, **PIPEDA**, **APPI** (Japan). PII handling rules apply to anything you exfil, including PoC screenshots.
- **DORA** (EU financial) — operational resilience testing requirements
- Third-party assets — get authorization from the actual asset owner, not just the engagement sponsor (SaaS, ISP-managed, cloud-shared infra)
- Multi-jurisdiction engagements — operator location, infrastructure location, target location each create legal exposure
- Evidence handling — chain of custody for anything you collect

### Threat Modeling

- What does the customer actually fear? (Map to scenarios, not generic "ransomware")
- Who realistically targets this org? (Sector + geography + value)
- TI inputs: [MITRE ATT&CK groups](https://attack.mitre.org/groups/), [Mandiant APT reports](https://www.mandiant.com/resources), [CrowdStrike threat reports](https://www.crowdstrike.com/resources/), [Recorded Future](https://www.recordedfuture.com/research)
- Build engagement scenarios from this, not from your favorite techniques

---

## Infrastructure

- **Operator boxes** — dedicated, ephemeral, no personal accounts logged in, full disk encryption
- **Team server** — locked down, behind redirector, never directly exposed
- **Redirectors** — HTTP(S), DNS, SMTP as needed. [Apache mod_rewrite](https://github.com/threatexpress/cs2modrewrite), Nginx, or socat. CDN fronting still works on some providers (Fastly, Cloudflare Workers patterns) though Azure/Cloudfront fronting is dead.
- **Categorized domains** — aged, business-themed; check categorization on Cisco Talos, Forcepoint, Bluecoat / Symantec, McAfee, Palo Alto, Zscaler. [Chameleon](https://github.com/mdsecactivebreach/Chameleon) for staging.
- **Domain selection** — typosquats, expired domains with prior categorization, [ExpiredDomains.net](https://www.expireddomains.net/)
- **TLS** — Let's Encrypt is fine but watch fingerprinting (issuer, cert age, SCT logs)
- **Mail infrastructure** — SPF, DKIM, DMARC must be set up correctly on sender side or phishing lands in spam. Warm domains before use.
- **Operator anonymity** — payment trail, registration trail, hosting trail should not connect to the firm
- **Tooling**:
  - C2 frameworks — [Cobalt Strike](https://www.cobaltstrike.com/), [Sliver](https://github.com/BishopFox/sliver), [Mythic](https://github.com/its-a-feature/Mythic), [Havoc](https://github.com/HavocFramework/Havoc), [Brute Ratel](https://bruteratel.com/), [Nighthawk](https://www.mdsec.co.uk/nighthawk/)
  - Phishing — [GoPhish](https://github.com/gophish/gophish), [Evilginx3](https://github.com/kgretzky/evilginx2), [Modlishka](https://github.com/drk1wi/Modlishka), [Muraena](https://github.com/muraenateam/muraena)
  - Ops management — [Ghostwriter](https://github.com/GhostManager/Ghostwriter) for tracking, [BloodHound CE](https://github.com/SpecterOps/BloodHound) for graphs

---

## Reconnaissance (TA0043)

- **Org footprint** — domains, subsidiaries, M&A history, public filings
- **People** — LinkedIn enum ([linkedin2username](https://github.com/initstring/linkedin2username)), [Hunter.io](https://hunter.io/), [SignalHire](https://www.signalhire.com/), conference talks, GitHub commits
- **Email format** — [ROCKYOU](https://github.com/initstring/email-format-search), validation via [o365creeper](https://github.com/Raikia/UhOh365), [office365userenum](https://github.com/gremwell/o365enum)
- **Breach data** — [Dehashed](https://www.dehashed.com/), [IntelX](https://intelx.io/), [Snusbase](https://snusbase.com/), HIBP enterprise
- **External attack surface** — Shodan, Censys, FOFA, ZoomEye, [Amass](https://github.com/OWASP/Amass), [Subfinder](https://github.com/projectdiscovery/subfinder)
- **Tech fingerprinting** — VPN appliances (Ivanti, Fortinet, Palo, Citrix — the perennial entry points), Exchange/OWA, Citrix, RDWeb, SharePoint, GitLab, Jira
- **M365 tenant enum** — [AADInternals](https://github.com/Gerenios/AADInternals), [o365recon](https://github.com/nyxgeek/o365recon), [Microsoft365_devicePhish](https://github.com/optiv/Microsoft365_devicePhish)
- **Cloud footprint** — [cloud_enum](https://github.com/initstring/cloud_enum), [TruffleHog](https://github.com/trufflesecurity/trufflehog) on the org's GitHub/GitLab
- **Code repos** — search org name, employee handles, leaked creds (TruffleHog, [gitleaks](https://github.com/gitleaks/gitleaks))
- **Physical** — building photos, badge designs, vendor signage, dumpster review (where in scope and legal)

---

## Resource Development (TA0042)

### Payload Development

- Initial-access loader vs. post-ex Beacon are different problems. Keep them separate.
- **Loaders** — staged shellcode loaders ([Donut](https://github.com/TheWover/donut), [PIC-Get-Privileges](https://github.com/EncodeGroup/PIC-Get-Privileges)), reflective DLLs, .NET assemblies via [DInvoke](https://github.com/TheWover/DInvoke)
- **Evasion building blocks**:
  - AMSI bypass (patching `AmsiScanBuffer`, hardware breakpoints)
  - ETW bypass (patching `EtwEventWrite`)
  - Direct/indirect syscalls — [SysWhispers3](https://github.com/klezVirus/SysWhispers3), [Hell's Hall](https://github.com/Maldev-Academy/HellHall)
  - Sleep masking — [Ekko](https://github.com/Cracked5pider/Ekko), [Foliage](https://github.com/SecIdiot/foliage), [DeathSleep](https://github.com/Cracked5pider/DeathSleep)
  - String / API hashing
  - Module stomping, thread stack spoofing, call stack masking
  - PE unhooking ([Perun's Fart](https://github.com/zeze-zeze/PerunsFart))
- **Beacon profiles** — Malleable C2 profile tuning; never ship the default
- **BOFs (Beacon Object Files)** — [TrustedSec BOFs](https://github.com/trustedsec/CS-Situational-Awareness-BOF), [outflanknl/C2-Tool-Collection](https://github.com/outflanknl/C2-Tool-Collection)
- **Sign your binaries** — code-signing cert, even self-issued or stolen-looking, helps with some EDRs and user trust
- **Testing** — VirusTotal is *not* your test bed. Run in a private lab with the customer's EDR if you can get it, or representative EDRs ([antiscan.me](https://antiscan.me/), [kleenscan](https://kleenscan.com/) — non-distributing scanners)

### Delivery Containers

- **HTML smuggling** — embedded blob, dropped on click ([HTMLSmuggler](https://github.com/sayadkarim/HTML-Smuggling), [SharpHTMLSmuggling](https://github.com/Mr-Un1k0d3r/SharpHTMLSmuggling))
- **ISO / IMG / VHD** — mount-on-double-click, bypasses Mark-of-the-Web (MoTW) propagation (Microsoft has tightened this — check current behavior on target Win build)
- **LNK** — argument abuse, icon spoofing
- **MSI / MSIX** — signed installer abuse
- **OneNote** — declined since Microsoft locked embedded files, but still relevant
- **PDF + JS** — limited surface
- **macro-enabled Office** — dead-ish on default modern tenants but enterprises still allow it via policy. Check before assuming.
- **ClickOnce** — `.application` files
- **XLL / WLL / PPA** — add-in formats

---

## Initial Access (TA0001)

- **Spear-phish with link** — credentials, MFA bypass (Evilginx3), device code
- **Spear-phish with attachment** — see Delivery Containers
- **OAuth illicit consent** — register attacker app, request high-priv scopes, victim clicks consent
- **Device code phishing** — abuse OAuth device code flow on M365 ([TokenTactics](https://github.com/rvrsh3ll/TokenTactics), [Microsoft365_devicePhish](https://github.com/optiv/Microsoft365_devicePhish))
- **Password spray** — `Spring2026!` and `<Companyname>2026` are still the answer surprisingly often. Tools: [TREVORspray](https://github.com/blacklanternsecurity/TREVORspray), [MSOLSpray](https://github.com/dafthack/MSOLSpray), [DomainPasswordSpray](https://github.com/dafthack/DomainPasswordSpray)
- **Adversary-in-the-middle (AiTM)** — Evilginx3 + Cloudflare worker for MFA-bypass against M365 / Okta / generic SSO
- **Public-facing app exploit** — Citrix, Fortinet, Ivanti, Palo, Exchange ProxyShell/ProxyNotShell, Confluence, Jenkins — the 2024–2026 era has been brutal here
- **Supply chain** — JS package, browser extension, CI/CD pipeline injection (in scope only)
- **Physical / drop** — Rubber Ducky, O.MG cable, malicious USB, network implant ([LANTurtle](https://shop.hak5.org/products/lan-turtle), [Plunder Bug](https://shop.hak5.org/products/plunder-bug-lan-tap))
- **Vishing / SE callback** — Microsoft IT impersonation → Quick Assist or AnyDesk install is the dominant 2025–2026 pattern

---

## Execution & Defense Evasion (TA0002 / TA0005)

- LOLBin execution — [LOLBAS Project](https://lolbas-project.github.io/), [GTFOBins](https://gtfobins.github.io/) for Linux
- Living-off-the-land — `rundll32`, `regsvr32`, `mshta`, `installutil`, `msbuild`, `cmstp` — most are flagged hard now; pick recent additions
- AppLocker / WDAC bypass — signed binary abuse, DLL hijack of allowed apps
- AMSI / ETW bypass per-process
- Parent-process spoofing
- Token impersonation / manipulation
- Process injection — moved well past CreateRemoteThread:
  - Process hollowing
  - Process Ghosting / Doppelgänging / Herpaderping
  - Mockingjay
  - Module stomping
  - APC injection (early-bird)
- EDR evasion BOFs — Sysmon view, ETW provider enum, [bofhound](https://github.com/fortalice/bofhound) for offline BloodHound from BOF output

---

## Command & Control (TA0011)

- **Channels** — HTTPS (most common), DNS (slow but covert), SMB (peer-to-peer internal), ICMP, custom
- **Domain fronting** — largely dead on major CDNs since 2018–2022; check current state per provider; Cloudflare Workers patterns still work in some configs
- **Beacon timing** — long sleep + jitter, not default 60s
- **Egress testing** — test C2 channels from inside the target environment ASAP after foothold; some channels die at the proxy
- **Frameworks** — Cobalt Strike (de facto), Sliver, Mythic, Havoc, Brute Ratel, Nighthawk. Pick based on engagement realism + budget + detection profile.
- **Profile tuning** — every default CS profile is signatured; modify `http-config`, `process-inject`, `stage`, `post-ex`
- **Peer-to-peer** — SMB / TCP Beacons for internal pivots, only one egress-talking node
- **Redirector chains** — at least one HTTP redirector in front of every team server

---

## Active Directory Tradecraft

AD is the engagement target for most enterprise red teams. Treat this section as the core, not an appendix.

### Enumeration

- **BloodHound CE** — [SpecterOps/BloodHound](https://github.com/SpecterOps/BloodHound) (community edition is the current line; legacy BloodHound is EOL)
- **SharpHound** / **AzureHound** collectors
- **ADExplorer** snapshot → BloodHound ingest via [ADExplorerSnapshot.py](https://github.com/c3c/ADExplorerSnapshot.py)
- **PowerView** / [PowerSploit](https://github.com/PowerShellMafia/PowerSploit) (heavily signatured, run from memory)
- **SOAPHound** — LDAP enum via SOAP for AMSI evasion ([SOAPHound](https://github.com/FalconForceTeam/SOAPHound))
- **LDAPDomainDump** ([dirkjanm/ldapdomaindump](https://github.com/dirkjanm/ldapdomaindump))
- Manual `net`, `dsquery`, ADSI queries when tools are caught

### Credential Access (TA0006)

- **LSASS dump** — direct via Mimikatz / [nanodump](https://github.com/fortra/nanodump) / [SafetyKatz](https://github.com/GhostPack/SafetyKatz); MiniDumpWriteDump with PPL bypass; comsvcs.dll MiniDump
- **DPAPI** — browser creds, RDP saved creds, vaulted secrets ([SharpDPAPI](https://github.com/GhostPack/SharpDPAPI))
- **NTDS.dit** — via shadow copy, [DSInternals](https://github.com/MichaelGrafnetter/DSInternals), [secretsdump](https://github.com/fortra/impacket)
- **Kerberoasting** — request TGS for SPN-enabled accounts, crack offline ([Rubeus](https://github.com/GhostPack/Rubeus) `kerberoast`, [hashcat](https://hashcat.net/) `-m 13100`)
- **AS-REP roasting** — accounts with `DONT_REQ_PREAUTH` set ([Rubeus](https://github.com/GhostPack/Rubeus) `asreproast`, hashcat `-m 18200`)
- **Coerced authentication** — [PetitPotam](https://github.com/topotam/PetitPotam), PrinterBug, [DFSCoerce](https://github.com/Wh04m1001/DFSCoerce), [Coercer](https://github.com/p0dalirius/Coercer)
- **NTLM relay** — [ntlmrelayx](https://github.com/fortra/impacket) → LDAP, ADCS HTTP enrollment, SMB
- **Shadow Credentials** — write `msDS-KeyCredentialLink` ([Whisker](https://github.com/eladshamir/Whisker), [pyWhisker](https://github.com/ShutdownRepo/pywhisker))
- **Cleartext / passwords on disk** — GPP `cpassword`, scripts in SYSVOL, Unattend.xml, web.config, IIS apppool, [SnaffPoint](https://github.com/nheiniger/SnaffPoint), [Snaffler](https://github.com/SnaffCon/Snaffler)
- **LAPS** — read `ms-MCS-AdmPwd` or new `msLAPS-Password` if delegated; check ACLs

### Privilege Escalation (TA0004)

- Local — [PrivescCheck](https://github.com/itm4n/PrivescCheck), [WinPEAS](https://github.com/peass-ng/PEASS-ng), [Seatbelt](https://github.com/GhostPack/Seatbelt)
- Token abuse — `SeImpersonate` → [PrintSpoofer](https://github.com/itm4n/PrintSpoofer) / [JuicyPotatoNG](https://github.com/antonioCoco/JuicyPotatoNG) / [GodPotato](https://github.com/BeichenDream/GodPotato)
- Unquoted service paths, weak service ACLs, registry autoruns
- UAC bypass — [UACME](https://github.com/hfiref0x/UACME); often via signed binary auto-elevation
- AD path PrivEsc — BloodHound shortest path, focus on `GenericAll`, `WriteOwner`, `WriteDACL`, RBCD setup primitives
- **Resource-Based Constrained Delegation (RBCD)** — abuse `msDS-AllowedToActOnBehalfOfOtherIdentity` ([Rubeus s4u](https://github.com/GhostPack/Rubeus))
- **Unconstrained delegation** — coerce DC → ticket capture

### Lateral Movement (TA0008)

- **Pass-the-Hash** — Mimikatz `sekurlsa::pth`, [Impacket](https://github.com/fortra/impacket) `psexec`/`wmiexec`/`smbexec`/`atexec`
- **Pass-the-Ticket** — Rubeus `ptt`, ticket import
- **Pass-the-Cert** — [PassTheCert](https://github.com/AlmondOffSec/PassTheCert), [Certipy](https://github.com/ly4k/Certipy) `auth`
- **Overpass-the-Hash** — NTLM → Kerberos TGT
- **WMI / WinRM / DCOM** lateral — Invoke-DCOM, [SharpWMI](https://github.com/GhostPack/SharpWMI)
- **RDP hijacking** — `tscon` from SYSTEM
- **SCCM abuse** — [SharpSCCM](https://github.com/Mayyhem/SharpSCCM); ConfigMgr → DA is increasingly common path
- **Group Policy abuse** — write to GPO with `WriteProperty` → SharpGPOAbuse

### AD CS Abuse (ESC1–ESC15)

[Certified Pre-Owned (SpecterOps)](https://posts.specterops.io/certified-pre-owned-d95910965cd2) — still the canonical reference; ESC categories continue to expand.

- **Enumeration** — [Certify](https://github.com/GhostPack/Certify), [Certipy](https://github.com/ly4k/Certipy) `find`
- **ESC1** — template allows SAN, low-priv can enroll → impersonate any user
- **ESC2** — Any Purpose EKU
- **ESC3** — Enrollment Agent template
- **ESC4** — vulnerable template ACL
- **ESC6** — `EDITF_ATTRIBUTESUBJECTALTNAME2` flag on CA
- **ESC7** — vulnerable CA ACL
- **ESC8** — NTLM relay to HTTP/RPC enrollment endpoint → cert for relayed account
- **ESC9 / ESC10** — `no-security-extension` / weak certificate mapping
- **ESC11** — relay to ICPR RPC
- **ESC13** — OID group link abuse
- **ESC14 / ESC15** — newer (2024–2025), weak certificate mapping & schema-based abuse paths — review current SpecterOps research before assuming applicability

### Trust & Forest Attacks

- **Golden Ticket** — krbtgt hash → forge any TGT
- **Silver Ticket** — service account hash → forged TGS for that service
- **Diamond Ticket** — modify a real TGT in-memory (more OPSEC-safe than Golden)
- **Sapphire Ticket** — Diamond + S4U2self
- **SID History injection** — across trusts (with caveats post-CVE-2020-0665)
- **Trust ticket forgery** — forge inter-realm TGT
- **Foreign security principals** — abuse cross-forest group memberships

---

## Cloud / Entra ID

- **Recon** — tenant ID enum (`login.microsoftonline.com/<domain>/.well-known/openid-configuration`), user enum (o365creeper, `office365userenum`)
- **Password spray** — [MSOLSpray](https://github.com/dafthack/MSOLSpray), [TREVORspray](https://github.com/blacklanternsecurity/TREVORspray), Conditional Access aware
- **Token theft / replay** — [TokenTactics](https://github.com/rvrsh3ll/TokenTactics), [TokenTacticsV2](https://github.com/f-bader/TokenTacticsV2), [ROADtools](https://github.com/dirkjanm/ROADtools), [AADInternals](https://github.com/Gerenios/AADInternals)
- **Device code phishing** — high success against MFA-protected accounts when CA doesn't block
- **Illicit consent grant** — register multi-tenant app, request `Mail.Read` / `Files.Read.All` / `offline_access`
- **Pass-the-PRT** — extract Primary Refresh Token from logged-in workstation, replay
- **Service Principal abuse** — overprivileged SPs are everywhere; map with ROADtools or [Stormspotter](https://github.com/Azure/Stormspotter)
- **Azure RBAC paths** — [AzureHound](https://github.com/SpecterOps/AzureHound) → BloodHound
- **Hybrid attacks** — on-prem AD → Azure via AD Connect (Seamless SSO, Pass-through Auth, sync account abuse) — [adconnectdump](https://github.com/fox-it/adconnectdump)
- **Conditional Access bypass** — non-interactive auth flows, legacy protocols (mostly killed), trusted IP exclusions, device-trust requirements
- **AWS** — [Pacu](https://github.com/RhinoSecurityLabs/pacu), [enumerate-iam](https://github.com/andresriancho/enumerate-iam), instance metadata abuse, IAM PrivEsc paths (see [Rhino Security IAM PrivEsc](https://rhinosecuritylabs.com/aws/aws-privilege-escalation-methods-mitigation/))
- **GCP** — [GCPBucketBrute](https://github.com/RhinoSecurityLabs/GCPBucketBrute), service account impersonation chains

---

## Persistence (TA0003)

- **AD** — AdminSDHolder ACL, DCSync rights to a low-priv account, `DnsAdmins` membership, Skeleton Key (loud), Golden Ticket (long-term but krbtgt-bound)
- **Host** — scheduled tasks, services, WMI event subscription, COM hijack, registry Run keys, Image File Execution Options, AppInit DLLs, Winlogon helper, screensaver
- **Cloud** — additional admin, service principal credentials, app role assignment, federated identity, OAuth refresh tokens
- **M365** — mailbox rules (auto-forward), delegated mailbox access, app permissions
- **Implants** — long-haul Beacon on separate infrastructure, ideally different family from primary
- **OPSEC** — persistence on a *separate* host from active operator activity; rotate primary and long-haul

---

## Collection & Exfiltration (TA0009 / TA0010)

- **Targeting** — Snaffler / SnaffPoint output; SharePoint; OneDrive; mail PSTs; code repos; KeePass `.kdbx`; browser profiles
- **Document grep** — keyword sweep for `password`, `secret`, `apikey`, `confidential`, customer names
- **Mail extraction** — [MailSniper](https://github.com/dafthack/MailSniper), Graph API queries, `New-MailboxExportRequest` (Exchange on-prem)
- **Archives** — encrypted 7z or AES container before exfil
- **Channels** — same C2 channel (slow, safe), separate cloud upload (faster, noisier), DNS exfil (`iodine`, `dnscat2`)
- **Volume control** — never exfil real PII without explicit RoE permission; use representative samples + manifest

---

## Impact (TA0040)

Real impact (encryption, destruction) is **out of scope** for nearly all engagements. Demonstrating it without doing it:

- Place a benign canary file in crown-jewel locations and show retrieval
- Demonstrate write/delete capability against test files, not production
- Document the exact blast radius (hosts, accounts, data classes accessible) without acting
- Show ability to disable EDR / backup / replication without actually disabling

---

## Operational Security

- **Logging discipline** — every command logged, timestamped, attributed; Ghostwriter or equivalent
- **Host hygiene** — clear browser cache, sessions, history on operator boxes between engagements; never reuse infrastructure across customers
- **Tradecraft separation** — initial access infra ≠ post-ex infra ≠ long-haul infra
- **Tool versioning** — track which build of each tool was used (some leave version strings in artifacts)
- **Artifact cleanup** — uploaded BOFs, dropped binaries, scheduled tasks, created accounts — list, then remove after engagement
- **Capture-the-flag discipline** — record file hashes of anything dropped, MAC addresses, IP allocations, account names, ticket lifetimes
- **Comms** — encrypted operator chat (Signal, Matrix), no engagement detail in personal accounts
- **Don't write the company name in plaintext** in C2 profiles, file paths, or screenshots that might leak

---

## Purple Teaming & Detection Engineering

- ATT&CK Navigator overlay — pre-engagement (what you'll run) and post-engagement (what was detected)
- For each technique executed, capture: timestamp, source host, command, expected telemetry, observed detection (yes/no/partial)
- Tools — [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) for declarative technique execution; [VECTR](https://vectr.io/) or [SCYTHE](https://www.scythe.io/) for tracking; [APTSimulator](https://github.com/NextronSystems/APTSimulator)
- Deliver detection opportunities back to blue with:
  - Sigma rules ([SigmaHQ](https://github.com/SigmaHQ/sigma)) per missed detection
  - Specific log sources required (Sysmon configs, EDR query language, cloud audit logs)
  - Test queries the SOC can run to validate fix

---

## Reporting

- **Executive summary** — one page, no jargon, business impact framed in customer's terms
- **Engagement narrative** — chronological attack path, screenshots, timestamps
- **Findings** — per technique:
  - ATT&CK technique ID
  - Description and impact
  - Reproduction steps
  - Affected assets
  - Detection guidance (Sigma rule, log source, sample query)
  - Remediation
  - Severity (CVSS optional, business-impact reasoning required)
- **Attack path diagram** — visual; BloodHound exports work well
- **IOC list** — hashes, domains, IPs the SOC can use to validate detection
- **Tooling** — [Ghostwriter](https://github.com/GhostManager/Ghostwriter) for tracking and reporting, [PwnDoc](https://github.com/pwndoc/pwndoc), [SysReptor](https://github.com/Syslifters/sysreptor), Obsidian/Notion for ops notes
- **Don't** ship raw scanner output. Don't ship a "list of vulns." Tell the story.

---

## Post-Engagement

- Debrief with Control Group
- Joint readout with blue / SOC (purple debrief)
- Deliver detection content (Sigma rules, queries, log source recommendations)
- Confirm artifact removal — every account, file, task, persistence mechanism
- Sign-off on infrastructure decommission
- Customer satisfaction signed off in writing
- Internal retro — what worked, what got caught, what to do differently
- Update internal TTP library with anything new learned

---

## Contributing

PRs welcome. Useful additions: new tradecraft with sources, framework updates, replacements for tools that have died, links to authoritative research. Keep entries tight.

---

## License

[MIT](LICENSE) — use freely, attribution appreciated.

---

This is a living document. Adversaries change; the runbook should too.
