---
title: "TryHackMe: GLA II — Robbing the Real Vault"
date: 2026-08-18 12:00:00 +0500
categories: [CTF, TryHackMe]
tags: [reverse-engineering, godot, api, ctf-writeup]
---

## Recon

Started with a standard nmap scan against the target.

```bash
nmap -Pn -sV 10.113.172.96 -sC
```

Results showed two open ports:
- **22/tcp** — OpenSSH 9.6p1 (Ubuntu)
- **80/tcp** — Kestrel (ASP.NET Core's web server)

`Server: Kestrel` was the first real clue — this isn't a typical Apache/nginx site, it's a .NET backend API.

![nmap scan](/assets/img/gla2/01-nmap.png)

## First Look at the Web Service

Added the lab's hostname to `/etc/hosts`:


10.113.172.96 gla2.thm


Browsing `http://gla2.thm/` gave a plain `404 Not Found`. At first this looked broken, but a 404 on `/` from an API server is completely normal — there's no HTML page to serve at the root, only specific endpoints.

Confirmed the server was alive and responding by hitting `/health`:

```bash
curl http://gla2.thm/health
```

```json
{"ok":true}
```

![health check](/assets/img/gla2/02-health.png)

## Digging Into the Game Client

The lab provided a downloadable game — **Grand Larceny Auto**, a GTA-style Godot game. Ran `file` on the shipped binaries:

```bash
file *
```

- `GrandLarcenyAuto.x86_64` → ELF 64-bit executable
- `GrandLarcenyAuto.pck` → Godot's packed resource archive (`GDPC` magic bytes)

Confirmed the engine version:

```bash
strings GrandLarcenyAuto.x86_64 | grep -i godot
```

→ **Godot Engine v4.7.1.stable.mono.official**

![godot version](/assets/img/gla2/03-godot-version.png)

## Extracting and Decompiling the Game

Used **GDRE Tools** to extract the `.pck` and decompile the packaged C# scripts:

```bash
./gdre_tools.x86_64 --headless --recover="GrandLarcenyAuto.pck" --output-dir="extracted/"
```

This pulled out 8 decompiled C# scripts, including two that mattered:

- `SafehouseVault.cs` — the in-game vault
- `PoPClient.cs` — the game's network/API client

![gdre extraction output](/assets/img/gla2/04-gdre-extract.png)

## The First Flag (a Decoy)

`SafehouseVault.cs` printed a flag directly — but it was self-aware about being fake:

THM{th3_v4ult_w4s_4_d3c0y}


![decoy flag in source](/assets/img/gla2/05-decoy-flag.png)

## Finding the Real Protocol

`PoPClient.cs` laid out the actual backend API the game talks to:

- `POST /session` — starts a session, returns `session_id`, a `token`, and a randomized `stash_order`
- `POST /checkpoint` — reports progress through steps (`heat5` → 3 stashes in order → `vault`), each request HMAC-SHA256 signed with a key hardcoded in the binary
- `POST /claim` — claims a flag, but takes a `role` parameter

A method called `DeriveStaffRole()` stood out — it builds a string from the stash order and SHA-1 hashes it, clearly meant to be used as a hidden `role` value instead of `"player"`.

![PoPClient.cs source](/assets/img/gla2/06-popclient-source.png)

## Writing a Script to Replay the Protocol

Instead of playing the game and hoping the traffic lined up, wrote a Python script to talk to the API directly, replicating the exact request sequence and HMAC signing scheme found in the decompiled source.

![solve.py script](/assets/img/gla2/07-solve-script.png)

## Debugging — Hint #1: Too Fast

First run failed:


HTTP 425 error body: {"error":"too_fast","need":6,"got":0.27}


The server enforces a minimum delay between steps to simulate real gameplay pacing. Fixed by adding `time.sleep(7)` between each request.

![too fast error](/assets/img/gla2/08-too-fast-error.png)

## Debugging — Hint #2: Bad Token

Second run failed differently:

HTTP 401 error body: {"error":"bad_token"}


Turned out each checkpoint response returns a **new rotating token** that must be used in the next request — the script was still using the original token from `/session`. Fixed by updating the token after every response.

![bad token error](/assets/img/gla2/09-bad-token-error.png)

## Getting the Real Flag

With both issues fixed, the script ran the full sequence cleanly and submitted two claims:

- `role: "player"` → `THM{n1c3_dr1v1ng_but_th4ts_th3_wr0ng_v4ult}` (note: *"civilian access — the real vault is staff-only"*)
- `role: <SHA-1 staff hash>` → **`THM{Th4ts_th3_wr0ng_g4m3_t0mmy}`**

![final flag output](/assets/img/gla2/10-final-flag.png)

## Takeaway

The in-game vault was pure set dressing. The real access-control decision happened entirely server-side, based on a `role` string the client itself computed and sent — nothing stopped a player from computing that same hash and sending it directly, once the client-side logic was read.




