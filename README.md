<p align="center">
  <img src="assets/icon-256.png" width="120" alt="whisper-ai" />
</p>

# whisper-ai

Whisper speech-to-text — this plugin targets a Wyoming/OpenAI-compatible ASR server.

A first-party [orca](https://github.com/argyle-labs/orca) plugin (service-backend).

This repo is **self-contained** — the steps below run whisper-ai **by hand, without orca**. orca automates exactly this (same image, ports, and data) through one generic surface.

---

## Run it without orca

### Docker / Podman

```yaml
# compose.yml
services:
  whisper-ai:
    image: onerahmet/openai-whisper-asr-webservice:latest
    container_name: whisper-ai
    restart: unless-stopped
    ports:
      - "9000:9000/tcp"   # ASR API
    volumes:
      - ./cache:/root/.cache
```

```sh
docker compose up -d
```

Podman: the same file with `podman-compose up -d`.

### Ports & data

| | |
|---|---|
| Default port | `9000` |
| Upstream | <https://github.com/ahmetoner/whisper-asr-webservice> |


### Backup & restore

Back up the config/data volume(s) above — that's the whole service state (stop the container first for a clean copy). Restore by putting them back and starting it.

> With orca this is **`service.backup` / `service.restore`** — location-agnostic (docker / podman / lxc / vm), one command regardless of where whisper-ai runs. No per-service backup script.

## With orca

orca drives this plugin through the single generic `service.*` surface — no per-plugin tools:

```sh
orca service.deploy whisper-ai      # render + launch on any supported runtime
orca service.status whisper-ai      # health + rich diagnostics (typed payload)
orca service.backup whisper-ai      # location-agnostic backup (tar; PBS on Proxmox)
orca service.configure whisper-ai   # apply config via the upstream API
```

## Layout

- `src/` — the plugin (pure Rust): the `ServiceBackend` descriptor + `configure` / `status`.
- [CAPABILITIES.md](CAPABILITIES.md) — the service-backend contract checklist.
- `assets/` — plugin icon.
