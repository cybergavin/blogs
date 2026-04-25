## vLLM + Gemma 4 26B (AWQ 4-bit) on EC2 (L4 GPU)
### Tested on: g6.2xlarge (1x NVIDIA L4, 24GB VRAM)

## 1. EC2 Requirements
- **AMI:** amazon/Deep Learning OSS Nvidia Driver AMI GPU PyTorch 2.10 (Amazon Linux 2023) 20260418
- **Instance:** g6.2xlarge
- **Disk:** 100GB gp3 (the AMI has ~480 GB NVMe which is adequate)
- **Ports:**
  - 8000 (inbound)
  - SSH (tcp/22 inbound or tcp/443 outbound for SSM Session Manager)
---

## 2. System Preparation

### Upgrade packages

```bash
sudo apt update && sudo apt install -y git
```

### Set up mountpoint, service account and permissions

For the AMI specified in section 1, execute the following script:

**Note:** Ensure the disk partition being used for /opt/models is available as a raw partition (no filesystem or data)

```bash
# 1. Format the large disk with XFS (only do this if it's currently empty!)
sudo mkfs -t xfs /dev/nvme1n1

# 2. Create the target mount point
sudo mkdir -p /opt/models

# 3. Mount it immediately
sudo mount /dev/nvme1n1 /opt/models

# 1. Extract the UUID of the new filesystem
DISK_UUID=$(blkid -s UUID -o value /dev/nvme1n1)

# 2. Backup your fstab just in case
sudo cp /etc/fstab /etc/fstab.bak

# 3. Add the entry to /etc/fstab
# xfs: the filesystem type
# defaults,nofail: allows the system to boot even if the disk is missing
echo "UUID=$DISK_UUID /opt/models xfs defaults,nofail 0 2" | sudo tee -a /etc/fstab

# 4. Verify there are no errors in fstab
sudo findmnt --verify
df -h /opt/models

# Create a system group and user
sudo groupadd -r vllm
sudo useradd -r -g vllm -d /opt/models -s /sbin/nologin -c "vLLM Service User" vllm
# Change ownership to the new vllm user
sudo chown -R vllm:vllm /opt/models
# Set directory permissions (rwxr-xr-x)
sudo chmod 755 /opt/models
```
---

## 3. Install uv + Python env
```bash
curl -Lsf https://astral.sh/uv/install.sh | sh
uv venv /opt/models/.venvs/vllm --python 3.11
source /opt/models/.venvs/vllm/bin/activate
```
---
## 4. Install vLLM

**NOTE:** I used pinned commit due to compatibility issues with latest release at time of testing

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm
git reset --hard 47e605092b7fce3d64264b34250b1a286f344633
VLLM_USE_PRECOMPILED=1 uv pip install -e .

# Supporting libs
uv pip install transformers huggingface_hub
```
---
## 5. Set up Hugging Face Account
- Set up Hugging Face account if you don't have one
- Generate a Hugging Face token
---

## 6. Download Model (Gemma 4 26B AWQ)

```bash
export HF_HOME=/opt/models
export HF_TOKEN=<your_huggingface_token>

python - <<EOF
from huggingface_hub import snapshot_download

snapshot_download(
    repo_id="cyankiwi/gemma-4-26B-A4B-it-AWQ-4bit",
    local_dir="/opt/models/gemma-4-26B-A4B-it",
    token="${HF_TOKEN}"
)
EOF
```
---
## 7. Run vLLM (Manual Test)

**IMPORTANT:** Run outside the vllm repo directory to avoid import conflicts

```bash
cd /tmp

vllm serve /opt/models/gemma-4-26B-A4B-it \
  --served-model-name gemma-4-26b-a4b \
  --dtype float16 \
  --gpu-memory-utilization 0.90 \
  --max-model-len 32768 \
  --max-num-seqs 16 \
  --enable-chunked-prefill \
  --enable-prefix-caching \
  --enable-auto-tool-choice \
  --tool-call-parser gemma4 \
  --reasoning-parser gemma4 \
  --generation-config auto \
  --host 0.0.0.0 \
  --port 8000
```
---
## 8. Create systemd Service
```bash
sudo useradd -r -s /bin/false vllm || true
sudo chown -R vllm:vllm /opt/models

sudo tee /etc/systemd/system/vllm.service > /dev/null <<EOF
[Unit]
Description=vLLM Gemma 4 26B A4B
After=network.target

[Service]
Type=simple
User=vllm
Group=vllm
WorkingDirectory=/tmp

Environment="HF_HOME=/opt/models"
Environment="VLLM_NO_USAGE_STATS=1"

ExecStart=/opt/models/.venvs/vllm/bin/vllm serve \
  /opt/models/gemma-4-26B-A4B-it \
  --served-model-name gemma-4-26b-a4b \
  --dtype float16 \
  --gpu-memory-utilization 0.90 \
  --max-model-len 32768 \
  --max-num-seqs 16 \
  --enable-chunked-prefill \
  --enable-prefix-caching \
  --enable-auto-tool-choice \
  --tool-call-parser gemma4 \
  --reasoning-parser gemma4 \
  --generation-config auto \
  --host 0.0.0.0 \
  --port 8000

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```
---
## 9. Enable Service
```bash
sudo systemctl daemon-reload
sudo systemctl enable vllm
sudo systemctl start vllm

# Logs
sudo journalctl -u vllm -f
```
---
## 10. Verify API
```bash
curl http://localhost:8000/v1/models

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemma-4-26b-a4b",
    "messages": [
      {"role": "user", "content": "Hello, are you working?"}
    ],
    "max_tokens": 64
  }'
  ```