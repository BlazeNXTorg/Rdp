# BlazeNXTOrg RDP — Cloud Workstation

On-demand Windows workstation you connect to from **your phone** over Tailscale.
No local PC required: the GitHub Actions runner *is* the machine.

> **This is a public repository.** That is deliberate: on a public repo, standard
> GitHub-hosted runners get **4 vCPU / 16 GB** and **unlimited free minutes**.
> The same workflow on a private repo gets 2 vCPU / 8 GB and burns the free
> 2,000 min/month quota. No secret value is ever visible to the public —
> but **run logs, run inputs and the job summary are public**, which is why the
> password lives in a secret and never in an input.

---

## What one run gives you

| | |
|---|---|
| Machine | Windows Server 2025, **4 vCPU / 16 GB / 14 GB SSD** |
| Session | up to **355 minutes** (GitHub hard-kills at 6 h) |
| Network | Tailscale ephemeral node `blazenxt-ws-<run_id>` + MagicDNS |
| RDP | port 3389, NLA relaxed so thin clients / mobile work |
| Extras | audio redirection, clipboard + drive sharing, High-Performance power plan |
| Software | Chrome, VS Code, Git, 7-Zip, Python 3.11, Node LTS (optional) |
| Docker | **Docker Engine, Windows containers only** + a keep-alive container (optional) |

---

## Runs are billed to the org, not to you

Because this repo is public, standard runners here are **free and unlimited**.
No card required, no quota consumed.

---

## Setup (already done, listed for reference)

Repository secret — `Settings → Secrets and variables → Actions`:

| Secret | Purpose |
|---|---|
| `TAILSCALE_AUTH_KEY` | Ephemeral, reusable auth key from the Tailscale admin console |
| `RDP_PASSWORD` | The Windows password for the workstation account (min 8 chars) |

Both are required. If `RDP_PASSWORD` is missing the run **fails fast** with
instructions rather than printing a password into a public log.

---

## Run it

1. `Actions` tab → **BlazeNXT Workstation Cloud RDP** → `Run workflow`.
2. Pick options (defaults are fine) → **Run workflow**.
3. Wait ~8–12 minutes for the software bundle and Docker to finish.
4. Read the run's **Summary** page for your connection details.

### Connect from your phone (Android / iOS)

1. Install **Tailscale** ([Android](https://play.google.com/store/apps/details?id=com.tailscale.ipn) / [iOS](https://apps.apple.com/app/tailscale/id1470499037)) and sign in to the **same tailnet**.
2. Install **Remote Desktop** by Microsoft ([Android](https://play.google.com/store/apps/details?id=com.microsoft.rdc.androidx) / [iOS](https://apps.apple.com/app/microsoft-remote-desktop/id714464092)).
3. Add a PC:
   - **Address:** `blazenxt-ws-<run_id>` (MagicDNS) or the `100.x.y.z` Tailscale IP
   - **Username:** `BlazeAdmin` (or the custom username you chose)
   - **Password:** the value of the `RDP_PASSWORD` secret
4. Accept the certificate warning, connect, and you're on a full Windows desktop.

---

## Hard limits — read these once

- **6-hour ceiling.** GitHub kills every job at 360 min; this workflow stops itself at 355.
  The VM and everything on it are destroyed at the end of the run. Nothing persists.
- **24×7 is not possible here.** Always-on requires a machine that isn't an
  ephemeral GitHub runner. Each session is a fresh machine.
- **Docker Desktop cannot be installed.** It requires Windows 10/11 client;
  this is Windows Server 2025. Both of its backends (WSL2, Hyper-V) need nested
  virtualization, which GitHub-hosted Windows runners do not have.
  What works: **Docker Engine with Windows containers** (`mcr.microsoft.com/...` images).
  Linux containers are not available on this machine.
- **No GPU, no larger runners.** GPU and 8/16/32/64-core runners require a
  Team or Enterprise Cloud plan, are billed per minute with no free allowance,
  and must be provisioned as runner groups first. On the Free plan,
  `windows-latest` **is** the maximum.
- **Terms of service.** GitHub's Additional Product Terms say GitHub-hosted
  runners may not be used for activity "unrelated to the production, testing,
  deployment, or publication of the software project associated with the
  repository". Occasional dev/testing use of a workstation is one thing; a
  permanent always-on box is explicitly the kind of use they act on
  (job termination, Actions restrictions, account suspension).
  Keep this tied to real work in this repo.

---

## Automation: the 6-hourly cron

`on.schedule: cron '0 */6 * * *'` (UTC) starts a fresh session every 6 hours:
**05:30 / 11:30 / 17:30 / 23:30 IST**.

The chain works because sessions self-stop at **350 min**, leaving slack before the
next cron fires. When a cron fires while a session is still live, `concurrency`
holds it pending rather than double-starting — it takes over the moment the old
machine dies. Expect a **~10-20 min gap every 6 hours**; a session cannot be truly
continuous, since 355 min is the hard GitHub ceiling.

**Scheduled runs use one stable node name: `blazenxt-ws`.**
Manual runs are `blazenxt-ws-<run_id>`. Add the PC to your phone once using
`blazenxt-ws` and it keeps working across every automated session, because the
password comes from the `RDP_PASSWORD` secret and never changes.

### Stopping it

Run **BlazeNXT Workstation STOP** (Actions -> Run workflow -> type `STOP`).
It force-cancels every live session and disables the cron. Force-cancel matters:
the keep-alive loop ignores a normal cancel.

To bring it back: **Enable workflow** on *BlazeNXT Workstation Cloud RDP*,
then press **Run workflow** or wait for the next cron tick.

### Cron caveats

- Scheduled workflows are **best-effort**: under load GitHub delays them by
  5-15+ min or drops individual firings. The pending-queue design absorbs most of it.
- On a **public** repo GitHub auto-disables scheduled workflows after
  **60 days of no repository activity**. Push anything to keep it alive.
- This runs 24x7 on GitHub-hosted runners. GitHub's Additional Product Terms
  prohibit using hosted runners for activity unrelated to the software project
  in the repo. The exposure is real: job termination, Actions restriction,
  repo disabling, account suspension. The STOP workflow is your off-ramp.
- **Never flip this repo back to private while the cron is on.** Private repos
  burn the Free plan's 2,000 min/month; 24x7 Windows = ~43,800 min, roughly
  **$438/month**. Public repos are free and unlimited.

## Upgrade path

| Want | Needs |
|---|---|
| More cores (8/16/32/64) | Org upgrade to **Team** (~$4/user/mo) + provision larger runners, then they bill per minute |
| GPU | Team/Enterprise + GPU larger runner (billed per minute) |
| True 24×7 | A real VM somewhere (cloud or a box at home) — self-hosted runner or plain RDP |
| Linux containers | A Linux host, not a Windows runner |
| Cheaper than cloud VM | Linux VPS (~$5–15/mo) with a browser-based remote desktop |
