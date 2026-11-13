# End-to-end walkthrough: from nothing to playing video

The complete instructions — flashing real boards first, with the emulator
path as a separate testing option — live in
[QUICKSTART.md](https://github.com/ESPlab-labs/ESP-On-Demand/blob/main/QUICKSTART.md).

Real boards, short version:

```bash
# flash boards first: ESPlab/docs/HARDWARE.md
python3 tools/webui/webui.py            # dashboard: http://localhost:8080
python3 tools/eod_prep.py test test --synth
python3 tools/eod_serve.py --webui http://127.0.0.1:8080 \
        --clip test.eod --port 8090
```

Open http://localhost:8090/ and press play.
