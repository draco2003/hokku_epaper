# Upgrading a Debian Hokku server from v4.0.0-beta2 to v4.0.0-beta3

This guide covers upgrading a running Hokku server from **v4.0.0-beta2**
(commit [`74a888e`][c-b2]) to **v4.0.0-beta3** (commit [`0869a60`][c-b3]). It
also covers moving the server onto the new beta3 settings and re-rendering
every photo.

Short version (Debian `.deb` install):

```sh
sudo systemctl stop hokku-server
sudo tar -C /var/lib -czf ~/hokku-beta2-backup.tgz hokku          # config + photos + DB + cache
sudo apt install ./hokku-server_4.0.0.beta3-1_all.deb             # restarts the service
# Web UI -> Config: set each of the 3 pipelines to its "(default)" preset
#   (or at least set Autocontrast = Off), Save
# Web UI -> Admin: "Clear caches and reconvert"
```

> **Verified end to end** on a Debian 12 (bookworm) systemd host using the
> official release `.deb` files: install beta2, upload photos, upgrade to beta3,
> migrate settings, regenerate everything, fetch a panel as a frame, and roll
> back to beta2 byte-for-byte. See [§12 Verification log](#12-verification-log-and-deviations-from-the-docs).
> Two observed behaviours **contradict the beta3 release notes**. Read
> [§12.2](#122-deviations-from-the-documentation) before you plan downtime.

Links point at the upstream tags on `defl/hokku_epaper`. `B3:` means
`https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/` and `B2:` means the
same URL at `v4.0.0-beta2`.

---

## 1. What changes in beta3 (summary)

| Area | beta2 | beta3 | Source |
|---|---|---|---|
| App version | `4.0.0b2` / deb `4.0.0~beta2-1` | `4.0.0b3` / deb `4.0.0~beta3-1` | [pyproject.toml][b3-pyproject], [debian/changelog][b3-deb-changelog] |
| `config.json` schema `version` | 9 | 11 (two automatic migrations: v9→v10, v10→v11) | [app_config.py L32][b3-appcfg-ver], [L102][b3-mig910], [L134][b3-mig1011] |
| New setting | – | `prepare_autocontrast` in each of the 3 pipelines (`per_channel` / `preserve_tone` / `off`) | [image_config.py L28/L40][b3-imgcfg-ac] |
| Shipped pipeline defaults | Floyd–Steinberg, CIELAB, colour boosts on | Atkinson, OKLAB, boosts off, autocontrast off | [config.json.example][b3-example] vs [beta2][b2-example] |
| Image DB (`image_manager.json`) | schema v4 | schema v4, plus optional per-picture override fields | [image_manager_abstract.py L55-56][b3-db-ver], [image_record.py L61-62][b3-rec] |
| Per-picture settings | – | New: UI + `GET/PATCH /hokku/api/image/<name>/config` | [flask_app.py L673/L704][b3-img-cfg-api], [manual §Conversion for this picture][b3-manual-perpic] |
| Debian packaging (`control`, `postinst`, service unit) | – | **unchanged** (`git diff v4.0.0-beta2 v4.0.0-beta3 -- python/debian/` only touches `changelog`) | [debian/][b3-debian] |
| Python deps | – | no runtime change. `waitress` was already a runtime dependency in beta2 and is now also listed in `requirements.txt`. New tool-only deps: `scipy scikit-learn matplotlib pandas` | [requirements.txt][b3-req], [pyproject.toml][b3-pyproject] |
| Bundled firmware | huessen 1.2.21, bigme_f7 1.2.10, seeed 1.2.5 | huessen **1.2.25**, bigme_f7 **1.2.12**, seeed 1.2.5 | [beta3 release notes][rel-b3] ("Versions"), [firmware/huessen_epf1301/VERSION][b3-fw-ver] |
| Rendering chain | – | measured panel anchors, bounded S-curve tone map, measured gamut-correction LUT | [CHANGELOG beta3 "Changed"][b3-changelog], [release notes][rel-b3] |

---

## 2. Supported install paths (and which parts of this guide apply)

The docs describe three ways to run the server ([README][b3-readme],
[docs/install.md][b3-install]):

| Install path | How it runs | Upgrade mechanism | Sections |
|---|---|---|---|
| **A. Debian package** (`hokku-server_*.deb`) on Debian, Ubuntu, or Raspberry Pi OS | systemd unit `hokku-server.service`, user `hokku`, `ExecStart=/usr/bin/hokku-server /var/lib/hokku/config.json` ([service L22/L39][b3-service]) | `apt install ./hokku-server_4.0.0.beta3-1_all.deb` | all |
| **B. From source / pip** (`.venv` + `pip install -r requirements.txt`, `cd python && python -m hokku.webserver`), or the pip wheel | whatever you start it with (no unit file ships) | `git checkout v4.0.0-beta3` + `pip install -r requirements.txt`, or `pip install --upgrade hokku_server-4.0.0b3-py3-none-any.whl` | §4.B, then §5 onward |
| **C. Raspberry Pi appliance image** (`image_*-hokku.img.xz`) | the same `.deb` + `hokku-installer` `.deb`, pre-installed by pi-gen ([os/pi/stage-hokku/00-install/00-run.sh][b3-pigen]) | the docs describe no in-place upgrade. Installing the new `.deb` (path A) is the practical route. See §4.C | §4.C |

The docs do not describe a Docker install path. `test_server/` is a
development harness, not a supported deployment.

---

## 3. Prerequisites and backup

### 3.1 Prerequisites

- Shell access with `sudo`.
- The beta3 assets from the [v4.0.0-beta3 release][rel-b3]:
  - `hokku-server_4.0.0.beta3-1_all.deb` (path A/C)
  - `hokku_server-4.0.0b3-py3-none-any.whl` (path B, wheel variant)
  - `hokku-installer_4.0.0.beta3-1_all.deb` (appliance only, optional)
- Keep the beta2 `.deb` too, for rollback: [v4.0.0-beta2 release][rel-b2].
- Internet access, **or** Debian packages that already satisfy `postinst`'s
  pip checks. `postinst` pip-installs `pillow-heif`, `opencv-python-headless`,
  `numba`, and others when the installed versions are missing or too old
  ([postinst][b3-postinst]). On an upgrade from beta2 they are already present.
  In testing, no downloads were needed.

```sh
cd ~
curl -LO https://github.com/defl/hokku_epaper/releases/download/v4.0.0-beta3/hokku-server_4.0.0.beta3-1_all.deb
curl -LO https://github.com/defl/hokku_epaper/releases/download/v4.0.0-beta2/hokku-server_4.0.0.beta2-1_all.deb   # rollback copy
# or: gh release download v4.0.0-beta3 --repo defl/hokku_epaper -p 'hokku-server_*.deb'
```

### 3.2 What lives where (path A defaults)

| Data | Path | Source |
|---|---|---|
| Settings (JSON, has a top-level `"version"`) | `/var/lib/hokku/config.json` | [install.md L68][b3-install-cfg], [service L21-22][b3-service] |
| Original photos | `upload_dir`, default `/var/lib/hokku/images/` | [install.md L56][b3-install-deb], [app_config.py][b3-appcfg-fields] |
| Image database | `<cache_dir>/image_manager.json` | [image_manager_abstract.py L55][b3-db-ver] |
| B&W/face classifier cache | `<cache_dir>/image_classifier.json` | [CHANGELOG 3.0 beta3–4][b3-changelog-classifier] |
| Rendered panels / previews / thumbnails | `<cache_dir>/images/*_panel.bin.zst`, `*_preview.png`, thumbnails | observed (§12) |
| Downloaded firmware library + per-model pin | `firmware_dir`, default `/var/lib/hokku/firmware` (`selection.json`) | [manual L138-148][b3-manual-fw] |
| Numba JIT cache | `/var/lib/hokku/numba_cache` | [service L42][b3-service] |
| Bundled firmware (read-only, from the package) | `/usr/share/hokku-server/firmware/` | observed with `dpkg -L hokku-server` |

If you moved `upload_dir` or `cache_dir` to another disk, back those paths up
too. Check with `jq '.upload_dir, .cache_dir, .firmware_dir' /var/lib/hokku/config.json`.

### 3.3 Backup

```sh
sudo systemctl stop hokku-server            # quiesce DB writes
TS=$(date +%Y%m%d-%H%M%S)
sudo mkdir -p /root/hokku-backup-$TS
sudo tar -C /var/lib -czf /root/hokku-backup-$TS/var-lib-hokku.tgz hokku
dpkg-query -W hokku-server | sudo tee /root/hokku-backup-$TS/pkg-version.txt   # expect: hokku-server	4.0.0~beta2-1
sudo cp /var/lib/hokku/config.json /root/hokku-backup-$TS/config.v9.json
```

The tarball holds everything: config, originals, `image_manager.json`,
`image_classifier.json`, rendered cache, and the firmware library. **Keep it.**
It is the only clean rollback. A beta2 server started on a beta3-migrated
config does not reproduce the beta2 renders (§11).

---

## 4. Install beta3

### 4.A Debian package (the common case)

```sh
sudo apt install ./hokku-server_4.0.0.beta3-1_all.deb
```

- apt reports `1 upgraded` and `Unpacking hokku-server (4.0.0~beta3-1) over (4.0.0~beta2-1)`.
- `postinst` stops the service first ([postinst L8][b3-postinst-stop]), re-checks
  the pip-only dependencies, chowns `/var/lib/hokku`, and debhelper **starts the
  service again automatically**. You do not need a separate `systemctl start`.
  Stopping it in §3.3 was only for a consistent backup.
- `/var/lib/hokku/config.json` is **not** a dpkg conffile. It is only seeded when
  missing ([service L21][b3-service]), so the package never overwrites your
  settings.
- **Dependency update:** none needed beyond what `postinst` does. The Debian
  `Depends:` list and the pip list in `postinst` are identical between the two
  tags ([python/debian/control][b3-control], [postinst][b3-postinst]).

### 4.B From source (venv) or pip wheel

Source checkout ([install.md L89-94][b3-install-src]):

```sh
sudo systemctl stop <your-unit>   # or Ctrl-C whatever runs `python -m hokku.webserver`
cp /path/to/config.json /path/to/config.json.beta2.bak     # plus upload_dir / cache_dir tarball, as in §3.3
cd hokku_epaper
git fetch --tags https://github.com/defl/hokku_epaper.git
git checkout v4.0.0-beta3
source .venv/bin/activate
pip install -r requirements.txt   # adds waitress (already a runtime dep) + tool-only scipy/scikit-learn/matplotlib/pandas
cd python && python -m hokku.webserver /path/to/config.json
```

Wheel ([beta2 release notes "Packaging"][rel-b2]):

```sh
.venv/bin/pip install --upgrade ./hokku_server-4.0.0b3-py3-none-any.whl
.venv/bin/hokku-server /path/to/config.json
```

The wheel bundles **no firmware** ([beta2 release notes][rel-b2]). The log line
reads `Bundled firmware: none found`, so OTA and flash-a-screen need the
`.deb` or the appliance. Tested: a venv with the beta2 wheel upgraded to the
beta3 wheel, then served a v9 config fine (§12).

### 4.C Raspberry Pi appliance

The appliance image is Raspberry Pi OS with the same `hokku-server` and
`hokku-installer` `.deb`s pre-installed ([00-install/00-run.sh][b3-pigen]).
[docs/appliance.md][b3-appliance] documents reflashing the SD card (fresh
start) and `reset.sh` (back to setup mode), but **no in-place upgrade**. The
practical route is path A over SSH:

```sh
sudo apt install ./hokku-server_4.0.0.beta3-1_all.deb ./hokku-installer_4.0.0.beta3-1_all.deb
```

Between the two tags, `hokku-installer` only changed its version number
(`git diff --stat v4.0.0-beta2 v4.0.0-beta3 -- installer/` shows 2 files:
`debian/changelog` and `pyproject.toml`), so upgrading it is optional. At the
time of writing, the beta3 release has no `.img.xz` asset yet. The release
notes say it is built after publication ([rel-b3][rel-b3] "Versions"). The
appliance path was **not** tested on a Pi (§12.3).

---

## 5. Config migration: what happens automatically

On first start, beta3 walks the migration chain `9 → 10 → 11` in memory
([app_config.py `_migrate` L166][b3-migrate]):

- **v9 → v10** ([L102][b3-mig910]): fills every missing field of
  `image_config_default`, `image_config_bw`, and `image_config_face`, and drops
  unknown keys, so later parsing can be strict. A missing
  `prepare_autocontrast` becomes `"per_channel"`.
- **v10 → v11** ([L134][b3-mig1011]): same guarantee, adds
  `prepare_autocontrast: "per_channel"` where absent. That is exactly what
  beta2 did implicitly. The upgrade must not restyle the library "without being
  asked".

Important consequences, all verified in §12:

1. **Your stored beta2 pipeline tuning is kept verbatim.** That includes values
   you never touched, because beta2 wrote the full pipeline blobs into
   `config.json`. After the upgrade, `image_config_default` still reads
   `floyd_steinberg / cielab / color_enhance 1.25 / adaptive_vivid true /
   prepare_autocontrast per_channel`. **The new beta3 defaults only apply after
   you choose them** (§6).
2. **The file on disk stays at `"version": 9`** until something saves the
   config (UI *Save*, or `POST /hokku/api/config`). The in-memory config, and
   `GET /hokku/api/config`, report `11`. This is expected: `load()` writes back
   only unversioned configs ([app_config.py L341][b3-appcfg-load]).
3. Top-level settings (`refresh_image_at_time`, `port`, `server_threads`,
   `poll_interval_seconds`, `crop_to_fill_threshold`, `auto_clear_cache`,
   `memory_budget_mb`, `mdns_hostname`, `firmware_*`, `flash_wifi_*`,
   `classifier_*`) are unchanged between the beta2 and beta3 example configs.
   `jq -S` diff of [beta2][b2-example] vs [beta3][b3-example]
   `config.json.example`: only the three pipeline blobs and `version` differ.
   There are **no renamed or removed settings**.

---

## 6. New and changed settings, with recommendations

All of these live in the three pipeline objects `image_config_default`,
`image_config_bw`, and `image_config_face` in `config.json`. In the UI they are
under **Config → Dither preset / Custom…**.

"beta2 default" and "beta3 default" are the shipped `config.json.example`
values ([beta2][b2-example], [beta3][b3-example]). They match
`GET /hokku/api/config` → `dither_presets.default_general / default_bw /
default_face` (verified). "Upgraded value" is what an untouched beta2 install
has after the migration.

### 6.1 The new setting

| Setting | Pipelines | beta3 default (fresh install) | Upgraded value | Meaning | Recommended |
|---|---|---|---|---|---|
| `prepare_autocontrast` (UI: *Autocontrast*: Off / Keep colour balance / Per channel) | all 3 | `"off"` | `"per_channel"` | `per_channel` = PIL per-RGB stretch, which acts as an accidental white balance (up to 13 units toward yellow on 30 test photos). `preserve_tone` = one luminance stretch applied to R, G, and B equally. `off` = no stretch ([image_config.py L20-28][b3-imgcfg-ac], [index.html L1595/L2064][b3-ui-ac], [release notes][rel-b3]) | **`off`** (what ships and "rated best on glass"). Use `preserve_tone` if you want the old punch without the cast |

`prepare_autocontrast_cutoff` (0.5) is unchanged. The UI hides it when
autocontrast is `off` ([index.html L2069-2070][b3-ui-ac]).

### 6.2 Changed defaults (fresh installs only; upgrades keep beta2 values)

| Pipeline | Setting | beta2 default | beta3 default | Meaning / reason | Recommended |
|---|---|---|---|---|---|
| General (`image_config_default`) | `dither.algorithm` | `floyd_steinberg` | `atkinson` | error-diffusion kernel, "picked by scoring every combination over a test corpus" ([CHANGELOG][b3-changelog]) | beta3 value |
| General | `drc_l_space` / `drc_chroma_space` | `cielab` | `oklab` | colour space for dynamic-range compression of lightness/chroma | beta3 value |
| General | `color_enhance` | `1.05` | `1` | global saturation multiplier. Boosts off = "single biggest improvement" ([rel-b3][rel-b3]) | beta3 value |
| B&W (`image_config_bw`) | `dither.algorithm` | `floyd_steinberg` | `atkinson` | as above | beta3 value |
| B&W | `drc_l_space` / `drc_chroma_space` | `cielab` | `oklab` | as above | beta3 value |
| B&W | `color_enhance` | `1.25` | `1` | removes the "5 % colour boost" from the B&W preset ([CHANGELOG][b3-changelog]) | beta3 value |
| B&W | `saturate_max_enhance` | `1.25` | `1` | cap for adaptive saturation | beta3 value |
| B&W | `adaptive_saturate_space` | `cielab` | `oklab` | only matters when saturation is boosted. With the caps at 1 it is effectively neutral | beta3 value |
| B&W | `adaptive_vivid` | `true` | `false` | "vivid" chroma boost off | beta3 value |
| Faces (`image_config_face`) | `adaptive_saturate_space` | `cielab` | `off` | no per-stripe saturation on faces | beta3 value |
| Faces | `adaptive_vivid` | `true` | `false` | vivid boost off | beta3 value |
| Faces | `color_enhance` | `1.25` | `1` | boosts off | beta3 value |
| Faces | `saturate_max_enhance` | `1.25` | `1` | boosts off | beta3 value |
| Faces | `clahe_clip_limit` | `1.25` | `1.75` | faces now get the same local contrast as general images ([CHANGELOG][b3-changelog]) | beta3 value |
| Faces | `prepare_usm_amount` / `prepare_usm_radius` | `130` / `1.2` | `120` / `1` | same sharpening as general images | beta3 value |
| Faces | `dither.algorithm` / `dither.serpentine` | `floyd_steinberg` / `false` | `atkinson` / `true` | kernel + scan direction | beta3 value |

The faces pipeline keeps `drc_*_space = cielab` in beta3 (unchanged).

The beta3 preset values match what the tone-mapping and LUT changes were tuned
against. A beta2-tuned pipeline on beta3's new anchors and S-curve is an
untested combination. So the recommendation is **"use the beta3 default preset
for all three pipelines unless you deliberately tuned one"**.

### 6.3 How to apply

**UI (recommended):** open `http://<server>:8080/hokku/ui` → **Config**. For
each of the three pipeline slots, pick **General (default)**,
**Black & white (default)**, and **Faces (default)** in the *Dither preset*
dropdown (or **Reset to preset**), then **Save**. If you have your own tuning
you want to keep, open *Custom…* and change only *Autocontrast* to *Off*
([manual L111-117][b3-manual-preset]).

**API equivalent** (tested). `POST /hokku/api/config` shallow-merges top-level
keys, so send the whole pipeline objects ([flask_app.py L1227][b3-cfg-post]):

```sh
H=http://localhost:8080
curl -s $H/hokku/api/config | jq '.dither_presets |
  {image_config_default: (.default_general|del(.label,.description)),
   image_config_bw:      (.default_bw|del(.label,.description)),
   image_config_face:    (.default_face|del(.label,.description))}' > /tmp/b3-presets.json
curl -s -X POST -H 'Content-Type: application/json' --data @/tmp/b3-presets.json $H/hokku/api/config
# -> {"ok":true,"restarting":false}
```

"Keep my tuning, just switch autocontrast off":

```sh
curl -s $H/hokku/api/config | jq '.config | {image_config_default, image_config_bw, image_config_face}
  | map_values(.prepare_autocontrast = "off")' > /tmp/ac-off.json
curl -s -X POST -H 'Content-Type: application/json' --data @/tmp/ac-off.json $H/hokku/api/config
```

**Hand-editing `config.json`** also works. Stop the service, edit, then
`systemctl restart hokku-server` ([install.md L62][b3-install-svc]). Invalid
values are rejected at load (strict parser since v10), and the service then
exits, retrying every 30 s ([service L49-50][b3-service]).

Saving writes `"version": 11` to disk and reloads the config in-process. No
restart is needed.

---

## 7. Restart (if you did not already)

The `.deb` upgrade restarts the service by itself (§4.A). For path B, or after
a hand edit:

```sh
sudo systemctl restart hokku-server
sudo journalctl -u hokku-server -n 40 --no-pager
```

---

## 8. Regenerate the images

### 8.1 What actually happens (read this)

The cache key for every render is a hash of the **entire** `ImageConfig`
dataclass ([image_config.py L67-69][b3-imgcfg-slug]), folded into
`ScreenImageConfig.cache_slug()` ([screen_image_config.py][b3-sic]). beta3 adds
the field `prepare_autocontrast`, so **every image's slug changes on the first
beta3 start, even with identical values**. The server logs
`ScreenImageConfig slug changed for '<name>': re-converting` for every picture
and re-renders the whole library in the background
([image_manager_abstract.py L969][b3-slug-changed]). Saving new pipeline
settings (§6.3) changes the slug again and triggers another full re-render
automatically.

The release notes say existing renders are reused. **That is not what happens.**
See §12.2.

So a manual regeneration is not strictly needed. Do it anyway, once, **after**
choosing the settings in §6, for three reasons:

- it re-runs the B&W/face **classifier** too. Clearing the conversion cache
  alone does not;
- it deletes the **stale** beta2-slug files left behind in `<cache_dir>/images/`.
  In testing, the cache grew from 25 to 45 files after the upgrade, and a
  clear brought it back to 25;
- it gives you a clean, known point to verify from.

### 8.2 Steps

**UI:** Admin tab → **Clear caches and reconvert** → confirm
([index.html L1167/L2863][b3-ui-clear]). The manual calls this button
"Clear Cache & Re-convert" ([manual L131][b3-manual-clear]). Progress shows in
the Images-tab status strip.

**API** (exactly what the button does):

```sh
H=http://localhost:8080
curl -s -X POST $H/hokku/api/classifier/clear    # -> {"ok":true}  re-detect B&W / faces
curl -s -X POST $H/hokku/api/clear_cache         # -> {"ok":true}  delete panels/previews/thumbs, mark all pending, sync
```

([flask_app.py L778/L784][b3-clear-api],
[`clear_caches()` L626][b3-clear-caches]). Per-picture overrides survive this
([manual L50][b3-manual-perpic]).

Lighter option, if you only want the stale files gone:
`curl -X POST $H/hokku/api/scrub` ([flask_app.py L1004][b3-scrub]).

Deleting `/var/lib/hokku/cache/` by hand while the service is stopped also
works (the DB rebuilds from `upload_dir`), but **loses per-picture overrides**,
which are stored in `image_manager.json`. Prefer the API.

### 8.3 Watch it finish

```sh
watch -n5 "curl -s localhost:8080/hokku/api/status | jq -c '{converting, converting_done, converting_total}'"
sudo journalctl -u hokku-server -f | grep -E 'Dithered|Dithering complete'
```

Done when the status reads `{"converting":0,...}` and the log shows
`Dithering complete: all N image(s) done`. On an 8-core VM, 5 photos took
about 5 s. A Pi Zero 2 W takes roughly 1–2 s per image per orientation
([CHANGELOG 3.0 alpha][b3-changelog-numba]). Frames pick up the new renders at
their next scheduled refresh, or press a frame's button.

---

## 9. Verification checklist

| # | Command | Expected |
|---|---|---|
| 1 | `dpkg-query -W hokku-server` | `hokku-server	4.0.0~beta3-1` |
| 2 | `systemctl is-active hokku-server` | `active` |
| 3 | `journalctl -u hokku-server -b --no-pager \| grep -E 'starting — version\|Bundled firmware\|Serving on'` | `Hokku image server starting — version 4.0.0b3`, `Bundled firmware: hokku-huessen_epf1301-1.2.25.bin (version 1.2.25)`, `Serving on http://0.0.0.0:8080` |
| 4 | `curl -s localhost:8080/hokku/api/config \| jq '.git_describe, .config.version'` | `"4.0.0b3"`, `11` |
| 5 | `curl -s localhost:8080/hokku/api/config \| jq -c '[.config.image_config_default, .config.image_config_bw, .config.image_config_face] \| map(.prepare_autocontrast)'` | `["per_channel","per_channel","per_channel"]` straight after upgrade; `["off","off","off"]` after §6 |
| 6 | `sudo jq .version /var/lib/hokku/config.json` | `9` until first save, then `11` |
| 7 | `curl -s localhost:8080/hokku/api/config \| jq -r '.dither_preset_order[]'` | starts `default_general`, `default_bw`, `default_face` |
| 8 | `curl -s localhost:8080/hokku/api/status \| jq -c '{converting, converting_done, converting_total}'` | `{"converting":0,...}` once regeneration is done |
| 9 | `sudo jq -r '.images[] \| "\(.name) \(.convert_status)"' /var/lib/hokku/cache/image_manager.json` | every image `ok` |
| 10 | `curl -s -o /tmp/p.bin -w '%{http_code} %{size_download}\n' -H 'X-Screen-Name: check' -H 'X-Screen-Model: huessen_epf1301' localhost:8080/hokku/screen/` | `200 960000` (Huessen panel size). Note: this registers a fake screen named `check` in the dashboard |
| 11 | `curl -s localhost:8080/hokku/api/image/<name>/config \| jq -c .overrides` | `{"crop_to_fill_threshold":null,"image_config":null}` (per-picture API present) |
| 12 | `curl -s localhost:8080/hokku/api/status \| jq '.bundled_firmware_versions'` | `huessen_epf1301: "1.2.25"`, `bigme_f7: "1.2.12"`, `seeedstudio_e1004: "1.2.5"` |
| 13 | Web UI → Config | Each pipeline editor shows *Autocontrast*. After §6 the preset dropdowns read *General (default)* / *Black & white (default)* / *Faces (default)* |

---

## 10. Frame firmware

**A firmware update is not required for beta3.** Firmware is versioned
`PROTOCOL.CONFIG.N` ([AGENTS.md "Versioning — firmware"][b3-agents]). Huessen
goes `1.2.21 → 1.2.25` ([VERSION][b3-fw-ver]) and Bigme F7 goes
`1.2.10 → 1.2.12` ([rel-b3][rel-b3]). Neither bumps `PROTOCOL` (wire protocol)
or `CONFIG` (NVS schema), so beta2-era firmware speaks the same protocol. In
testing, a request claiming `X-Firmware-Version: 1.2.21` got a normal
`200` + 960000-byte panel from beta3. The server also still serves pre-1.2.9
frames that send no model header ([flask_app.py L288-294][b3-screen]).

**Recommended, not required:** Huessen 1.2.25 adds USB whole-panel upload,
interactive mode, and "three protocol bugs found bringing it up on real
hardware" ([CHANGELOG][b3-changelog]). The upgraded `.deb` bundles 1.2.25, and
the dashboard flags older frames as outdated ([manual L78-80][b3-manual-ota]).

How to update (OTA; frame must already run Hokku firmware with OTA support):

1. Web UI → click the frame → tick **Update firmware on next refresh**
   ([index.html L3278][b3-ui-ota], [manual L82-88][b3-manual-ota]).
2. At its next check-in the frame downloads the firmware into the inactive A/B
   slot and boots it. If it cannot reach the server afterwards, it rolls back to
   the previous slot automatically ([manual L82-88][b3-manual-ota]).
3. Verify: the frame's dashboard entry shows `firmware_version` `1.2.25` and
   the outdated flag clears
   (`curl -s localhost:8080/hokku/api/status | jq '.screens'`).

Do one frame first. Do **not** USB-flash with raw `esptool`/full erase unless
you know the frame's partition layout; use the web UI *Flash a screen* page or
`hokku-setup` ([manual §Flash a screen][b3-manual-ota],
[install.md][b3-install]). Firmware pins in the Firmware library
(`firmware_dir/selection.json`) override "highest bundled". If you pinned
1.2.21 under beta2, unpin it (*Auto (highest stable)*) to be offered 1.2.25
([manual L143-148][b3-manual-fw]).

---

## 11. Rollback to beta2

The clean rollback restores the §3.3 tarball:

```sh
sudo systemctl stop hokku-server
sudo apt install --allow-downgrades ./hokku-server_4.0.0.beta2-1_all.deb    # postinst restarts the service
sudo systemctl stop hokku-server
sudo mv /var/lib/hokku /var/lib/hokku.beta3-$(date +%s)                     # keep beta3 state for later
sudo tar -C /var/lib -xzf /root/hokku-backup-<TS>/var-lib-hokku.tgz
sudo systemctl start hokku-server
journalctl -u hokku-server -n 30 --no-pager | grep 'starting — version'     # -> version 4.0.0b2
```

Tested: after the restore, the beta2 panel files matched the pre-upgrade
checksums byte-for-byte, and the config was back at `"version": 9`.

Things to know:

- `apt install` of the older `.deb` needs `--allow-downgrades`. The beta2
  `postinst` starts the service immediately, on whatever `config.json` is
  there.
- beta2 does **not** refuse a `"version": 11` config. Its `_migrate` only walks
  upward ([B2 app_config.py `_migrate`][b2-migrate]). It starts and
  **re-renders everything** (new slug), using whatever pipeline values the v11
  file holds (for example beta3's Atkinson/OKLAB). That is "beta2 code with
  beta3 settings", not your old library. Restore the tarball (or at least
  `config.v9.json`) for a true rollback.
- Per-picture overrides set under beta3 are ignored by beta2, because its
  `ImageRecord.from_dict` does not read those keys
  ([B2 image_record.py][b2-rec]). They are dropped the next time beta2 saves
  the DB.
- Frames already on 1.2.25 keep working with beta2, since the protocol is the
  same. To downgrade a frame, pin 1.2.21 in the beta2 Firmware library and use
  *Update firmware on next refresh* (§10).

---

## 12. Verification log and deviations from the docs

### 12.1 What was run

Environment: Debian 12 (bookworm) with systemd (container), official assets
from the GitHub releases, no source builds.

1. `apt install ./hokku-server_4.0.0.beta2-1_all.deb`. `postinst` pip-installed
   its extras. The service came up as `version 4.0.0b2`, bundled firmware
   1.2.21, Waitress on :8080.
2. Uploaded 4 test photos from `images/test/` (portrait with face, B&W forest,
   group photo, AVIF landscape) via `POST /hokku/api/upload`. All converted.
   The classifier marked the B&W photo `is_bw` and found faces. Result: 25
   cache files, config v9.
3. Stopped the service and took the tarball backup (§3.3).
4. `apt install ./hokku-server_4.0.0.beta3-1_all.deb`: 0 new downloads, service
   restarted automatically, `version 4.0.0b3`, bundled firmware 1.2.25.
   In-memory config v11 with `prepare_autocontrast` = `per_channel` ×3. The
   on-disk file was still v9 and byte-identical. **All 5 images re-converted
   immediately** (slug change). The cache grew to 45 files (stale beta2 files
   kept).
5. `POST /hokku/api/config` with `prepare_autocontrast: off` ×3. The file
   became v11, the config reloaded in-process, and all 5 re-converted again.
6. `POST /hokku/api/config` with the three beta3 default presets. Checked:
   `dither_presets.default_*` equal the beta3 `config.json.example` pipelines.
   All re-converted.
7. `POST /hokku/api/classifier/clear` then `POST /hokku/api/clear_cache`. The
   classifier re-ran (5 observations), all 5 re-dithered, all
   `convert_status: ok`, and the cache was back to 25 files (stale files gone).
8. `GET /hokku/screen/` as a 1.2.21 Huessen frame returned `200`, a
   960000-byte body, and normal `X-Sleep-Seconds` headers.
9. `PATCH /hokku/api/image/Fitz_Roy_1.avif/config {"crop_to_fill_threshold":0.3}`
   returned `{"ok":true,"queued":true}`, and `GET` showed the override.
10. Rollback: `apt install --allow-downgrades` beta2. Before the tarball
    restore, beta2 ran on the v11 config and re-rendered everything. After
    `tar -x` of the backup, the service was `version 4.0.0b2`, config v9, and
    the panel `md5sum`s were identical to step 2.
11. Path B: in a fresh Python 3.10 venv, `pip install` the beta2 wheel, then
    `pip install --upgrade` the beta3 wheel, then
    `hokku-server <beta2-example-config>`. It started as `4.0.0b3` and
    converted a photo. The config stayed v9 on disk, as with the `.deb`.

### 12.2 Deviations from the documentation

1. **"Photographs already converted stay as they are … existing renders are
   reused"** ([release notes][rel-b3], [CHANGELOG L80-84][b3-changelog-upg]):
   **false for a beta2 upgrade.** Adding the `prepare_autocontrast` field
   changes `ImageConfig.cache_slug()` for everyone
   ([image_config.py L67-69][b3-imgcfg-slug]), so the whole library
   re-renders on first start, with the new beta3 tone chain but the old
   pipeline values. Plan for the CPU time on a Pi with a large library.
2. **"Changing any setting doesn't automatically re-convert existing images"**
   ([manual L117][b3-manual-preset]): with the in-process reload, saving
   pipeline settings **does** re-queue every affected image (log:
   `ScreenImageConfig slug changed … re-converting`). The manual's advice to
   press *Clear Cache & Re-convert* is still useful for the classifier and for
   stale files.
3. Button label: the docs say **"Clear Cache & Re-convert"**, but the beta3 UI
   says **"Clear caches and reconvert"** ([index.html L1167][b3-ui-clear]).
4. [docs/install.md][b3-install-cfg] shows an example config with
   `"version": 6`. The real shipped example is `"version": 11`. This is
   harmless: old versions migrate.
5. The on-disk `config.json` keeps `"version": 9` after the upgrade until the
   first save. No doc says otherwise, but it is easy to misread as "migration
   failed".
6. Stale cache files from beta2 slugs are not removed after the upgrade unless
   `auto_clear_cache` is `true` (default `false`), or you run *Clear caches* or
   `/hokku/api/scrub` ([image_manager_abstract.py L994][b3-scrub-orphan]).

### 12.3 Not verified

- Raspberry Pi appliance hardware (path C) and real frames. OTA was not
  exercised; a frame was only simulated over HTTP.
- Bigme F7 and Seeed E1004 panels.
- Debian 11/13, and hosts without internet during `postinst` (on an upgrade no
  downloads were needed, but a fresh beta2 install downloads a lot).

---

## 13. Open questions for your setup

1. **Install path:** `.deb` on plain Debian, the Pi appliance, or a source/venv
   checkout? Check `dpkg-query -W hokku-server` and `systemctl cat hokku-server`.
2. **Custom pipeline tuning:** did you deliberately tune any of the three
   pipelines under beta2? If yes, keep them and only switch *Autocontrast*
   (§6.3, second command). If no, take the three beta3 default presets.
3. **Non-default paths:** is `upload_dir` / `cache_dir` on another disk or a
   Samba share? Back those up too, and make sure the `hokku` user still owns
   them.
4. **Library size and hardware:** how many photos, on what CPU? The upgrade
   triggers at least one full re-render, and up to three if you follow §6 and
   §8 separately. On slow hardware, do §6 and §8 back to back to avoid wasted
   passes.
5. **Frames:** which models and firmware versions are on the wall
   (`/hokku/api/status → screens`)? Are any pinned to a specific firmware in
   the Firmware library? Do you want them on 1.2.25 (optional)?
6. **Per-picture overrides:** do you plan to use them? They do not survive a
   rollback to beta2.
7. **Offline host:** does the server have internet access? That only matters if
   `postinst` finds a pip dependency missing or too old.

[c-b2]: https://github.com/defl/hokku_epaper/commit/74a888e14aae8a6a3b9070228ee4b52c43dc07c5
[c-b3]: https://github.com/defl/hokku_epaper/commit/0869a6058d4e77d2aa18f7d5bc704e4aca979468
[rel-b2]: https://github.com/defl/hokku_epaper/releases/tag/v4.0.0-beta2
[rel-b3]: https://github.com/defl/hokku_epaper/releases/tag/v4.0.0-beta3
[b3-readme]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/README.md
[b3-install]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/install.md
[b3-install-deb]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/install.md#L53-L56
[b3-install-svc]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/install.md#L61-L63
[b3-install-cfg]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/install.md#L68-L80
[b3-install-src]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/install.md#L89-L97
[b3-appliance]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/appliance.md
[b3-manual-perpic]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/manual.md#L43-L50
[b3-manual-ota]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/manual.md#L78-L95
[b3-manual-preset]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/manual.md#L111-L117
[b3-manual-clear]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/manual.md#L131
[b3-manual-fw]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/docs/manual.md#L138-L148
[b3-changelog]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/CHANGELOG.md#L3-L97
[b3-changelog-upg]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/CHANGELOG.md#L71-L84
[b3-changelog-classifier]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/CHANGELOG.md#L583-L590
[b3-changelog-numba]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/CHANGELOG.md#L612-L614
[b3-agents]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/AGENTS.md#L138-L143
[b3-pyproject]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/pyproject.toml
[b3-req]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/requirements.txt
[b3-debian]: https://github.com/defl/hokku_epaper/tree/v4.0.0-beta3/python/debian
[b3-deb-changelog]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/debian/changelog
[b3-control]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/debian/control
[b3-postinst]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/debian/postinst
[b3-postinst-stop]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/debian/postinst#L4-L8
[b3-service]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/debian/hokku-server.service
[b3-pigen]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/os/pi/stage-hokku/00-install/00-run.sh
[b3-fw-ver]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/firmware/huessen_epf1301/VERSION
[b3-example]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/config/config.json.example
[b2-example]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta2/python/hokku/webserver/config/config.json.example
[b3-appcfg-ver]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/app_config.py#L32
[b3-mig910]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/app_config.py#L102-L131
[b3-mig1011]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/app_config.py#L134-L148
[b3-migrate]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/app_config.py#L166
[b3-appcfg-fields]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/app_config.py#L196-L220
[b3-appcfg-load]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/app_config.py#L341-L346
[b2-migrate]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta2/python/hokku/webserver/app_config.py#L114-L120
[b3-imgcfg-ac]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/image_config.py#L20-L40
[b3-imgcfg-slug]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/image_config.py#L67-L69
[b3-sic]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/screen_image_config.py#L38-L60
[b3-db-ver]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/image_manager_abstract.py#L55-L56
[b3-clear-caches]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/image_manager_abstract.py#L626
[b3-slug-changed]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/image_manager_abstract.py#L969
[b3-scrub-orphan]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/image_manager_abstract.py#L994
[b3-rec]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/image_record.py#L61-L62
[b2-rec]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta2/python/hokku/webserver/image_record.py#L68-L96
[b3-screen]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/flask_app.py#L279-L294
[b3-img-cfg-api]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/flask_app.py#L673-L758
[b3-clear-api]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/flask_app.py#L778-L790
[b3-scrub]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/flask_app.py#L1004-L1008
[b3-cfg-post]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/flask_app.py#L1227-L1260
[b3-ui-clear]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/templates/index.html#L1167
[b3-ui-ac]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/templates/index.html#L2064-L2070
[b3-ui-ota]: https://github.com/defl/hokku_epaper/blob/v4.0.0-beta3/python/hokku/webserver/templates/index.html#L3275-L3280
