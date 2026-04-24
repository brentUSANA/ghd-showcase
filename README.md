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
3. [Intune Device Lookups](#3-intune-device-lookups)
4. [Printer & Toner Workflow](#4-printer--toner-workflow)
5. [Incident Management](#5-incident-management--real-time-coordination)
6. [Email Investigation](#6-email-investigation)
7. [Living Knowledge Base](#7-living-knowledge-base)
8. [Lansweeper Asset Management](#8-lansweeper-asset-management)
9. [Application Access Provisioning](#9-application-access-provisioning)
10. [SentinelOne Threat Response](#10-sentinelone-threat-response)
11. [Teams & Slack Communication](#11-teams--slack-communication-drafting)
12. [GHD Daily Digest Dashboard](#12-ghd-daily-digest-dashboard)
13. [GHD Digests](#13-ghd-digests)
14. [Persistent Memory](#14-persistent-memory----learns-and-doesnt-forget)
15. [Open API Approvals](#open-api-approvals-in-progress)

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

**Ticket description standards (trained):**
- Description = device + symptom + behavior. No "user reported..." narration.
- Resolution steps go in a separate comment, never the description.
- Location names written in full — no internal abbreviation codes.
- Hostname always included — Claude looks it up in Intune before writing the ticket.

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

**PIM role activation — automated (`/pim` command):**

Many AD, Exchange, and Azure operations require activating a Privileged Identity Management (PIM) role first. Previously this meant navigating the Azure portal manually — select role, set duration, enter justification, submit, wait. Done in a browser every time.

Now: type `/pim` in the terminal. A numbered menu appears listing every available role. Pick one, enter duration and justification. A PowerShell script activates the role via the Microsoft Graph API. No browser, no portal navigation.

Before running an Exchange trace or an Azure device lookup, Claude reminds you to activate the required PIM role — one command in the terminal, then back to the ticket.

---

## 3. Intune Device Lookups

Claude looks up any Windows device in Intune by user's full name and returns hostname, device model, serial number, OS version, last check-in, and compliance status.

**How it works:**
- Script lives at a designated path on the Windows machine (Claude-managed, separate from user-deployed tools)
- Lookups run fully silently — no authentication prompts required

Real ticket example: [GHD-97382](https://usana.atlassian.net/browse/GHD-97382) — BitLocker recovery key request. Hostname retrieved via Intune lookup and included in the ticket automatically.

**WINOPS-19012 — Approved & closed (April 2026):** Service principal granted. Token does not expire — Intune lookups are now fully silent all day, every day.

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
- DHCP printer issues always get "Printer DHCP Reservation" in the title so the network team knows immediately what they're looking at
- Claude flags any printer IP outside the expected VLAN subnet as a missing DHCP reservation before the ticket is written

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

**Open ticket — watching:**
`WINOPS-19021` — Requesting a PagerDuty API key so Claude can trigger alerts directly from conversation instead of manually. Once approved: identify incident, page the right team, create the ITIL ticket, and post to the status channel — all in one command.

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

## 8. Lansweeper Asset Management

Claude looks up hardware assets in Lansweeper to find hostname-to-user mapping, purchase dates, warranty status, and device specs.

**Trained behavior:**
- Uses the correct username format for lookups (ignores legacy domain prefix entries)
- Knows that Jira and AD descriptions never contain hostnames — goes straight to Lansweeper or Intune rather than guessing

**Open ticket — watching:**
`WINOPS-19019` — Requesting a local read-only Jamf account for Mac device lookups. Right now Mac lookups require opening the Jamf portal manually. Once approved, Claude can query Mac device info the same way it queries Windows devices via Intune.

---

## 9. Application Access Provisioning

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

---

## 10. SentinelOne Threat Response

Claude knows the malware detection workflow end-to-end.

**What it can do:**
- Walk through the triage checklist (Action Status: Failed vs. Succeeded)
- Identify when to physically pull a machine vs. remote remediate
- Draft the desk note to leave if a machine is pulled without reaching the user
- Know that GHD can clear detections and reconnect devices, but exclusion rules require InfoSec

Real ticket: [GHD-97386](https://usana.atlassian.net/browse/GHD-97386) — SentinelOne false positive locked a Mac off the network. GHD cleared the detection and reconnected the device. Exclusion request forwarded to InfoSec and tracked as an open item.

---

## 11. Teams & Slack Communication Drafting

Claude drafts Teams and Slack messages with trained behavior:

- No em dashes or arrow characters — they render as garbage in Teams. Claude uses plain hyphens.
- Messages are personalized — pulls the specific ticket or issue detail, never uses generic placeholder phrasing.
- Multi-step instructions are offered to clipboard rather than pasted raw, because terminal copy-paste garbles formatting when a user pastes into Teams or a ticket.

---

## 12. GHD Daily Digest Dashboard

A custom-built visual ticket queue dashboard:

- Pulls live data from Jira REST API
- Shows all open GHD tickets assigned to the team
- Highlights low-hanging fruit (tickets likely closeable quickly)
- Launches via a batch file on Brent's desktop each morning
- Runs a local CORS proxy so the browser can talk to Jira securely from WSL

This was entirely designed and built by Claude based on Brent describing what he wanted to see each morning.

---

## 13. GHD Digests

Three structured digest outputs bracket and close the workday. Each is a single command — or a single phrase — that produces clipboard-ready plain text with no manual aggregation.

---

### Morning Digest

The first output of the day. One command pulls the full open ticket queue from Jira and returns a categorized snapshot:

- **Needs Action Today** — new tickets requiring a response
- **Follow Up Today** — in-progress work with expected movement (hardware returns, pending callbacks)
- **On Hold** — parked tickets with a clear hold reason noted
- **Waiting for Dell Stock** — refresh queue grouped separately so it does not dilute the action list

Plain text, clipboard-ready, safe to paste directly into Teams or a notes app without encoding issues.

---

### Additional Info Digest

Immediately after the morning digest, Claude runs a second pass on every Needs Action ticket and enriches it with live lookup data — without being asked.

What gets pulled automatically, per ticket type:

- **All tickets:** Intune device lookup by user name — returns hostname, which is the first thing needed to remote in or identify the machine
- **Printer tickets:** Live query to the print server — returns printer name, IP address, and driver. Hostname, IP, and a diagnostic note land on one line so everything needed to resolve the call is available before picking up the phone
- **Multi-user requests:** One line per user, hostname and abbreviated title aligned — readable at a glance

Each ticket in the Additional Info digest also includes a ready-to-send Teams outreach message, pre-written for the specific issue. Messages are reviewed and dispatched manually — nothing is auto-sent.

**Example — printer ticket:**

```
GHD-XXXXX [New] Jane Smith - Printer Error #740, USSL_3F_PrinterXX
  Machine: USSLW-XXXXXXXX | IP: 10.1.190.XX | Driver: SHARP UD2 PCL6 | Error 740 = elevation required
  Teams: "Hi Jane, I saw your ticket about the printer connection error. I can remote
          in and get that installed for you - just let me know when works."
```

**Example — multi-user software request:**

```
GHD-XXXXX [New] Application install request
  User A (Dir IT PM):  USSLW-XXXXXXXX
  User B (VP PM):      USSLW-XXXXXXXX
  Teams: "Hi [User A], I saw your install request and am looking into getting that
          set up - I will be in touch shortly."
```

**Why two outputs instead of one:**

The morning digest is fast to scan for priorities and safe to share. The additional info digest is dense with action data and personal. Keeping them separate means neither is cluttered by the other. The full workflow — Jira query, Intune lookups, print server query, Teams draft generation, clipboard delivery — runs in one session with no manual steps.

---

### End of Shift Digest

At the end of the day, one phrase triggers a full shift summary. Claude runs three Jira queries in parallel and returns a formatted plain-text digest copied to clipboard:

- **Closed Today** — tickets resolved during the shift, with resolution detail and time closed
- **Progressed Today** — open tickets updated or moved forward, with current status and last action
- **New Assignments** — tickets that arrived today and are still open (the next morning's queue)

Shift totals at the bottom: `X closed | X progressed | X new | X total actioned`

Before pulling Jira data, Claude checks memory for any Slack threads flagged for monitoring from previous sessions. Those threads are read first and any updates surface at the top of the digest — so nothing falls through the cracks between shifts.

Real example: A network device ticket had been pending confirmation from a Netops engineer for two days. The EOD digest opened with his reply confirming resolution. The ticket was closed in the same session and the watch entry removed from memory automatically.

---

## 14. Persistent Memory — Learns and Doesn't Forget

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
| WINOPS-19019 | Local Jamf read-only account | Open | Mac device lookups automated, same as Windows via Intune |
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
