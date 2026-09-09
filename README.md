# Home Media Stack (Raspberry Pi)

Jellyfin + Unmanic + Tailscale, designed so the Pi **almost never has to transcode**.

- **Jellyfin** – serves your library
- **Unmanic** – automatically re-encodes new files to a universally-compatible
  format (H.264 / AAC-stereo / MP4) so clients **direct play**
- **Tailscale** – secure remote access through Hyperoptic's CGNAT (HTTPS on your
  tailnet, no port forwarding, no public IP needed)

```
 drop raw file ->  /mnt/storage/staging/Movies/foo.mkv
                        |
                    Unmanic  (re-encode: H.264 / AAC 2.0 / MP4)
                        |
                    move  ->  ./media/Movies/foo.mp4  ->  Jellyfin (direct play)
                        |
                    Tailscale  ->  https://aurora-jellyfin.tail872d42.ts.net

Jellyfin never sees a file until Unmanic has finished processing it.
```

---

## 1. Folder layout

```
nginxmanager/
├─ compose.yaml
├─ .env                 # your Tailscale key (gitignored)
├─ .env.example
├─ Tailscale/serve.json # exposes Jellyfin (:443) + Unmanic UI (:8443) over the tailnet
├─ /mnt/storage/staging/ # DROP raw files here (mirrors media subfolders)
│  ├─ Movies/ TVShows/ Music/ Photos/ Holidays/ Memories/
└─ media/               # FINISHED library Jellyfin serves — Unmanic writes here
   ├─ Movies/
   ├─ TVShows/
   ├─ Music/
   ├─ Photos/
   ├─ Holidays/
   └─ Memories/
```

You add files to `/mnt/storage/staging/<category>/`; Docker exposes that host
directory to Unmanic as `/staging`. Unmanic processes the files and moves the
result to the matching local `media/<category>/` directory, exposed to the
container as `/media`.
`appdata/` and `cache/` are created automatically on first run.

The repository's local `staging/` directory is not used by this compose file.
The input directory is the host path `/mnt/storage/staging` (or whatever host
path is configured on the left side of the `/staging` volume mapping).

Unmanic runs as UID/GID `1000:1000`. The output directory and its category
directories must therefore be writable by GID `1000`; otherwise processing can
complete but Mover v2 will fail with `Permission denied` and no file will reach
`media/`. On the Pi, prepare the directories with:

```bash
sudo mkdir -p media/{Movies,TVShows,Music,Photos,Holidays,Memories}
sudo chown -R root:1000 media
sudo chmod -R g+rwX media
```

> **Using an external drive?** Change the host-side paths in `compose.yaml`.
> For example, use `/mnt/usbdrive/staging:/staging` for the Unmanic input and
> `/mnt/usbdrive/media:/media` for the finished library. Keep the Jellyfin
> mount read-only: `/mnt/usbdrive/media:/media:ro`.

---

## 2. First run (on the Pi)

```bash
# Install Docker (if not already)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # then log out/in

# Copy this folder to the Pi, then from inside it:
cp .env.example .env
nano .env                       # paste your Tailscale auth key

docker compose up -d
docker compose ps               # all three should be "running"
```

Get your Tailscale name:
```bash
docker compose exec tailscale tailscale status
```

---

## 3. Jellyfin setup

1. Browse to `http://<pi-ip>:8096`, complete the wizard.
2. Add libraries pointing at the mounted paths:
   - Movies → `/media/Movies`
   - Shows  → `/media/TVShows`
   - Music  → `/media/Music`, etc.
3. Dashboard → Playback: leave hardware acceleration **off** (not needed once
   Unmanic standardises files; the Pi's software path is only a rare fallback).

---

## 4. Unmanic — "lowest transcode probability" config

Goal: every file ends up as **MP4 / H.264 8-bit / AAC 2.0**, which direct-plays
on virtually every client (browsers, phones, TVs, Jellyfin apps).

Open Unmanic at `http://<pi-ip>:8888`, then **Settings → Libraries** and set the
library **Path = `/staging`**. Enable the **file monitor** (process new files),
the library scanner, and **Run a one off library scan on startup**. Set the
scanner schedule to a short interval such as **5 minutes** rather than the
default 1440 minutes (once per day).

Use `http://localhost:8888` only when the browser is running on the Pi itself.
From another computer on the LAN, use the Pi's address, for example
`http://192.168.1.132:8888`. Over Tailscale, use the HTTPS URL documented below.

To scan immediately, open the library's actions or three-dot menu and choose
**Rescan library now**. The scan should place discovered files in the queue;
watch **Dashboard** for scan progress and processing status.

The extension filter must have **Add all matching files to pending tasks list**
enabled. This is needed when the input is already H.264/AAC/MP4: otherwise
Unmanic can correctly decide that the file needs no codec work and skip the
task entirely, which also means Mover v2 never gets a chance to move it.

### Install these plugins (Plugins → Manage Plugins)

Add them in this flow order. These are the plugin names used by the current
Unmanic plugin catalog.

**A. Library file test**
- **Limit Library Search by File Extension** — include the media extensions
  you use, such as `mp4`, `mkv`, `avi`, `mov`, and `webm`.

**B. Worker — video**
- **Transcode Video Files** — settings:
  - Encoder: `libx264` (software — best quality/compatibility on a Pi)
  - Constant quality (CRF): **`22`** (20–23 is the practical compatibility range)
  - Preset: **`Very fast`** — suitable for background processing on the Pi
  - Profile: **Auto** — this is the only profile exposed by the installed plugin
  - The installed plugin does not expose a pixel-format control; the sample
    output is `yuv420p`, but verify 10-bit/HDR sources separately if needed.

**C. Worker — audio**
- **Add Extra Stereo Audio** (a.k.a. stereo clone) — configure the source
  language as **English**, leave source channel count and codec blank, use the
  native AAC encoder, keep the original multichannel stream, make the new
  stereo stream the default, and move it to the first audio stream. This adds
  an **AAC stereo (2.0)** track when a matching multichannel English track is
  present. Files without an English language tag will not match this setting.

**D. Worker — container**
- **Remux Video Files** — set the container to `MP4`. MP4 is the safest streaming
  container. (Text subtitles become `mov_text`; image subs like PGS can't live
  in MP4 and will be dropped — extract them to external `.srt` first if you need them.)

**E. Post-processor — MOVE to the media library (this is the staging step)**
- Install **Mover v2** and configure:
  - **Destination:** `/media`
  - **Recreate directory structure:** ON — so `staging/Movies/foo`
    lands in `media/Movies/foo`, `staging/TVShows/...` in `media/TVShows/...`, etc.
  - **Also include library path:** OFF — do not create a `/media/staging/...`
    path.
  - **Remove source files:** ON — this completes the move after a successful task.
- This is what makes staging strict: the finished file is *moved out* of
  `/staging` into `/media`, so Jellyfin only ever sees completed files.

> If a suitable move plugin isn't available, the fallback is to make the library
> in-place (Path = `/media`, drop files straight into `media/<category>`), losing
> the strict staging separation but keeping the same transcode result.

Save. Drop a file into `/mnt/storage/staging/Movies/` and watch **Dashboard** —
it should queue, process, and move to `media/Movies/`. Even a compliant MP4
must be queued by the extension filter for Mover v2 to relocate it. Confirm
that the source disappears only after the destination file exists. Play it in
Jellyfin: the stream info should say **"Direct playing"**, and the Pi's CPU
should stay near idle.

---

## 5. Remote access (Tailscale)

- Install the Tailscale app on your phone/laptop/TV, sign into the **same account**.
- **Jellyfin:** `https://aurora-jellyfin.tail872d42.ts.net`
- **Unmanic UI:** `https://aurora-jellyfin.tail872d42.ts.net:8443`
- Set Jellyfin → Dashboard → Networking, and the compose
  `JELLYFIN_PublishedServerUrl`, to the Jellyfin URL above so apps auto-configure.
- To let someone without the app in, use **Tailscale Funnel** later (public URL,
  no install) — but keep an eye on streaming volume.

> **Important — how to reach services over Tailscale.** The `tailscale` container
> runs in *userspace* mode, so its tailnet IP (`100.x`) does **not** expose the
> raw `:8096` / `:8888` ports. Only what `serve.json` proxies is reachable, via
> the **HTTPS URLs above** (`:443` Jellyfin, `:8443` Unmanic). The raw
> `http://<host>:8096` URLs work **only on the LAN**, directly to the machine
> running Docker.

---

## 6. Things that can *still* transcode (and the fix)

Unmanic removes the **codec/audio/container** triggers. Two remain, unrelated to
the GPU:

1. **Client quality cap** — if the Jellyfin app is set to e.g. "4 Mbps", it
   downscales regardless. Set clients to **Auto/max** on good connections.
2. **Remote bandwidth** — streaming out is limited by your **home upload speed**.
   Hyperoptic is **symmetric fibre**, so high-bitrate direct play works remotely.

Burned-in **image subtitles** also force a video transcode — prefer external
`.srt` / soft text subs.

---

## 7. Pi performance reality

Software libx264 on a Pi is **slow** (Pi 4 ≈ 0.5–1× realtime for 1080p; Pi 5 is
noticeably faster). That's fine — Unmanic is a background batch, not live. A
full initial library conversion may run for days; set `preset faster` and let it
churn. **Storage cost:** H.264 files are larger than HEVC, so budget disk space.
```
