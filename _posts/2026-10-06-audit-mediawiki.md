---
title: "Security audit of my own MediaWiki server"
layout: post
categories: [Security Audit]
tags: [mediawiki, apache, cloudflare, web-security]
description: "A white-box audit of my self-hosted MediaWiki before migration. How one directory listing exposed the app's source and a user database."
last_modified_at: 2026-10-06 12:00:00 +0200
image:
  path: /assets/img/audit/audit-thumb.png
  alt: "Security audit - directory listing"
---

> 🇵🇱 **[Czytaj po polsku](/pl/audyt-mediawiki/)**

Since August 2025 I've been running a self-hosted wiki on MediaWiki. I set it up myself on a VPS and today it gets around 300 users a day. I built it feeling my way, with AI: learning as I went and adding things whenever I needed them. In December 2025 I bolted on my own upload form in Python Flask, quickly and without thinking about security. Two days later the site got attacked, which I wrote up in a [separate post](/posts/bot-attack-mediawiki/).

In September 2026 I decided to migrate the wiki to another VPS, and I used the chance to properly go through the whole thing before moving it. Over the past year I've learned a lot about security and today I understand far more than when I set all this up. Before I moved anything, I went through everything and wrote down the mistakes that had been sitting in it the whole time.

The audit is white-box: it's my own infrastructure, I know the code and I have access to everything. That's a rare opportunity, because on my own system I can test things I'd never let myself touch on someone else's.

> Everything below is the state before the migration. As of October 2026 the site is freshly set up on a new VPS and every exposed secret has been rotated. None of this can be reproduced today!

In this post I cover the most interesting findings. The full report with all findings, PoCs and screenshots: **[download PDF](/assets/audit-report.pdf)**.

## 1. Directory listing ON (HIGH)

I had directory listing turned on, and in practice anyone could download the database with user data. The Apache config had `Options Indexes`, so any directory without an `index` file was simply listed. Visiting `/website/` returned "Index of /website" and showed what was sitting there: `formularz/` and `upload/`.

![Index of /website - directory listing through Cloudflare](/assets/img/audit/audit1.png)


And inside sits `app.py`. The upload form (a Flask app) physically lived in Apache's docroot. Normally traffic went through a reverse proxy to `/formularz/` and Flask handled it. But since the files physically sit in the docroot, the same app is reachable a second way: under `/website/formularz/...` Apache serves it statically, bypassing Flask. Apache has no handler for `.py` or `.db`, so instead of executing those files it hands them over as a plain file to download.

```
$ curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/website/formularz/app.py
200

GET https://[REDACTED]/website/formularz/app.py
StatusCode: 200 | Content-Type: text/x-python | Content-Length: 8466
```

![app.py served by Apache, password masked](/assets/img/audit/audit2.png)

That source contained the admin password in plaintext and the Turnstile secret key - the same one the production wiki uses. From a single downloaded file you get the keys to the form's admin panel and the secret that was supposed to protect registration across the whole wiki.

Right next to it, in the same directory, sat the `submissions.db` database. Also downloadable with a single GET:

![Downloading submissions.db](/assets/img/audit/audit3.png)

The database held nicknames, descriptions, real IPs of the people who submitted, file paths and timestamps. It was downloadable from the moment the form landed in the docroot (the directory is dated 27.12.2025) until the migration, so about 9 months. That's **real personal data publicly available for that entire time**.

The whole chain: directory listing shows the files → WSGI app in the docroot → Apache serves its source as a static file → hardcoded secrets in that source → public exposure of the admin password and Turnstile key → access to the admin panel and the entire database of user data.

Fix: turn off directory listing (`Options -Indexes`) and move the WSGI app out of the docroot (e.g. `/opt/formularz`). One enabled listing and one badly placed directory turned an internal app into public source code and a database of user data.

## 2. Server reachable bypassing Cloudflare (MEDIUM)

After the bot attack, the site's entire protection rested on Cloudflare: challenge, rate-limit, WAF rules. Traffic was supposed to go through CF first and only then to my server. The problem is that the server answered anyone who knocked directly on its IP.

All it takes is sending a request to the server's IP with the domain in the `Host` header:
```
$ curl -skI -H "Host: [REDACTED]" https://[REDACTED_IP]/
HTTP/1.1 200 OK
Server: Apache/2.4.58 (Ubuntu)
```

The server returns the full wiki page, and Cloudflare takes no part in it at all. No challenge, no rate-limit, and the WAF rules never even get a chance to fire. Everything I set up after the bot attack **could be bypassed with a single header**.

The server's real IP can be found without much effort: in DNS history (before the domain moved behind Cloudflare it pointed straight at the server) or on Shodan, which scans the whole internet and indexes what responds at a given address.


Fix: a firewall on ports 80/443 allowing only Cloudflare's IP ranges and dropping everything else. Then the only route to the server goes through CF and the protection set up there actually means something.

## 3. Half the stack with no update path (HIGH)

MediaWiki was running version 1.43.0. That's the first release of this branch, from December 2024, and since I set the site up it never got a single one of the later patches. The latest release on the same branch is now 1.43.9, so about 9 patch releases that I never had. On top of that, 1.43.0 was already out of date at the moment I installed it.

![Special:Version - versions of the installed software](/assets/img/audit/audit4.png)

Not all of the server was neglected. Apache, PHP and MySQL were up to date (the Apache build date is July 2026), meaning the system was patching them continuously. And here's the crux: those components are installed by `apt`, so `apt upgrade` pulls in security patches on its own. MediaWiki and its extensions I set up from a tarball, outside `apt`, so no `apt upgrade` ever sees or touches them. Half the stack updated itself, and the other half - the part most exposed to the world - had no update mechanism at all. Nobody was watching it. Back then I wasn't into security at all and had no idea that running an old version is a real problem (mostly XSS), not cosmetics. I assumed that if the system patches itself, everything patches itself.

Those 9 releases mostly contain a family of XSS/escaping bugs and a few info-disclosure ones (leaking metadata, hidden usernames, etc.). Part of the risk was cut down by the controls left over from the bot attack: anonymous editing disabled and Turnstile on registration. But XSS through page content also affects logged-in, trusted editors, so "an outsider can't post anything" isn't enough here.

Exactly the same problem applied to the extensions. EmbedVideo was running a 2022 version, about 4 years without updates, for the same reason: third-party, outside `apt`, with nothing watching it. This particular extension embeds external content (video URLs) and processes input, which makes it a classic spot for XSS, so being out of date hurts twice here.

The fix isn't "clicking update". Auto-updating MediaWiki is a bad idea, because a release can require a database schema migration (`maintenance/update.php`) and can break extensions. What actually closes the hole:
- a fresh install from a current tarball during the migration, without copying the core 1:1,
- subscribing to the `mediawiki-announce` list, just to know a security release has come out at all,
- for the system layer (`apt`), `unattended-upgrades` with security updates only, closing off the half that was working anyway.

## Takeaways

All three findings are, at bottom, the same mistake: every time, **security rested on a single safeguard**. Secrets "protected" by the fact that nobody knows the path to the file. All traffic "protected" by the fact that it goes through Cloudflare. The server's currency "ensured" by the assumption that `apt` handles everything. In each case it was enough to break that one assumption and the whole protection vanished, because there was nothing underneath.

The most dangerous thing wasn't any single bug, but the fact that they chained. Directory listing on its own is a trifle. The form in the docroot on its own is a trifle. Only together did they give up the user database in one click. Each finding sounds harmless on its own; the chain doesn't.

Looking back, what helped most wasn't the patching itself, but looking at my own system through the eyes of someone trying to break in. A year ago I wouldn't have spotted these things, because I was building for "it works", not for "it can't be bypassed". Those are two different ways of looking at the same code.
