---
title: "Multi-stage mass bot attack on MediaWiki: incident analysis"
date: 2026-09-28 12:00:00 +0200
categories: [Incident Response]
tags: [forensics, mediawiki, bot, cloudflare]
description: "A multi-stage, adaptive bot attack on a self-hosted MediaWiki: 856 bot accounts in ~1h20, reconstructed minute by minute from a pre-remediation database backup."
---

> 🇵🇱 **[Czytaj po polsku](/pl/atak-bota-mediawiki/)**

**Incident date:** 28–30 December 2025  
**Reconstruction:** ~six months later, from a database backup taken before remediation  
**Author:** Krzysztof Pieczka

---

## TLDR

I run a self-hosted wiki (MediaWiki, Linux VPS, ~300 daily users) with a custom file upload form written in Flask. On 29 December 2025 the site was deliberately attacked. The attack was **multi-stage and adaptive**: whenever I blocked one vector, the attacker switched to another. After the form was spammed and I took it offline, the attacker pivoted to **mass account registration** - in about 1h20 (16:17–17:35) they created **856 bot accounts** (77% of all accounts on the site; peak: **42 registrations per minute**). I stopped the attack by closing registration entirely; the next day I reopened it with a CAPTCHA, and a few days later I preventively moved verification to Cloudflare Turnstile. Six months later I reconstructed the attack minute by minute from a database backup taken before cleanup - below is the full analysis, timeline, telemetry and lessons learned.

---

## 1. Context and infrastructure

- **Site:** self-hosted wiki (MediaWiki, MySQL/MariaDB), Linux VPS, Apache (MediaWiki/PHP) + mod_wsgi (Flask).
- **Upload form:** a separate Flask app (`/formularz`) with an SQLite database, accepting links, images and videos.
- **Registration protection on the wiki (before the attack):** custom verification questions (MediaWiki ConfirmEdit, **QuestyCaptcha** module) - a pool of about 8 randomly chosen, simple questions about the wiki. Deliberately chosen for the threat model at the time: mass spam bots pasting links (casinos, Russian sites etc.) that don't know the site and can't answer these questions. I did not expect an attack from someone inside the community.
- **Traffic:** ~300 users per day.

**Honest context:** the upload form went live the day before the attack (28 Dec), built quickly and without security in mind.

---

## 2. Attack timeline

### Phase 0 - 28 Dec 2025: form goes live
The upload form is launched and receives its first submissions.

### Phase 1 - 29 Dec, late morning: form spam + IP rotation
A bot started flooding the form with submissions - mostly **file uploads (video/image)**, not links.
- **First response:** rate limiting + IP blocking (in the app and in Apache: `Require not ip ...` → `403`).
- **Attacker adapts:** switches to **rotating IP addresses** - per-IP blocking stops working.
- **Second response:** content-based blocking (deduplication/cooldown for identical submissions) + a CAPTCHA under consideration.
- **Containment:** with no reliable way to tell legitimate traffic from the bot under IP rotation, I **temporarily disabled the form** to stop the attack and protect the rest of the site.

> Observation from that day: the first wave started around 11:45, and IP rotation began around 12:00. The data that survived only covers accepted submissions - blocked traffic was never recorded, so the peak of this phase isn't visible in the data. These times are first-hand observation.

### Phase 2 - 29 Dec, 16:17–17:35: pivot to mass account registration
After the form went offline, the attacker **changed vectors**: they started mass-registering accounts on the wiki, bypassing the verification questions. The pool had about 8 questions with fixed, simple answers, and the randomly selected question appeared in the form as plain text. All it took was collecting all the questions once and giving them to the bot, together with the answers, as a lookup table. Picking randomly from 8 questions doesn't make this any harder - it's still a **static secret**: solve it once and it works forever. This stage is fully documented in the data:

- **856 accounts** created on 29 Dec within about 1h20 (confirmed with an SQL query on the `user_registration` field).
- Hourly breakdown: **16:00 → 142, 17:00 → 712, 18:00 → 2.** (83% of the volume in the 17:00–17:59 hour.)
- Peak: **42 registrations in a single minute (17:18)**.
- Per-minute curve: warm-up (16:17–16:40, a few accounts/min) → escalation (16:43–17:10, up to ~15/min) → **3 minutes of silence (approx. 17:11–17:13)** → full assault (17:14–17:35, 23–42/min) → sharp drop after 17:35.
- **Intervention:** the drop after 17:35 is the moment I **closed new account registration entirely**. The assault stopped, but not to zero - another **19 accounts** were created before 18:00 (details and hypotheses in section 3).

### Phase 3 - 30 Dec: registration reopened with a CAPTCHA
The next day I reopened registration and added a **CAPTCHA** (the MediaWiki **ConfirmEdit** extension with reCAPTCHA - requires an API key). This was the first layer after closing registration.

### Phase 4 - 2 Jan: Cloudflare Turnstile
A few days later I **preventively** replaced the CAPTCHA with Turnstile, because I considered it a better layer. The difference from a CAPTCHA: Turnstile doesn't hand the bot a puzzle to solve - Cloudflare makes the "human or bot" call based on browser signals, and the application only verifies the issued token server-side. Turnstile is still active on registration today (the site's infrastructure has changed since, but this layer stayed).

> The form was eventually disabled for good.

---

## 3. Data analysis - forensic reconstruction

### Data source
Primary source: **`backup_before_delete_20251229.sql`** - a full dump of the MediaWiki database (MySQL) taken on **29 Dec, before the bot accounts were deleted**. Taking that backup before cleanup turned out to be crucial - without it, the data would have been lost. Unfortunately I wiped the form data (SQLite database) without a copy, so the telemetry mainly covers the registration phase.

### Scale
- **856 accounts** registered on 29 Dec 2025 (confirmed: `SELECT COUNT(*) FROM wiki_user WHERE user_registration LIKE '20251229%'`).
- **1114 accounts in total** in the database → the **856 bot accounts are ~77% of all accounts ever created on the site**.
- *Methodology note:* my initial `grep` on the raw dump returned 1713 date matches - inflated, because each `wiki_user` row has two timestamp fields (`user_registration` and `user_touched`). Only an SQL query on the registration field alone gives the exact number. A good illustration of why you verify data at the source instead of trusting the first grep.

### Attack curve

![Attack curve – account registrations per minute, 29 Dec 2025](/assets/img/bot-attack-curve-29122025.png)

*Bot account registrations per minute.*

| Hour (29 Dec) | Registrations |
|---|---|
| 16:00–16:59 | 142 |
| 17:00–17:59 | 712 |
| 18:00 | 2 |

The shape of the chart is a typical signature of an automated attack.

### The pause before the assault
The most interesting moment on the chart isn't the peak but **the three minutes of silence right before it**. Until about 17:10 the bot registers a few to a dozen-odd accounts per minute, then for about 17:11–17:13 not a single account is created, and from 17:14 the rate jumps straight to 23–37 accounts/min and stays there for ~20 minutes.

This doesn't look like a tool ramping up naturally, but like **a stop and restart with a different configuration** - most likely with more threads/parallel sessions. A step change in throughput after a short pause suggests the attacker was **actively watching and tuning the attack.** It fits the rest of the picture: switching vectors after the form went offline, and manual test accounts before the automation.

### The tail after registration closed
After registration was closed (~17:36), another **19 accounts** were created: 7 at 17:37, then small batches (4, 5, 1) up to 17:59, and 2 at 18:00. The assault stopped, but not to zero. Two hypotheses:

1. **Closing registration wasn't instant** - e.g. the first configuration change didn't lock everything down immediately, and some requests kept getting through for another quarter of an hour.
2. **A different registration path.** If closing registration only covered the registration page (the form) and not the account-creation permission itself, accounts could still be created, e.g. through the MediaWiki API (`action=createaccount`).

The backup data doesn't settle this, and the Apache logs from that day no longer exist. I'm leaving it as an open question.

### Username generator signature (mini threat intel)
The distribution of bot username "families" reveals how the tool worked:

| Name pattern | Accounts | Interpretation |
|---|---|---|
| `DawidKamilPatryk NNNNNNN` (one base + random 7-digit suffix) | 844 | core of the attack, machine-generated |
| `PentestUser NNNNN` | 5 | tool's default names |
| `Tester NNNN` | 4 | test phase |
| `BotTest` | 1 | test before automation |
| single, manually typed names (MatiPiw, Kuszot) | 2 | manual warm-up before automation |

**Takeaways from the signature:**
- The prefix references a **recognisable Polish internet personality** - which points to a **deliberate, targeted attack** (an attacker who knows the site and its community), not a random botnet scan.
- Names like `PentestUser`/`Tester`/`BotTest` suggest an off-the-shelf registration/spam tool with default naming.

---

## 4. Lessons learned
- **The adversary can be intelligent and adaptive.** Every individual defence - IP blocking, the verification question - was bypassed by the attacker changing approach. The attack ended when registration was closed, and the gap the bot exploited was closed by reCAPTCHA, and then Cloudflare Turnstile. Patching individual vectors doesn't scale against an adaptive attacker.
- **Per-IP blocking doesn't work against IP rotation.** A sensible first line, but easy to get around.
- **Containment is a good call, not a failure.** Taking the form offline, and then registration, limited the damage and bought time for a better defence.
- **The static questions were a good defence - for a different threat model.** They filtered out spam bots, but they're a static secret: anyone who knows the community collects the answers once and automates forever.
- **Preserve evidence before cleanup.** Backing up the wiki database before deleting the accounts saved the entire analysis; I wiped the form database without a copy and that phase was lost. The Apache logs (`rotate 14`) didn't survive until the reconstruction either.

**What I'd do differently:** Turnstile and edge rate limiting from day one, not reactively; a backup and log-retention policy set before the site goes public, not after an incident.

---

*Analysis based on data recovered from a full MediaWiki database backup taken before remediation (29 Dec 2025).*
