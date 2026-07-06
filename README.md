# local-language-model
Add this to '.config/systemd/user/local-ai.service'
```
[Unit]
Description=Local AI Services (LLM & Embeddings via Podman Compose)
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
