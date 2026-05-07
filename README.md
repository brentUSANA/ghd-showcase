# GHD Automation Toolkit
### USANA Global Help Desk — AI-Assisted Operations
*Built by Brent Christensen | April 2026*

---

USANA's Global Help Desk supports hundreds of users across 20+ countries — United States, Canada, Mexico, Colombia, Europe, India, Australia, New Zealand, and Asia Pacific markets including Korea, Philippines, Taiwan, and Japan. There are help desk counterparts in Asia Pacific and Austria who cover phones outside of Salt Lake City hours, but the overwhelming majority of hands-on support, escalations, incident management, and tooling is handled by two people in Salt Lake City.

This toolkit integrates Claude AI directly into daily GHD operations. These are not generic AI features — every capability was built to solve a specific, real GHD problem.

The full scripts and Claude configuration live in a private repository. This page documents what was built and why.

---

## Table of Contents

1. [Jira Ticket Automation](#1-jira-ticket-automation)
2. [Active Directory Operations](#2-active-directory-operations)
3. [Intune & Jamf Device Lookups](#3-intune--jamf-device-lookups)
4. [Printer & Toner Workflow](#4-printer--toner-workflow)
5. [Incident Management](#5-incident-management--real-time-coordination)
6. [Email Investigation](#6-email-investigation)
7. [Living Knowledge Base](#7-living-knowledge-base)
8. [Token Efficiency & Cost Management](#8-token-efficiency--cost-management)
9. [Lansweeper Asset Management](#9-lansweeper-asset-management)
10. [Application Access Provisioning](#10-application-access-provisioning)
11. [SentinelOne Threat Response](#11-sentinelone-threat-response)
12. [Teams & Slack Communication](#12-teams--slack-communication-drafting)
13. [Live Ticket Queue Dashboard](#13-live-ticket-queue-dashboard)
14. [GHD Digests](#14-ghd-digests)
15. [Persistent Memory](#15-persistent-memory----learns-and-doesnt-forget)
16. [Open API Approvals](#open-api-approvals-in-progress)

---

## 1. Jira Ticket Automation

Claude creates, closes, and comments on GHD tickets in Jira from plain English — with zero copy-paste into a browser.

**What it does automatically:**
- Sets the correct Component (Printer, Software, VPN, AD, Cell Phone, etc.) based on keywords in the description
- Sets Sub-Component (Hardware, Software - Issues, Software - Install, etc.)
- Detects the Channel (Phone Call, Walk Up, Chat, Email, Ticket)
- Sets the correct Reporter to the affected user, falls back to Brent if the user isn't found in Jira
- Copies the ticket URL to the clipboard the moment it's created
- Closes the ticket immediately if resolved (FCR = Yes if applicable)
- Posts a separate resolution comment so the description stays clean

**Example interactions:**

> *"Carolyn Green called in — her Zoom local recordings weren't showing up. I switched her to the Local PC Recording tab in the Zoom app and it was resolved."*

→ Claude looks up Carolyn in Jira to set her as reporter, creates the ticket, sets Component: Software, Sub-Component: Software - Issues, Channel: Phone Call, FCR: Yes, closes it, and copies the URL to clipboard. One sentence from Brent.
Real ticket: [GHD-97384](https://usana.atlassian.net/browse/GHD-97384)

> *"Charlotte Mukayiranga walked up with her PC — can't log into Agile PLM."*

→ Claude looks up Charlotte, sets her as reporter, sets Component: Software, Channel: Walk Up, knows Agile uses its own credentials (not SSO), and walks through the fix sequence: username format first, email-suffix variant if that fails, admin reset as last resort. Cache clearing alone does not fix Agile login failures — learned from this ticket and stored permanently.
Real ticket: [GHD-97387](https://usana.atlassian.net/browse/GHD-97387)

> *"AD reset ticket for Kara Grieb — she can't change her password, says it rejects everything she tries."*

→ Claude looks up Kara in Active Directory: account not locked, `PasswordExpired` shows False, last set 89 days ago. Flags the near-expiry wall — a known state where the flag hasn't flipped yet but the user is already blocked. Recommends admin reset with must-change-at-next-logon rather than directing her to the self-service portal, which won't work in this state. Creates the ticket with account findings documented, FCR, closed.
Real ticket: [GHD-97378](https://usana.atlassian.net/browse/GHD-97378)

> *"Leave the ticket open — still waiting on user to test."*

→ Claude creates without FCR, transitions to Waiting for Customer via the proper workflow (New → In Progress → On Hold/To Customer).

> *"Katherine Bates called about fraud on an Amazon account tied to former employee Stefanie Myers' old USANA email. Stefanie needs the password reset email but her account was disabled in 2024. Her manager Josh Sube is also disabled — next up is Stephen Jones."*

→ Claude identified the approval chain required before any action can be taken (former employee email access plus fraud context equals manager approval plus legal or compliance sign-off), structured the ticket as pending rather than actionable, documented the proposed solution (convert the mailbox to shared so Katherine can receive the reset email on Stefanie's behalf), and flagged that no approvals have been obtained yet. The ticket is a clean record of the situation — not a task for GHD to act on independently.
Real ticket: [GHD-97680](https://usana.atlassian.net/browse/GHD-97680)

**Ticket description standards (trained):**
- Description = device + symptom + behavior. No "user reported..." narration.
- Resolution steps go in a separate comment, never the description.
- Location names written in full — no internal abbreviation codes.
- Hostname always included — Claude looks it up in Intune before writing the ticket.

**Reporter disambiguation:**
- If a userID is provided, Jira reporter is resolved by email (`userID@usanainc.com`) — not display name. Prevents substring matches like "Chris" hitting "Christian."
- If no userID is given and names could be ambiguous (Chris/Christian, Sara/Sarah, common surnames), Claude asks which person is intended before creating the ticket. Wrong-reporter notifications go to real people — a one-second confirmation beats an apology. Learned from a real incident (May 2026).

---

## 2. Active Directory Operations

Claude runs AD queries and performs operations against USANA's domain directly from the terminal using securely stored admin credentials (DPAPI-encrypted, machine-bound).

**What it can do:**
- Look up any user: account status, last logon, password age, locked status, group memberships
- Detect direct vs. nested group membership (catches what a basic `MemberOf` lookup misses)
- Unlock accounts
- Reset passwords with "must change at next logon"
- Spot near-expiry password issues before users hit a wall

**Known pattern Claude watches for:**

The "89-day wall" — a user's `PasswordExpired` flag stays `False` even though they can no longer change their own password. Claude flags this state and recommends an admin reset with must-change rather than directing the user to the self-service portal, because the self-service portal won't work in that state. Real ticket: [GHD-97378](https://usana.atlassian.net/browse/GHD-97378)

**PIM role activation — built, pending approval:**

Many AD, Exchange, and Azure operations require activating a Privileged Identity Management (PIM) role first. Previously this meant navigating the Azure portal manually — select role, set duration, enter justification, submit, wait.

A `/pim` command was built: a numbered menu in the terminal shows every available role, pick one, enter duration and justification, and a PowerShell script activates the role via the Microsoft Graph API. No browser.

The required admin consent (`RoleManagement.ReadWrite.Directory`) was declined by WinOps (WINOPS-19060). Portal activation is the current workflow. The command is built and ready if that decision is revisited.

**Proactive auth check — morning digest:**

The delegated Graph token used by `/pim` expires overnight. Rather than surfacing a device code prompt mid-interaction — while a user is standing at the desk waiting — Claude runs a silent auth check at the start of every morning digest. If the token is still valid, it moves on silently. If it has expired, it presents the device code URL immediately, before any ticket work begins. The re-auth is done once, at the start of the day, on your schedule — not someone else's.

This was added after a real walk-up incident on April 24 2026 where a BitLocker key request was delayed by an unexpected auth prompt during `/pim` activation.

---

## 3. Intune & Jamf Device Lookups

Claude looks up any device — Windows or Mac — by user's full name and returns hostname, device model, serial number, OS version, last check-in, and management status.

**Lookup order:**
1. **Intune first** — covers all Windows PCs. Runs silently via a service principal (no auth prompts). Returns hostname, model, serial, OS, last sync, and compliance state.
2. **Jamf if nothing found** — catches Mac-only users. Authenticates via OAuth client credentials, fetches the full device inventory, filters to the user, then pulls rich detail from the Classic API: friendly model name, chip, RAM, FileVault status, and enrolled date.

**How it works:**
- Both scripts live at a designated path on the Windows machine (Claude-managed, separate from user-deployed tools)
- Lookups run fully silently — no authentication prompts required for either system
- Results are cached in memory: once a hostname is learned, future mentions of the same user return it instantly without running any script. If the cached result fails or is stale, a fresh live lookup runs automatically.

Real ticket example: [GHD-97382](https://usana.atlassian.net/browse/GHD-97382) — BitLocker recovery key request. Hostname retrieved via Intune lookup and included in the ticket automatically.

**WINOPS-19012 — Approved & closed (April 2026):** Intune service principal granted. Token does not expire.

**WINOPS-19019 — Approved & closed (April 2026):** Jamf API client granted. Mac lookups are now fully automated — same experience as Windows.

---

## 4. Printer & Toner Workflow

Shipping calls every morning about printers. This workflow handles it in one sentence.

**Example interaction:**

> *"Machine ID NPI12345 needs toner."*

→ Claude looks up the Machine ID in the printer registry, identifies the printer model, cross-references the toner mapping spreadsheet to find the correct cartridge number, formats a Slack message in the exact format used for the stock closet, and sends it so Brent can read it on his phone while walking to the toner closet. Then creates the Jira ticket with the correct component and sub-component.

**Before:** Take the call → look up printer model manually → find toner number in a spreadsheet → write it down → walk to closet.

**After:** One sentence.

**Also trained:**
- Physical hardware issue → Sub-Component: Hardware - Printer
- Driver/config fix → Sub-Component: Software - Issues
- Hardware issued from GHD stock (adapters, accessories, peripherals) → Sub-Component: Hardware - New Hardware / Stock. Routing is keyword-aware — "USB-C adapter" and "from stock" resolve to Hardware before a networking keyword like "ethernet" can misfire.
- DHCP printer issues always get "Printer DHCP Reservation" in the title so the network team knows immediately what they're looking at
- Claude flags any printer IP outside the expected VLAN subnet as a missing DHCP reservation before the ticket is written

**BarTender label printing — Zebra printers:**

Shipping and warehouse staff print labels on Zebra barcode printers using BarTender software. When a Zebra printer stops producing labels, the root cause is almost always a licensing issue — BarTender's connection to the license server has dropped or rotated out.

Claude recognizes the trigger keywords — "bartender," "zebra printer," "barcode printer," "label printer," "won't print" — and treats it as a licensing issue first, every time:

> *"Shipping called — their Zebra barcode printer won't print, something about BarTender."*

→ Claude goes straight to the fix: open BarTender → Help → BarTender Licensing Wizard → add network license server → 10.35.10.149 (LVDC-BRTND01). No diagnostic tree, no guesswork. 30 licenses are available on that server — the issue is never a shortage, always a dropped connection.

---

## 5. Incident Management & Real-Time Coordination

The April 2026 Cycle 3 / APO Outage is the best example of what Claude can do when things go sideways at scale.

**The incident:**
Distributors across 20+ countries lost the ability to set their Auto-delivery Purchase Orders (APOs) for Cycle 3. Affected markets: Mexico, Korea, Philippines, Taiwan, Japan, and more. Root cause: a developer cycle filter code change that silently removed cycle dates from the customer-facing platform globally.

**The ticket chain Claude tracked end-to-end:**

```
GHD-97361  — Mexico region user reports missing cycle date
    ↓
PR-109481  — Problem Resolution opens, scope widens globally
    ↓
ITIL-2345  — Full incident declared, all teams paged
    ↓
DEV-95135  — Developer hotfix ticket opened
    ↓
APPSUP-53370 — PaaS hotfix release ticket (fixVersion: 2026.04.15.04)
    ↓
#hotfix-2026-04-15-04-...  (dedicated Slack hotfix channel)
```

**What Claude did during the incident:**
- Monitored three Slack threads simultaneously — the main incident channel, the dedicated hotfix channel, and a side thread from a separate team
- Filtered 200+ Slack messages down to ~15 clean ITIL timeline entries, skipping commentary, suggestions, and reactions
- Formatted every entry in the exact required format for the ITIL ticket
- Tracked the full hotfix approval chain in sequence (DevOps branch creation → PR approval → Bamboo stage build → QA validation → director approval → prod deploy → prod validation)
- Generated the full ITIL Resolution Summary at close: Issue / Root Cause / Resolution / Tickets Referenced / Follow-up — structured and ready to paste into Jira with zero editing

**Why this matters:**
During a P1 incident, Brent is simultaneously on the phone, on Slack, paging teams, creating tickets, and updating users. Claude took the Slack monitoring and timeline formatting entirely off his plate.

**ITIL ticket creation — now live:**

Claude can create ITIL incident tickets programmatically via the Jira REST API — the same way GHD tickets are created, but with the ITIL project's required fields handled automatically: Market selection, Technical Debt flag, priority, and ADF-formatted description.

> *"Jira is down — work items aren't loading. Wei Zhang reported it from China at 7:14pm and I confirmed it myself here at HQ."*

→ Claude discovers that the Jira outage is multi-site, not China-specific, writes a summary that doesn't over-scope to one region, resolves all required ITIL field IDs at runtime (Market, Technical Debt), creates the incident ticket with the correct priority, and copies the URL to clipboard.
Real ticket: [ITIL-2353](https://usana.atlassian.net/browse/ITIL-2353)

**Open ticket — watching:**
`WINOPS-19021` — Requesting a PagerDuty API key so Claude can trigger alerts directly from conversation instead of manually. ITIL ticket creation is already live. Once the PagerDuty key is approved: identify incident, page the right team, create the ITIL ticket, and post to the status channel — all in one command.

---

## 6. Email Investigation

**What it can do:**
- Trace missing or delayed emails via Exchange Online PowerShell
- Identify whether an email was lost in Exchange or stopped by the email security gateway before it ever reached Exchange
- Walk through the investigation pattern: narrow time window first, then widen, then broaden keywords, then escalate if zero results (zero results = email never hit Exchange = security gateway intercept)

**Example:**
A user reported never receiving an expected external email. Claude ran a full-day message trace with broad keywords, found zero results, concluded the email never reached Exchange, identified the email security gateway as the likely intercept point, and drafted the escalation note with the evidence already attached. Confirmed sender-side issue — email never hit USANA mail servers at all.
Real ticket: [GHD-97353](https://usana.atlassian.net/browse/GHD-97353)

---

## 7. Living Knowledge Base

Claude isn't working from generic IT knowledge. It has studied USANA-specific reference material and adds to it automatically every session.

**What was ingested and is actively used:**

- **Confluence GHD space** (scraped by page ID) — escalation routing, on-call phone tree, full step-by-step procedures available on demand
- **Printer translation spreadsheet** — every printer's Machine ID mapped to its physical model, every model mapped to its correct toner cartridge number
- **Stock closet inventory** — every hardware item GHD keeps on hand with current prices; surfaces automatically when a user mentions needing hardware
- **Application access matrix** — AD groups, ticket routing, login URLs, and credential quirks for 50+ applications
- **Network and infrastructure reference** — key services, escalation paths, share naming conventions, department-specific quirks

**How the knowledge keeps growing — automatically:**

A hook runs at the end of every Claude session. It reviews the conversation, identifies any new environment details that came up — new service names, AD group names, user quirks, ticket patterns, vendor contacts — and writes them into the appropriate reference file.

Real examples of auto-captured knowledge:
- A ticket revealed that a specific Isilon share requires a separate prerequisite group before the access group works. Stored. Claude catches this automatically on future access requests.
- A ticket revealed that Agile PLM cache clearing does NOT fix login failures. Stored as a rule. Claude won't recommend that path again.
- A network incident revealed that four Tableau hosts require outbound access to a specific OAuth broker endpoint. Stored. Claude knows this for future Tableau connectivity calls.

Brent doesn't maintain any of this manually. The knowledge base builds itself.

---

## 8. Token Efficiency & Cost Management

Running Claude against a 400+ line environment reference on every session adds up. These are the active measures that keep context lean without losing capability.

**On-call context injection:** A UserPromptSubmit hook watches for phrases like "going on call" and injects the full on-call escalation procedures — phone trees, PagerDuty routing, incident checklists — only for the duration of the on-call window. When the shift ends ("my on call shift is over"), the section unloads and auto-deactivation triggers at 6 AM MT Monday regardless. That section is roughly 9KB; loading it only when it's needed rather than every session is a measurable daily saving.

**Hook model tiering:** Background hooks run on Haiku — the session learning capture that writes new environment details into reference files, and the doc sync check that runs after edits. The main conversation runs on Sonnet. The session learning capture was previously running Sonnet after every single response; moving it to fire once at session close rather than continuously cut the per-session cost of that hook significantly.

**Plugin discipline:** Only GHD-relevant plugins are enabled. Non-IT-work plugins (frontend design tools, code simplifiers, etc.) are disabled to keep the skill listing context load small each session — every enabled plugin contributes to the prompt preamble whether or not it's ever invoked.

**Context trimming:** The access provisioning reference section was cut from 265 lines to 128 by removing redundant login URL callouts and routing notes already captured elsewhere. Nine separate Jira-related memory files were consolidated into two. The goal is one authoritative location per topic, not multiple files with overlapping content.

---

## 9. Lansweeper Asset Management

Claude looks up hardware assets in Lansweeper to find hostname-to-user mapping, purchase dates, warranty status, and device specs.

**Trained behavior:**
- Uses the correct username format for lookups (ignores legacy domain prefix entries)
- Knows that Jira and AD descriptions never contain hostnames — goes straight to Lansweeper or Intune rather than guessing

**WINOPS-19019 — Approved & closed (April 2026):** Jamf API client granted. Mac device lookups are now fully automated — same experience as Intune for Windows. See section 3.

---

## 10. Application Access Provisioning

Claude has the full application access matrix for USANA memorized — 50+ applications, including which AD groups to add, which teams to route tickets to, login credential formats, and any multi-step requirements.

**Example interactions:**

> *"New user needs Tableau access."*
→ Add the correct Azure AD groups, then route ticket to the BI team.

> *"User can't log into Agile PLM."*
→ Claude knows Agile uses its own credentials (not SSO), walks through the username format options, tries the email-suffix variant if the first fails, then recommends admin reset with must-change. Cache clearing alone does NOT fix Agile login failures — learned from a real ticket.
Real ticket: [GHD-97387](https://usana.atlassian.net/browse/GHD-97387)

> *"New hire needs Adobe Acrobat Pro."*
→ Add the license group, add the user to the SharePoint license tracker, email the Adobe admin. Claude knows all three steps and won't let you skip them.

> *"User needs access to the Studios file share."*
→ Claude knows this share requires a prerequisite "all users" group in addition to the specific access group — having Creative Services access does NOT grant Studios access. Completely separate shares.
Real ticket: [GHD-97086](https://usana.atlassian.net/browse/GHD-97086)

> *"Victor Esquivel needs Production Share Drive access for nine members of his Manufacturing team."*

→ Claude looks up each user in Active Directory, confirms the listed manager matches Victor before applying any groups, then adds the two required AD groups to all confirmed users and posts a comment to the ticket documenting every change.

Two users had a different manager listed in Active Directory — Claude flagged them and held before touching their accounts. That turned out to be the more significant outcome: those users are Victor's actual direct reports, but the Active Directory Manager field was wrong. HR confirmed the discrepancy via email and the org structure correction is now in progress. Access provisioned for all users. A data integrity issue surfaced and flagged as a byproduct of automated manager verification running on every user before any group change is made.
Real ticket: [GHD-97794](https://usana.atlassian.net/browse/GHD-97794) (resolved)

---

## 11. SentinelOne Threat Response

Claude knows the malware detection workflow end-to-end.

**What it can do:**
- Walk through the triage checklist (Action Status: Failed vs. Succeeded)
- Identify when to physically pull a machine vs. remote remediate
- Draft the desk note to leave if a machine is pulled without reaching the user
- Know that GHD can clear detections and reconnect devices, but exclusion rules require InfoSec

Real ticket: [GHD-97386](https://usana.atlassian.net/browse/GHD-97386) — SentinelOne false positive locked a Mac off the network. GHD cleared the detection and reconnected the device. Exclusion request forwarded to InfoSec and tracked as an open item.

---

## 12. Teams & Slack Communication Drafting

Claude drafts Teams and Slack messages with trained behavior:

- No em dashes or arrow characters — they render as garbage in Teams. Claude uses plain hyphens.
- Messages are personalized — pulls the specific ticket or issue detail, never uses generic placeholder phrasing.
- Multi-step instructions are offered to clipboard rather than pasted raw, because terminal copy-paste garbles formatting when a user pastes into Teams or a ticket.

---

## 13. Live Ticket Queue Dashboard

A custom-built visual ticket queue dashboard:

- Pulls live data from Jira REST API
- Shows all open GHD tickets assigned to the team
- Highlights low-hanging fruit (tickets likely closeable quickly)
- Launches via a batch file on Brent's desktop each morning
- Runs a local CORS proxy so the browser can talk to Jira securely from WSL

This was entirely designed and built by Claude based on Brent describing what he wanted to see each morning.

---

## 14. GHD Digests

Three structured digest outputs bracket and close the workday. Each is a single command — or a single phrase — and every output lands directly on the Windows clipboard automatically. No selecting, no copying. The moment Claude finishes generating a digest, it is ready to paste — into Teams, a ticket, an email, or a notes app — with no extra steps.

---

### Morning Digest

The first output of the day. One command pulls the full open ticket queue from Jira, runs live lookups on every action item, and copies a single categorized snapshot straight to clipboard.

Before any ticket work begins, Claude runs a silent Microsoft Graph auth check. If the delegated token used for PIM role activation has expired overnight, the device code URL appears at the top of the digest — re-auth happens once, proactively, before the phones start ringing.

Sections:

- **Needs Action Today** — new tickets requiring a response, with enriched detail inline
- **Sara's Queue / Britton's Queue** — their new items at a glance
- **Pending / On Hold** — parked tickets with a clear hold reason noted

Plain ASCII throughout — no encoding issues when pasting into Teams or Outlook.

The **Needs Action Today** section goes beyond a plain ticket list. For every ticket in that section, Claude automatically runs lookups and appends the results inline — no second command, no second output:

- **All tickets:** Intune device lookup by user name — hostname on the same line as the ticket, ready to paste into TeamViewer
- **Printer tickets:** Live query to the print server — printer IP, driver, and a diagnostic note alongside the hostname
- **Multi-user requests:** One line per user, hostname and abbreviated title aligned
- **Teams outreach draft:** A ready-to-send message pre-written for the specific issue, directly under the lookup data. Messages are reviewed and dispatched manually — nothing is auto-sent.

**Example — printer ticket:**

Real ticket: [GHD-97514](https://usana.atlassian.net/browse/GHD-97514) — Kate Cecotti, Lead Scientist, couldn't connect to the 3rd floor printer. Claude queried the print server live to pull the IP and driver alongside the Intune hostname lookup.

```
NEEDS ACTION TODAY
------------------
GHD-97514 [New] Kate Cecotti - Printer Error #740, USSL_3F_Printer23
  Machine: USSLW-hwZqXvT4T | IP: 10.1.190.42 | Driver: SHARP UD2 PCL6 | Error 740 = elevation required
  Teams: "Hey Kate, just saw your ticket about the printer. Error 740 means it needs an
          elevated install - I can remote in and get that sorted. Let me know when works."
```

**Example — multi-user software request:**

Real ticket: [GHD-97506](https://usana.atlassian.net/browse/GHD-97506) — Rayne Moore requested Claude Desktop; Jaclyn Johnson tagged herself in as a second requestor. Later in the same session, Crystal Bird asked to be added too. Claude resolved hostnames, ran an Active Directory group check on Crystal to confirm whether she had the `Azure_Access_Claude_Enterprise` permission group (it was unclear at the time), and surfaced that her manager on record is Rayne Moore — the same person who opened the ticket. Rayne is already in the loop.

```
GHD-97506 [New] Claude Desktop - Rayne Moore, Jaclyn Johnson, Crystal Bird
  Rayne Moore (Dir IT PM):    USSLW-ztcDh3O8m
  Jaclyn Johnson (VP PM):     USSLW-6QCr4eKxD
  Crystal Bird (Mgr IT PM):  No Intune device - awaiting TeamViewer ID (Teams sent)
  Teams: drafted for all three
```

The full workflow — Jira query, Intune lookups, print server query, Teams draft generation — runs in one session and lands on the clipboard in a single paste-ready output.

---

### End of Shift Digest

At the end of the day, one phrase triggers a full shift summary. Claude runs three Jira queries in parallel and copies a formatted plain-text digest straight to clipboard:

- **Closed Today** — tickets resolved during the shift, with resolution detail and time closed
- **Progressed Today** — open tickets updated or moved forward, with current status and last action
- **New Assignments** — tickets that arrived today and are still open (the next morning's queue)

Shift totals at the bottom: `X closed | X progressed | X new | X total actioned`

Before pulling Jira data, Claude checks memory for any Slack threads flagged for monitoring from previous sessions. Those threads are read first and any updates surface at the top of the digest — so nothing falls through the cracks between shifts.

Real example: A network device ticket had been pending confirmation from a Netops engineer for two days. The EOD digest opened with his reply confirming resolution. The ticket was closed in the same session and the watch entry removed from memory automatically.

---

## 15. Persistent Memory — Learns and Doesn't Forget

Everything described in this document is stored in Claude's persistent memory and survives across sessions. Brent does not re-explain context each morning.

Examples of what Claude remembers without being reminded:
- Which team uses Slack vs. Teams (confirmed from a real incident where the wrong channel was initially used)
- That Jabber reinstall tickets route to Software component, not Desk Phone
- Every open WinOps approval ticket and its current status
- The printer hardware vendor's number and when to call them vs. handling in-house
- Department-specific printer locations by floor

Claude also learns from corrections. If Brent says "no, that's wrong, do it this way," Claude saves that as a rule and never makes the same mistake again. Corrections from real tickets — not hypotheticals.

---

## API Approvals Status

| Ticket | Request | Status | Result |
|---|---|---|---|
| WINOPS-19012 | Intune service principal (Graph API app-only credentials) | **Approved & closed** | Token never expires — Intune lookups fully silent all day |
| WINOPS-19019 | Jamf API client (OAuth client credentials) | **Approved & closed** | Mac device lookups fully automated — same silent experience as Intune |
| WINOPS-19062 | BitLocker.Read.All Graph API permission | **Approved & closed** | BitLocker recovery keys retrieved in seconds from plain English — no Entra portal |
| WINOPS-19021 | PagerDuty API key | Open | Page teams directly from conversation — no browser, no SSO flow |

Once WINOPS-19021 is approved: *"Page the network team for this incident"* becomes a single command that fires the PagerDuty alert, creates the ITIL ticket, and posts to the status channel simultaneously.

---

## What This Is Not

Claude is not a chatbot used to answer trivia questions or generate images. Every capability described here was built to solve a real GHD problem:

- A help desk that is critically understaffed for the scale of the environment. There are counterparts in Asia Pacific and Austria who cover phones outside of SLC hours — but the overwhelming majority of hands-on support, escalations, incident management, and tooling is handled by two people in Salt Lake City.
- Hundreds of users across 20+ countries spanning five continents, many of whom contact SLC directly regardless of time zone.
- Thorough documentation required at every escalation step
- Morning call volume from Shipping before most IT staff are even at their desks.
- P1 incidents that require parallel tracking of multiple Slack threads simultaneously.
- An environment too large and fast-moving for any static reference document to keep up.

Practically speaking: two people, one environment spanning five continents.

Claude's value here is not novelty. It is speed, accuracy, and institutional memory — the things a team this size cannot maintain manually at this volume.

---

*USANA Health Sciences — Global Help Desk*
*brent.christensen@usanainc.com*
