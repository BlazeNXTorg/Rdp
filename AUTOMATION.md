# BlazeNXT Workstation — Always-On Automation

## Status: LIVE

| | |
|---|---|
| Cron | `0 */6 * * *` (UTC) — every 6 hours |
| Fires at | **05:30 / 11:30 / 17:30 / 23:30 IST** |
| Session length | 350 min (auto-stops, leaving slack for the next cron) |
| Free runner | 4 vCPU / 16 GB (public repo, unlimited free minutes) |
| Stable node name | **`blazenxt-ws`** (scheduled runs) |
| Live manual run now | `blazenxt-ws-37731636723` → ends ~16:42 IST today |
| Password | `RDP_PASSWORD` repo secret (never changes between sessions) |

---

## How the chain works

```
05:30 IST  cron fires  ──> session starts (~2 min boot + ~5 min provisioning)
11:20 IST  session self-stops at 350 min
11:30 IST  next cron fires ──> queued, starts as soon as runner is free
17:20 IST  self-stop  ... and so on, forever
```

Gap between sessions: **~10-20 minutes every 6 hours**. A session can never be
truly continuous — 355 minutes is GitHub's hard ceiling per job.

If a cron fires while a session is still alive, `concurrency` keeps it
**pending** instead of starting a second machine. It takes over the moment the
old one dies. So at 11:30 IST today you will see a second run sitting in
**pending** for ~5 hours — that is correct behaviour, not a bug.

---

## Your phone: set it up once

Use the stable name and you never touch the config again:

| Field | Value |
|---|---|
| PC name | `blazenxt-ws`  *(or the Tailscale IP from the run summary)* |
| User | `BlazeAdmin` |
| Password | your `RDP_PASSWORD` secret value |

Add it once in Microsoft's Remote Desktop app. Every scheduled session answers
to that same name with the same password.

---

## Killing it

**Actions → BlazeNXT Workstation STOP → Run workflow → type `STOP` → Run**

That does two things:
1. **Force-cancels** every live session (plain cancel is ignored by the keep-alive loop)
2. **Disables the cron**, so nothing restarts in 6 hours

Back on again later: Actions → *BlazeNXT Workstation Cloud RDP* → **Enable workflow**.

---

## Continuation between sessions

Each session now saves its state and the next one restores it automatically.

| | |
|---|---|
| Persistent folder | **`C:\BlazeNXT\workspace`** — the only folder that survives |
| Saved | at shutdown, uploaded as artifact `blazenxt-session-state` |
| Restored | ~1 min after the next session boots, before you connect |
| History kept | 3 newest backups (older ones pruned automatically) |
| Lifetime | 7-day retention, then GitHub deletes it |

**Put your work in `C:\BlazeNXT\workspace`.** Anything else — Desktop, Downloads,
installed apps, Docker images, open windows — is gone when the session ends.

### Optional: carry the whole profile too

Set **`persist_user_folders: true`** and Desktop/Documents/Downloads come along as
well. Read the warning first:

> **This repo is public.** Artifacts on a public repository are downloadable by
> anyone with read access — i.e. anyone at all. A profile backup would contain
> browser data, downloads, documents. Only enable this if you genuinely do not
> care who reads that content, or move backups to a private repo.

`persist_max_mb` (default 1024) caps the backup; over the cap it falls back to
the workspace folder only.

### Why sessions are now 330 min instead of 350

Saving state needs time inside the job budget. 330 min session + ~12 min
provisioning + upload leaves comfortable headroom under GitHub's 355 min
hard cut. Without the headroom a session could be killed mid-upload and lose
its work.

---

## Three things to know

1. **Scheduled workflows are best-effort.** Under load GitHub delays firings by
   5-15+ minutes or drops them entirely. Queued-run chaining absorbs most of it,
   but occasional longer gaps happen.
2. **60-day auto-disable.** On public repos GitHub silently disables scheduled
   workflows after 60 days with no repository activity. Push any commit to reset.
3. **Terms of service.** 24×7 on GitHub-hosted runners is the exact use GitHub's
   Additional Product Terms prohibit — hosted runners are for activity related to
   "the production, testing, deployment, or publication of the software project".
   Possible outcomes: terminated jobs, Actions restrictions, repo disabling,
   account suspension. This is your call to make; the STOP workflow is the off-ramp.

**Never flip this repo private while the cron runs** — 24×7 Windows at the Free
plan's 2,000 min/month would cost roughly **$438/month**.

---

## What is still impossible here

| Want | Why not |
|---|---|
| Zero-gap 24×7 | 355 min hard job ceiling |
| Persistent files outside `C:\BlazeNXT\workspace` | Each run = fresh VM; only the workspace artifact is restored |
| Docker Desktop | Needs Windows 10/11 client + nested virtualization |
| Linux containers | Windows Server 2025 runner, Windows containers only |
| 8/16/32/64-core or GPU | Team/Enterprise plan + provisioned larger runners |
| No ToS exposure | Only if it runs on a machine you own/rent, not a hosted runner |
