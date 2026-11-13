# ESP On Demand — end-to-end: from nothing to playing video

The complete walkthrough. Real boards are the intended way to run this; a
no-hardware emulator path exists for testing and is described at the end.

- **Real boards:** flash your ESP32s, start the dashboard, press play.
- **Testing without boards:** the whole chain runs on one PC with virtual
  boards — see [Testing without boards](#testing-without-boards-emulator)
  (also in the main repo as
  [docs/TESTING.md](https://github.com/ESPlab-labs/ESPlab/blob/main/docs/TESTING.md)).

Both paths end the same way: a browser page where pressing play makes the
cluster decode every frame in parallel.

---

## Step 1 — flash real boards

Full details in the main repo's hardware guide:
**[docs/HARDWARE.md](https://github.com/ESPlab-labs/ESPlab/blob/main/docs/HARDWARE.md)**
(toolchain install, per-chip builds, serial ports, troubleshooting).

```bash
git clone https://github.com/ESPlab-labs/ESPlab && cd ESPlab
cd firmware
idf.py set-target esp32          # or esp32s2 / esp32s3 / esp32c3
idf.py -p /dev/ttyUSB0 flash monitor
```

Repeat per board (each on its own port). Boards elect roles themselves —
the same image goes on every board, mixed chips included. A board that sees
no cluster elects itself coordinator; the rest join as workers.

Sanity check from any board's console (`idf.py -p PORT monitor`):

```text
(esplab) status        # lists every board with its chip type
```

## Step 2 — start the WebUI (the machine's dashboard)

On the computer the boards are plugged into:

```bash
python3 tools/webui/webui.py
# dashboard: http://localhost:8080
```

Open the URL: you should see every board, zero jobs, and the machine log.
**ESP On Demand requires this service** — it is EOD's only path to the
cluster.

## Step 3 — encode a clip

```bash
python3 tools/eod_prep.py test test --synth    # test clip, no ffmpeg needed
# or from any video file (needs ffmpeg):
python3 tools/eod_prep.py myvideo.mp4 myclip
```

## Step 4 — start ESP On Demand

```bash
python3 tools/eod_serve.py --webui http://127.0.0.1:8080 \
        --clip test.eod --port 8090
```

`--webui` points at the dashboard from Step 2 — that's the whole
dependency: EOD never talks to boards directly, only through the dashboard's
console-backed API.

## Step 5 — press play

Open `http://localhost:8090/` and press play. Every frame you see is being
decoded by the cluster as you watch: 12 pixel-band jobs per frame,
distributed across every board by the machine.

Watch the dashboard at :8080 while it plays — the frame jobs run on the
boards, live. Per-frame decode times: `http://localhost:8090/metrics`.

### If something looks wrong

| symptom | fix |
|---|---|
| dashboard shows 0 boards | give the boards ~10 s; check serial ports with `ls /dev/ttyUSB* /dev/ttyACM*` |
| player says the WebUI is unreachable | EOD started before the WebUI — start the WebUI first, then restart EOD |
| boards never join | keep boards within a meter; metal between boards blocks the radio (see HARDWARE.md troubleshooting) |
| video plays but stalls | more boards = more parallel bands = faster frames |

---

## Testing without boards (emulator)

Every part of ESPlab is portable C, so a "board" can be a Linux process.
The emulator path runs the identical code and wire protocol as real boards
— details and caveats (it is a correctness tool, not a speed tool) in
[docs/TESTING.md](https://github.com/ESPlab-labs/ESPlab/blob/main/docs/TESTING.md).
Short version:

```bash
git clone https://github.com/ESPlab-labs/ESPlab && cd ESPlab && make
./build/vboard --id 1 --cores 2 --ram 290 --flash 4 --console &
./build/vboard --id 2 --chip s3 --cores 2 --ram 512 --flash 8 --console &
python3 tools/webui/webui.py            # dashboard: http://localhost:8080
python3 tools/eod_prep.py test test --synth
python3 tools/eod_serve.py --webui http://127.0.0.1:8080 \
        --clip test.eod --port 8090
```

One-command demo: `python3 tools/eod_serve.py --own-cluster --clip test.eod`
spawns its own 3-board virtual cluster and serves on :8090.

---

## Where the pieces live

| piece | repo |
|---|---|
| the machine (firmware, VM, virtual boards) | [ESPlab](https://github.com/ESPlab-labs/ESPlab) |
| the dashboard service (required) | [ESPlab-WebUI](https://github.com/ESPlab-labs/ESPlab-WebUI) |
| this media service | [ESP-On-Demand](https://github.com/ESPlab-labs/ESP-On-Demand) |

## What "end-to-end" proves

The full chain — browser → EOD service → WebUI → machine console →
coordinator → boards (via radio-style fragmented messages) → bytecode
decoder → reassembled pixels — has been verified frame-exact: the complete
75-frame test clip decodes byte-identical to an independent reference
decoder, with every band job round-tripping the real wire protocol.
