# Home Media Stack (Raspberry Pi)

Jellyfin + Unmanic + Tailscale, designed so the Pi **almost never has to transcode**.

- **Jellyfin** – serves your library
- **Unmanic** – automatically re-encodes new files to a universally-compatible
  format (H.264 / AAC-stereo / MP4) so clients **direct play**
- **Tailscale** – secure remote access through Hyperoptic's CGNAT (HTTPS on your
  tailnet, no port forwarding, no public IP needed)

```
 drop raw file ->  ./staging/Movies/foo.mkv
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
├─ staging/             # DROP raw files here (mirrors media subfolders)
│  ├─ Movies/ TVShows/ Music/ Photos/ Holidays/ Memories/
└─ media/               # FINISHED library Jellyfin serves — Unmanic writes here
   ├─ Movies/
   ├─ TVShows/
   ├─ Music/
   ├─ Photos/
   ├─ Holidays/
   └─ Memories/
```

You add files to `staging/<category>/`; Unmanic processes them and moves the
result to the matching `media/<category>/`.
`appdata/` and `cache/` are created automatically on first run.

> **Using an external drive?** Point the library at it by changing `./media`
> in `compose.yaml` to an absolute path, e.g. `/mnt/usbdrive/media:/media:ro`
> (Jellyfin) and `/mnt/usbdrive/media:/library` (Unmanic).

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

Open `http://<pi-ip>:8888`, then **Settings → Libraries** and set the library
**Path = `/staging`**. Enable both the **file monitor** (process on new files)
and a periodic **scanner**.

### Install these plugins (Plugins → Manage Plugins)

Add them in this flow order. Exact names may vary slightly in the browser —
match by function.

**A. File-test (skip work that isn't needed)**
- **"Ignore files already meeting target"** / *Ignore by codec* — skip files
  that are already H.264 + AAC + MP4 so nothing is re-processed forever.

**B. Worker — video**
- **"Video Encoder H264 (libx264)"** — settings:
  - Encoder: `libx264` (software — best quality/compatibility on a Pi)
  - Constant quality (CRF): **`21`** (lower = better/bigger; 20–23 is the sweet spot)
  - Preset: **`faster`** or `veryfast` (Pi 4) / `medium` (Pi 5) — trades speed for size
  - Profile: **High**, Level **4.1**
  - Pixel format: **`yuv420p`** (forces 8-bit — kills the 10-bit/HDR transcode trigger)

**C. Worker — audio**
- **"Add extra audio stream"** (a.k.a. stereo clone) — add an **AAC stereo (2.0)**
  track if one isn't present. This is the #1 sneaky transcode cause: surround-only
  (DTS/TrueHD) files transcode even when the video is fine. Keep the original
  surround track too if you like; just guarantee an AAC 2.0 exists.

**D. Worker — container**
- **"Remux to MP4"** / *Force MP4 container*. MP4 is the safest streaming
  container. (Text subtitles become `mov_text`; image subs like PGS can't live
  in MP4 and will be dropped — extract them to external `.srt` first if you need them.)

**E. Post-processor — MOVE to the media library (this is the staging step)**
- Search the plugin browser for **"move"** and install a post-processor movement
  plugin (e.g. *"Postprocessor - move file to a new location"*). Configure:
  - **Destination:** `/media`
  - **Preserve source subdirectory structure:** ON — so `staging/Movies/foo`
    lands in `media/Movies/foo`, `staging/TVShows/...` in `media/TVShows/...`, etc.
- This is what makes staging strict: the finished file is *moved out* of
  `/staging` into `/media`, so Jellyfin only ever sees completed files.

> If a suitable move plugin isn't available, the fallback is to make the library
> in-place (Path = `/media`, drop files straight into `media/<category>`), losing
> the strict staging separation but keeping the same transcode result.

Save. Drop a non-compliant file into `staging/Movies/` and watch **Dashboard** —
it should queue, encode, and move to `media/Movies/`. Play it in Jellyfin: the
stream info should say **"Direct playing"**, and the Pi's CPU should stay near idle.

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
