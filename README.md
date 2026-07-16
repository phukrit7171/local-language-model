# Local Language Model

Run a local vision-language model with [`llama.cpp`](https://github.com/ggml-org/llama.cpp) and rootless Podman.

The Compose service exposes the llama.cpp OpenAI-compatible HTTP API on port `8080` and uses the NVIDIA GPU through Podman's CDI device notation.

## Requirements

- Linux with a working NVIDIA driver
- Podman and the Podman Compose provider
- NVIDIA Container Toolkit with a usable CDI specification
- A model directory containing the files referenced by `docker-compose.yml`

Check the basic tools before starting:

```bash
podman --version
podman compose version
nvidia-smi
```

The Compose file expects these files in `models/`:

- `Qwythos-9B-v2-Q4_K_M.gguf`
- `mmproj-Qwythos-9B-v2-BF16.gguf`

Model files are ignored by Git because they are usually large. Copy or download them into `models/` locally; do not commit them.

For rootless GPU containers, confirm that Podman can see the NVIDIA CDI device:

```bash
podman info --format '{{.Host.Security.Rootless}}'
podman system migrate
podman run --rm --device nvidia.com/gpu=all \
  nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

If the last command cannot find `nvidia.com/gpu=all`, configure NVIDIA CDI according to the NVIDIA Container Toolkit documentation and retry. On many systems the CDI file is generated with:

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
nvidia-ctk cdi list
```

## Start manually

From the repository root:

```bash
podman compose up -d
podman compose ps
podman compose logs -f
```

The API is available at `http://127.0.0.1:8080`. For example:

```bash
curl http://127.0.0.1:8080/health
```

Stop the service and remove its Compose-created container with:

```bash
podman compose down
```

The container itself is configured with `restart: always`, so Podman will restart it after an unexpected container failure. The systemd service below starts the Compose project when the user service manager starts.

## Start automatically with rootless Podman and user systemd

User units belong in `~/.config/systemd/user/`; they must not be installed in `/etc/systemd/system/` because that would make this a system service and usually require root.

### 1. Install the unit

Create the user-unit directory and open the unit file at the requested path:

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/local-ai.service
```

Paste this content into `~/.config/systemd/user/local-ai.service`:

```ini
[Unit]
Description=Local AI Services (LLM & Embeddings via Podman Compose)
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/phukrit7171/Development/local-language-model

ExecStartPre=/usr/bin/bash -c 'until [ -c /dev/nvidia-modeset ]; do sleep 0.5; done'
ExecStart=/usr/bin/podman compose up -d
ExecStop=/usr/bin/podman compose down

[Install]
WantedBy=default.target
```

The unit assumes this repository is located at `/home/phukrit7171/Development/local-language-model`. If it is elsewhere, change `WorkingDirectory` to the repository path. Also verify the Podman path with `command -v podman`; if it is not `/usr/bin/podman`, use the path returned by that command in `ExecStart` and `ExecStop`.

### 2. Reload and enable the service

```bash
systemctl --user daemon-reload
systemctl --user enable --now local-ai.service
```

`enable` makes the service start with the user systemd manager, and `--now` starts it immediately.

To verify it:

```bash
systemctl --user status local-ai.service
systemctl --user is-enabled local-ai.service
podman compose ps
```

Follow service logs with:

```bash
journalctl --user -u local-ai.service -f
```

### 3. Start it after logout or reboot

By default, a user systemd manager normally runs only while the user has an active login session. Enable lingering so the service starts at boot and continues after logout:

```bash
sudo loginctl enable-linger "$USER"
loginctl show-user "$USER" -p Linger
```

Then reboot, or restart the user manager and check the service again:

```bash
systemctl --user restart local-ai.service
systemctl --user status local-ai.service
```

### Service lifecycle commands

```bash
# Start
systemctl --user start local-ai.service

# Stop (also runs `podman compose down`)
systemctl --user stop local-ai.service

# Restart
systemctl --user restart local-ai.service

# Disable automatic startup
systemctl --user disable --now local-ai.service
```

## Configuration notes

- The service waits for `/dev/nvidia-modeset` before starting. This avoids starting Compose before the NVIDIA driver is ready during boot.
- `Type=oneshot` with `RemainAfterExit=yes` is intentional: `podman compose up -d` exits after starting the container, while systemd keeps the unit marked active.
- Stopping the systemd unit runs `podman compose down`, removing the container. If you only want to stop the container without removing it, use `podman compose stop` manually instead.
- The API currently binds to all container interfaces and is published on host port `8080`. Restrict the host binding to `127.0.0.1:8080:8080` in `docker-compose.yml` if it should not be reachable from other machines on the network.
- The model is loaded from read-only `models/`, so model files must exist before starting the service.

## Troubleshooting

### Check the full service error

```bash
systemctl --user status local-ai.service --no-pager
journalctl --user -u local-ai.service -b --no-pager
```

### Check Compose and container logs

```bash
podman compose ps
podman compose logs --tail=200
podman logs local-language-model-local-language-model-1
```

The exact container name can differ by Podman Compose version; use `podman ps -a` if the last command cannot find it.

### The service starts but the container exits

Check that both model files exist and that the GPU is visible:

```bash
ls -lh models/
podman inspect local-language-model-local-language-model-1
podman compose logs
```

### The service does not start at boot

Confirm that lingering is enabled and that the user unit is installed in the correct location:

```bash
loginctl show-user "$USER" -p Linger
systemctl --user list-unit-files local-ai.service
```

If the NVIDIA device is not ready, the unit's pre-start wait will continue until `/dev/nvidia-modeset` appears. Check the driver with `nvidia-smi` and inspect the journal if it never appears.

## Files

- `docker-compose.yml` — llama.cpp container, GPU, port, model volume, and runtime arguments
- `~/.config/systemd/user/local-ai.service` — rootless user-level systemd unit
- `models/` — local GGUF and projector model files (ignored by Git)
