# Relaycheck Mullvad 📡

Relaycheck Mullvad tells you which Mullvad VPN server to use, and when the connection is bad, where the problem is: your home network, your ISP, your ISP's international links, the route to the region, the city, one hosting provider, or a few individual servers.

It runs on Windows, detects your network on every run (location, ISP, home router, your ISP's first router), and shows the results on a local dashboard with a Run Now button.

By default it checks Mullvad's Seattle servers and uses Los Angeles as the West Coast reference. Any Mullvad city can be tested.

> [!WARNING]
> **Each run turns your VPN off and changes your IP address for about a minute.**
>
> To measure the real route, Relaycheck Mullvad disconnects the laptop's Mullvad app while it pings, then connects to the recommended server for the speed test.
>
> - **Your real IP is exposed:** while it's disconnected (about a minute per run, every 30 minutes with `service.cmd`), your traffic isn't protected and websites see your real IP address.
> - **Your VPN IP changes:** during the speed test your IP becomes the recommended server's. Afterwards the app goes back to the server setting it had before, which can give you a different exit IP than you had before the run.
> - **Open connections may drop:** logged-in sessions, downloads, calls and anything that locks to your IP may disconnect or ask you to sign in again.
>
> Don't run it while you depend on the VPN staying connected: stop the service first. `-NoSpeed` skips the connection to the recommended server, but the VPN is still off while it pings.

## Quick start

1. Double-click `service.cmd`.
2. Your browser opens **http://localhost:8765**. The first check runs straight away and takes about a minute.
3. After that it checks again every 30 minutes.

`service.cmd` starts Relaycheck Mullvad in the background with no window. To stop it, press **Stop Service** on the dashboard. Closing the browser tab doesn't stop it. Double-clicking `service.cmd` while it's running just opens the dashboard again.

## Requirements

- Windows 10 or 11, with Windows PowerShell 5.1 (built in)
- Nothing else to install: the database uses the SQLite library that ships with Windows (`winsqlite3.dll`), and the speed test uses `curl.exe`, which ships with Windows 10 1803 and later.
- The [Mullvad VPN app](https://mullvad.net/download) is optional; see below.

### With or without the Mullvad app

Relaycheck Mullvad gets the server list from Mullvad's public API, not from the app. So most of it works without the app installed:

| Works without the app | Needs the app |
|---|---|
| the ping ladder and diagnosis | the VPN speed test (connecting to the recommended server) |
| the recommended server and backup, with 24h and 7d track records | turning the VPN off before pinging and putting it back afterwards |
| the direct speed test | |
| the dashboard, history and flush | |

The app is found automatically, on your `PATH` or in `C:\Program Files\Mullvad VPN`. If it isn't installed, the two steps that need it are skipped and the VPN row of the speed test shows no result.

**Turn off the app's lockdown mode** ("Lockdown mode" in its VPN settings). It blocks all traffic while the app is disconnected, so the pings can't run: every run is skipped, and the dashboard says why.

**If you use a VPN some other way, turn it off before running.** Relaycheck Mullvad can only turn off the Mullvad app. With a plain WireGuard config, a router tunnel or another provider's VPN, the pings go through that tunnel and describe its route, not your real connection.

- **Mullvad tunnels** (your router's, or a WireGuard config) are detected, because am.i.mullvad.net reports a Mullvad exit. The page shows a warning, the speed test is skipped and the run doesn't count towards the track record.
- **Other VPN providers** aren't detected, so the results would be misleading.

## What each run does

1. **Pings a ladder of targets**, 50 pings each, all at once:

   | Level | What's tested |
   |---|---|
   | home network | your router (the default gateway) |
   | your isp | your ISP's first router, 1.1.1.1 and 8.8.8.8 (both answer from near you) |
   | isp international | 2 Mullvad servers at the nearest location outside your country (more than 150 km away) |
   | us-west-coast | every Mullvad server in the reference city (Los Angeles) |
   | seattle | every Mullvad server in the city under test |
   | provider | the city's servers grouped by hosting company |

2. **Gives each level a state** from how many of its targets are lossy. A target is lossy at 4% loss or more (2 or more of 50 pings lost), or when it gives no reply. It's clean at 2% loss or less (at most 1 of 50 pings lost).

   | State | Share of the level's targets that are lossy |
   |---|---|
   | clean | none |
   | minor | up to 25% |
   | degraded | more than 25%, less than 60% |
   | problem | 60% or more |

   The headline is the first level from the top that is a problem, else the first that is degraded, else any lossy servers on their own (minor). The reference level (us-west-coast) is only named when the city under test is at least as bad: it says the route is the problem, not Mullvad. Lossy reference servers on their own don't change the headline. A provider is only named when 2 or more of its servers are lossy and another provider is healthier.

   Two kinds of router aren't counted. One that never answers pings is shown as "ignores ping". One that loses pings while 1.1.1.1 and 8.8.8.8 are clean is shown as "limits pings to itself": many routers drop some pings aimed at themselves but pass traffic fine, and the clean targets beyond it show the traffic is fine.

3. **Recommends a server:** one that's clean now and has the best track record over the last 24 hours, plus a backup from a different provider. If no server is clean, it's the one with the best track record. Only servers the Mullvad app's provider and ownership settings allow are recommended, so the speed test can connect to them.

4. **Runs a speed test** on speedtest.net: first direct, then through the recommended server. It measures download and upload. For Seattle and Los Angeles it uses the same speedtest.net server each time; for other cities, the first one that answers.

5. **Saves the results** to `relaycheck-mullvad.db` and rebuilds `relaycheck-mullvad.html`.

## Dashboard

- **Recommended Server:** the server to use and a backup, with loss, latency and 24h track record.
- **Where's The Problem:** the headline and the full ladder. Each level shows what was tested and what its state means.
- **Speed To Seattle:** download and upload, direct and through the VPN.
- **Seattle Servers and Los Angeles Servers:** every server's loss, latency, jitter and how often it was clean over 24 hours and 7 days.
- **Recent Runs:** the last 24 runs, and a **Flush Older Than 7 Days** button.

Next to Run Now, the dashboard shows when the next automatic run is, and when the last run failed and why.

## Running it by hand

```
powershell -ExecutionPolicy Bypass -File relaycheck-mullvad.ps1                          one run
powershell -ExecutionPolicy Bypass -File relaycheck-mullvad.ps1 -NoSpeed                 skip the speed test
powershell -ExecutionPolicy Bypass -File relaycheck-mullvad.ps1 -Loop 30                 run, wait 30 min, repeat
powershell -ExecutionPolicy Bypass -File relaycheck-mullvad.ps1 -Serve -Loop 30          dashboard, run every 30 min (what service.cmd does)
powershell -ExecutionPolicy Bypass -File relaycheck-mullvad.ps1 -City lax -RefCity sea   test another Mullvad city
```

| Option | Default | Meaning |
|---|---|---|
| `-City` | `sea` | Mullvad city code to test |
| `-RefCity` | `lax` | Mullvad city code used as the regional reference |
| `-RegionName` | `us-west-coast` | label for the reference level |
| `-Pings` | `50` | pings per target (1 to 1000) |
| `-Loop` | `0` | minutes between runs (0 = run once) |
| `-Serve` | off | start the dashboard on localhost; without `-Loop` it only runs on Run Now (and once at the start if there's no report yet) |
| `-Port` | `8765` | dashboard port |
| `-NoSpeed` | off | skip the speed test |

The other settings are at the top of `relaycheck-mullvad.ps1`: the clean and lossy thresholds, state shares, history windows, the international check's distance, the slow-speed threshold for the speed notes (10 mbps), the number of recent runs shown and the preferred speedtest.net servers.

## Files

| File | What it is |
|---|---|
| `service.cmd` | starts the dashboard in the background |
| `relaycheck-mullvad.ps1` | the whole program |
| `relaycheck-mullvad.db` | results (SQLite), created on the first run |
| `relaycheck-mullvad.html` | the latest report, rebuilt on every run |
| `relaycheck-mullvad.db-wal`, `relaycheck-mullvad.db-shm` | SQLite's working files next to the database |
| `relaycheck-mullvad.html.tmp` | the new report while it's being written; it then replaces the old one |

The database and report files hold your public IP, ISP and location, so they're all excluded from git.

### Database

Open `relaycheck-mullvad.db` with [DB Browser for SQLite](https://sqlitebrowser.org).

- `runs`: one row per run. Time and duration (`duration_s`), where you were (`ip`, `country`, `city`, `isp`), whether a Mullvad tunnel outside the app carried it (`router_vpn`), the diagnosis (`level`, `severity`, `problem`, `diagnosis`), the recommended server and backup (`pick`, `pick_why`, `backup`) and speed results (`speed_server`, `direct_down`, `direct_up`, `vpn_down`, `vpn_up`, `speed_note`).
- `checks`: one row per target per run: `run_id`, the level (`grp`), `name`, `provider`, `ip`, `loss`, latency (`ms`) and `jitter`.

## Dashboard security

The dashboard only listens on `127.0.0.1`. Run Now, Stop Service and Flush need a custom request header that other websites can't send without a CORS preflight, which the server never answers. Requests whose `Host` header isn't `localhost` or `127.0.0.1` on the dashboard's port are refused, which blocks DNS rebinding.
