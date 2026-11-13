# ESP On Demand

[![release](https://img.shields.io/github/v/release/ESPlab-labs/ESP-On-Demand)](https://github.com/ESPlab-labs/ESP-On-Demand/releases)
[![license](https://img.shields.io/github/license/ESPlab-labs/ESP-On-Demand)](LICENSE)

A media server for [ESPlab](https://github.com/ESPlab-labs/ESPlab) clusters.
It serves video from your ESP32 cluster and uses every board to decode it:
each frame is split into horizontal bands, and the cluster's boards decode
the bands in parallel as sandboxed bytecode jobs. You watch in any browser
on your network.

Requires [ESPlab-WebUI](https://github.com/ESPlab-labs/ESPlab-WebUI), the
cluster's dashboard service — ESP On Demand reaches the boards only through
it.

[Full setup guide](QUICKSTART.md) ·
[Releases](https://github.com/ESPlab-labs/ESP-On-Demand/releases) ·
[Issues](https://github.com/ESPlab-labs/ESP-On-Demand/issues)

## How it works

```
your video ─▶ eod_prep.py ─▶ .eod clip ─▶ ESP On Demand ─▶ browser player
              (encode once,                    │
               ~11–25 KB/s)                    ▼
                                  ESPlab cluster: each frame becomes
                                  12 band jobs, decoded in parallel
```

- **Compact format.** Video is encoded once into EOD1: a 64-color palette
  plus run-length encoding, typically 11–25 KB/s of footage.
- **Parallel decode.** Every frame is a cluster job of 12 pixel-band tasks.
  More boards, more frames per second.
- **Any browser.** The player page draws what the boards return; nothing is
  decoded client-side.
- **Storage-friendly.** An hour of video is roughly 80–90 MB in EOD1, and
  the clip reader is isolated so external storage (SD card, USB drive) can
  back it later.

## Quick start

With a flashed cluster and the dashboard running (see the
[ESPlab hardware guide](https://github.com/ESPlab-labs/ESPlab/blob/main/docs/HARDWARE.md)):

```bash
# encode any video (ffmpeg required)
python3 tools/eod_prep.py myvideo.mp4 myclip

# serve it (dashboard on :8080, player on :8090)
python3 eod_serve.py --webui http://127.0.0.1:8080 \
        --clip myclip.eod --port 8090
```

Open `http://<machine>:8090/` and press play. The full walkthrough,
including a no-hardware test mode, is in
[QUICKSTART.md](QUICKSTART.md).

## Documentation

- [QUICKSTART.md](QUICKSTART.md) — from nothing to playing video
- [EOD1 format](https://github.com/ESPlab-labs/ESPlab/blob/main/tools/eod/README.md) —
  the clip format and its encoder/decoder

## Status

Plays real footage end to end (verified with a Creative-Commons Blender
clip at 64×48/10 fps). External storage backends are planned next.

## License

[MIT](LICENSE)
