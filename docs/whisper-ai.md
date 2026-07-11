# Whisper AI

Speech-to-text transcription service using OpenAI Whisper (Faster Whisper engine).

**Status:** running — Docker, `10300 -> 9000`

- **Host**: `<host>` (`<ip>`)
- **Type**: Docker container (on a NAS or Docker host)
- **Port mapping**: `10300` (host) → `9000` (container)

> Whisper AI benefits from iGPU access for acceptable transcription performance. Running it as a Docker container on a host with Intel iGPU access is preferred over a Proxmox LXC.

---

## LXC Setup (reference)

> The LXC procedure below is kept for reference. A Docker container on a host with iGPU access is generally simpler.

### Step 1 — Create the LXC

```bash
pct create <vmid> local:vztmpl/debian-12-standard_12.x-1_amd64.tar.zst \
  --hostname whisper-ai \
  --storage local-lvm \
  --rootfs local-lvm:8 \
  --cores 4 \
  --memory 4096 \
  --swap 512 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --unprivileged 0
```

### Step 2 — Add GPU passthrough

Stop the LXC, then edit `/etc/pve/lxc/<vmid>.conf`:

```ini
dev0: /dev/dri/card0,gid=44
dev1: /dev/dri/renderD128,gid=44
```

### Step 3 — Install dependencies

```bash
pct start <vmid>
pct enter <vmid>

apt-get update && apt-get upgrade -y
apt-get install -y --no-install-recommends \
  curl gnupg python3 python3-pip ffmpeg \
  intel-media-va-driver-non-free vainfo

# Enable non-free if needed
sed -i "s/main/main contrib non-free non-free-firmware/g" /etc/apt/sources.list
apt-get update && apt-get install -y intel-media-va-driver-non-free vainfo
```

### Step 4 — Install Faster Whisper

```bash
pip3 install faster-whisper openai-whisper-asr-webservice
# or deploy via Docker inside the LXC:
apt-get install -y docker.io
systemctl enable --now docker
docker run -d \
  --name whisper-ai \
  --restart always \
  -p 8000:8000 \
  -e ASR_MODEL=base \
  -e ASR_ENGINE=faster_whisper \
  -e ASR_DEVICE=cpu \
  --device /dev/dri:/dev/dri \
  -v whisper_models:/app/models \
  onerahmet/openai-whisper-asr-webservice:latest
```

### Step 5 — Assign static IP

Add a static DHCP reservation on your router for the LXC MAC → next available IP.
Update your network map / inventory.

---

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `ASR_MODEL` | `base` | Model size: `tiny`, `base`, `small`, `medium`, `large` |
| `ASR_ENGINE` | `faster_whisper` | Inference engine |
| `ASR_DEVICE` | `cpu` | `cpu` or `cuda` — use `cpu` with Intel iGPU |
| `ASR_COMPUTE_TYPE` | `int8` | Quantization — `int8` is fastest on CPU |

**Model size guide:**

| Model | RAM | Accuracy |
|-------|-----|---------|
| tiny | ~1 GB | Low |
| base | ~1 GB | OK (default) |
| small | ~2 GB | Good |
| medium | ~5 GB | High |
| large | ~10 GB | Best |

---

## API

- `GET /health` — health check
- `POST /asr` — transcribe audio file

```bash
curl -X POST http://<host>:8000/asr \
  -F "audio_file=@audio.mp3" \
  -F "task=transcribe" \
  -F "language=en"
```

---

## Troubleshooting

```bash
docker logs whisper-ai -f
# Slow transcription → use smaller model or verify iGPU is accessible
vainfo  # should show Intel iHD driver
```
