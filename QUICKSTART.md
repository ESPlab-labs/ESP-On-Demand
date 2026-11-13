# ESP On Demand — end-to-end: from nothing to playing video

This is the complete walkthrough. It has two paths:

- **Path A: no hardware** — everything runs on your PC with virtual boards.
  Ten minutes, zero soldering, and the exact same code path as real boards.
- **Path B: real boards** — actual ESP32s on your desk. Start at
  [Step 0](#step-0--flash-real-boards-esplab-firmware) after doing
  [Path A prerequisites](#path-a--no-hardware-recommended-first).

Both paths end the same way: a browser page where pressing play makes the
cluster decode every frame in parallel.

---

## Path A — no hardware (recommended first)

### Step 1 — get the code and build the machine

```bash
git clone https://github.com/TycoonCoder/ESPlab
cd ESPlab
make                # builds vboard (virtual boards), vbasm, eod_enc, tests
make test           # must print ALL TESTS PASSED
```

No compiler? The prebuilt tools are in
[Releases](https://github.com/TycoonCoder/ESP-On-Demand/releases) — but
building is one command if you have gcc/make.

### Step 2 — start three virtual boards

Open three terminals (or run them in the background):

```bash
./build/vboard --id 1 --cores 2 --ram 290 --flash 4 --console &
./build/vboard --id 2 --chip s3 --cores 2 --ram 512 --flash 8 --console &
./build/vboard --id 3 --cores 2 --ram 290 --flash 4 --console &
```

Within seconds they announce, elect a brain (the S3 wins on merit), and
report `joined cluster: 3 nodes`.

### Step 3 — start the WebUI (the machine's dashboard)

```bash
python3 tools/webui/webui.py
# dashboard: http://localhost:8080
```

The dashboard spawns its own virtual board and joins the machine. Open the
URL: you should see 4 boards, zero jobs, and the machine log.

### Step 4 — encode a test clip

```bash
python3 tools/eod_prep.py test test --synth    # makes test.eod, no ffmpeg needed
# or from any video file (needs ffmpeg):
python3 tools/eod_prep.py myvideo.mp4 myclip
```

### Step 5 — start ESP On Demand

```bash
python3 tools/eod_serve.py --webui http://127.0.0.1:8080 \
        --clip test.eod --port 8090
```

`--webui` must point at the dashboard from Step 3 — that's the whole
dependency: EOD never talks to boards directly, only through the dashboard's
console-backed API.

One-command alternative: `python3 tools/eod_serve.py --own-cluster --clip
test.eod` spawns its own 3-board virtual cluster and serves on :8090 —
handy for a smoke test, though the WebUI path above is the real one.

### Step 6 — press play

Open `http://localhost:8090/` and press play. Every frame you see is being
decoded by the cluster as you watch: 12 pixel-band jobs per frame,
distributed across every board by the machine.

Check the dashboard at :8080 while it plays — you'll see the frame jobs
running on the boards, live. Per-frame decode times: `http://localhost:8090/metrics`.

### Sanity checks if something looks wrong

| symptom | fix |
|---|---|
| dashboard shows 0 boards | give the boards ~10 s; check they're running and no firewall blocks localhost UDP |
| player says the WebUI is unreachable | EOD started before the WebUI — start the WebUI first, then restart EOD |
| video plays but stalls | the virtual boards are deliberately slow; fewer boards = slower. This is a correctness demo — real boards are far faster |
| `make` fails | you need gcc + make (`sudo apt install build-essential` on Debian/Ubuntu) |

---

## Step 0 — flash real boards (ESPlab firmware)

Do [Path A](#path-a--no-hardware-recommended-first) first, or skip to the
full hardware guide: **[docs/HARDWARE.md](https://github.com/TycoonCoder/ESPlab/blob/main/docs/HARDWARE.md)**
in the main repo. Short version:

```bash
# after installing ESP-IDF v5.2 (see docs/HARDWARE.md Step 1):
cd firmware
idf.py set-target esp32        # or esp32s2 / esp32s3 / esp32c3
idf.py -p /dev/ttyUSB0 flash monitor
# repeat per board on its own port; boards elect roles themselves
```

Then run Steps 3–6 above unchanged — the web services talk to real boards
over their serial console instead of virtual ones, and everything else is
identical.

---

## Where the pieces live

| piece | repo |
|---|---|
| the machine (firmware, VM, virtual boards) | [ESPlab](https://github.com/TycoonCoder/ESPlab) |
| the dashboard service (required) | [ESPlab-WebUI](https://github.com/TycoonCoder/ESPlab-WebUI) |
| this media service | [ESP-On-Demand](https://github.com/TycoonCoder/ESP-On-Demand) |

## What "end-to-end" proves

The full chain — browser → EOD service → WebUI → machine console →
coordinator → boards (via radio-style fragmented messages) → bytecode
decoder → reassembled pixels — has been verified frame-exact: the complete
75-frame test clip decodes byte-identical to an independent reference
decoder, with every band job round-tripping the real wire protocol.
