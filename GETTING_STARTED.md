# End-to-end walkthrough: from nothing to playing video

The complete instructions — including flashing real boards and the
dashboard — live in
[QUICKSTART.md](https://github.com/TycoonCoder/ESP-On-Demand/blob/main/QUICKSTART.md).

The short version (no hardware needed):

```bash
git clone https://github.com/TycoonCoder/ESPlab && cd ESPlab && make
./build/vboard --id 1 --cores 2 --ram 290 --flash 4 --console &
./build/vboard --id 2 --chip s3 --cores 2 --ram 512 --flash 8 --console &
python3 tools/webui/webui.py            # dashboard: http://localhost:8080
python3 tools/eod_prep.py test test --synth
python3 tools/eod_serve.py --webui http://127.0.0.1:8080 \
        --clip test.eod --port 8090
```

Open http://localhost:8090/ and press play.
