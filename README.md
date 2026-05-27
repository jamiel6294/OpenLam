# OpenLam
Configuration Setup Steps for Ollama/OpenWeb-UI


## 1. On the online Ubuntu VM: create the offline-copy folder

1. **Create base folder and subfolders**

   ```bash
   mkdir -p ~/offline-copy/{ollama,ollama-models,openwebui,docker-images,scripts}
   ```

   You’ll end up with:

   - `offline-copy/ollama` → installer  
   - `offline-copy/ollama-models` → models + config  
   - `offline-copy/openwebui` → configs (optional)  
   - `offline-copy/docker-images` → saved Docker images  
   - `offline-copy/scripts` → helper scripts for air‑gapped host  

---

## 2. Download Ollama tar.zst into offline-copy (do not install yet)

2. **Download the official Ollama Linux tarball into offline-copy**

   ```bash
   cd ~/offline-copy/ollama

   curl -fsSL https://github.com/ollama/ollama/releases/download/v0.30.0-rc23/ollama-linux-amd64.tar.zst -o ollama-linux-amd64.tar.zst
   ```

3. **(Optional) Verify file size and checksum**

   ```bash
   ls -lh ollama-linux-amd64.tar.zst
   sha256sum ollama-linux-amd64.tar.zst > ollama-linux-amd64.tar.zst.sha256
   ```

   Keep the `.sha256` file in the same folder for later verification on the air‑gapped system.

---

## 3. Temporarily install Ollama on the online VM to fetch models

We’ll install Ollama on the **online VM** so you can download models once, then copy them into `offline-copy`. If you don’t want Ollama to stay installed on the VM, you can remove it later.

4. **Extract Ollama to `/usr` (online VM)**

   ```bash
   # Still on the online VM
   sudo tar -x --zstd -f ~/offline-copy/ollama/ollama-linux-amd64.tar.zst -C /usr
   ```

5. **Enable and start the Ollama service**

   ```bash
   sudo /usr/bin/ollama serve &
   ```

   Or, if the tarball includes a systemd unit (depends on version), you might use:

   ```bash
   sudo systemctl enable ollama
   sudo systemctl start ollama
   ```

6. **Pull the models you want (online)**

   Example:

   ```bash
   ollama pull llama3
   ollama pull mistral
   ```

   Models are stored under:

   ```bash
   ~/.ollama
   ```

7. **Copy the models into offline-copy**

   ```bash
   cd ~
   cp -a .ollama ~/offline-copy/ollama-models
   ```

   Now `~/offline-copy/ollama-models/.ollama` contains all your pulled models and metadata.

---

## 4. Prepare OpenWebUI Docker image and dependencies (online VM)

8. **Install Docker on the online VM (if not already)**

   ```bash
   sudo apt update
   sudo apt install -y docker.io
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

9. **Pull the OpenWebUI image**

   Example (adjust tag if needed):

   ```bash
   sudo docker pull ghcr.io/open-webui/open-webui:main
   ```

10. **Save the OpenWebUI image into offline-copy**

   ```bash
   cd ~/offline-copy/docker-images

   sudo docker save ghcr.io/open-webui/open-webui:main \
     -o openwebui-main.tar
   ```

11. **(Optional) Save any other images you need**

   If OpenWebUI uses additional images (e.g., reverse proxy, helper containers), save them the same way:

   ```bash
   sudo docker save IMAGE_NAME:TAG -o IMAGE_NAME-TAG.tar
   ```

---

## 5. Add helper scripts for the air‑gapped GPU host

12. **Create an Ollama install script for the air‑gapped host**

   `~/offline-copy/scripts/install_ollama_airgapped.sh`:

   ```bash
   cat > ~/offline-copy/scripts/install_ollama_airgapped.sh << 'EOF'
   #!/usr/bin/env bash
   set -e

   OFFLINE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

   echo "[*] Installing Ollama from tar.zst..."
   sudo tar -x --zstd -f "$OFFLINE_DIR/ollama/ollama-linux-amd64.tar.zst" -C /usr

   echo "[*] Restoring models to ~/.ollama..."
   mkdir -p "$HOME"
   if [ -d "$OFFLINE_DIR/ollama-models/.ollama" ]; then
     cp -a "$OFFLINE_DIR/ollama-models/.ollama" "$HOME/"
   fi

   echo "[*] Starting Ollama server..."
   /usr/bin/ollama serve &
   echo "[*] Done. Ollama is running on port 11434."
   EOF

   chmod +x ~/offline-copy/scripts/install_ollama_airgapped.sh
   ```

13. **Create an OpenWebUI Docker run script for the air‑gapped host**

   `~/offline-copy/scripts/run_openwebui_docker.sh`:

   ```bash
   cat > ~/offline-copy/scripts/run_openwebui_docker.sh << 'EOF'
   #!/usr/bin/env bash
   set -e

   OFFLINE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

   echo "[*] Loading OpenWebUI Docker image..."
   sudo docker load -i "$OFFLINE_DIR/docker-images/openwebui-main.tar"

   echo "[*] Running OpenWebUI container..."
   sudo docker run -d \
     --name openwebui \
     -p 3000:8080 \
     -v openwebui-data:/app/backend/data \
     -v /var/run/ollama.sock:/var/run/ollama.sock \
     -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
     ghcr.io/open-webui/open-webui:main

   echo "[*] OpenWebUI is now running on http://localhost:3000"
   echo "[*] It is configured to talk to Ollama on the host."
   EOF

   chmod +x ~/offline-copy/scripts/run_openwebui_docker.sh
   ```

   Notes:

   - `-v /var/run/ollama.sock:/var/run/ollama.sock` lets the container talk directly to the host’s Ollama socket.  
   - `OLLAMA_BASE_URL` is set to the host’s Ollama HTTP endpoint (`127.0.0.1:11434`)—OpenWebUI supports both socket and HTTP.

---

## 6. Copy offline-copy to USB and AV-scan

14. **Copy the entire offline-copy folder to your USB drive**

   ```bash
   # Example: USB mounted at /media/$USER/OFFLINE_USB
   cp -a ~/offline-copy /media/$USER/OFFLINE_USB/
   sync
   ```

15. **Run your AV scan on the USB contents**  
   (Use your organization’s standard tools/procedure.)

---

## 7. On the air‑gapped GPU Ubuntu system: prepare environment

16. **Copy offline-copy from USB to the air‑gapped host**

   ```bash
   # Example: USB mounted at /media/$USER/OFFLINE_USB
   cp -a /media/$USER/OFFLINE_USB/offline-copy ~/
   ```

17. **(Optional but recommended) Verify the Ollama tarball checksum**

   ```bash
   cd ~/offline-copy/ollama
   sha256sum -c ollama-linux-amd64.tar.zst.sha256
   ```

18. **Install Docker on the air‑gapped host (from local packages if needed)**

   If you can’t use `apt` online, you’ll need `.deb` packages mirrored beforehand; but if this host had access when first installed, you can:

   ```bash
   sudo apt update
   sudo apt install -y docker.io
   sudo systemctl enable docker
   sudo systemctl start docker
   ```

19. **Ensure GPU drivers / CUDA / ROCm are installed**  
   (This is environment‑specific; once installed, Ollama will use the GPU if supported.)

---

## 8. Install Ollama and restore models on the air‑gapped host

20. **Run the Ollama install script**

   ```bash
   cd ~/offline-copy/scripts
   ./install_ollama_airgapped.sh
   ```

   This will:

   - Extract `ollama-linux-amd64.tar.zst` to `/usr`  
   - Restore `~/.ollama` from `offline-copy/ollama-models`  
   - Start `ollama serve` on port `11434`  

21. **Confirm Ollama is running**

   ```bash
   ps aux | grep ollama
   curl http://127.0.0.1:11434/api/tags
   ```

---

## 9. Load OpenWebUI Docker image and run container

22. **Run the OpenWebUI Docker script**

   ```bash
   cd ~/offline-copy/scripts
   ./run_openwebui_docker.sh
   ```

   This will:

   - `docker load` the OpenWebUI image from `docker-images/openwebui-main.tar`  
   - Start the container on port `3000`  
   - Mount `/var/run/ollama.sock` from the host  
   - Set `OLLAMA_BASE_URL` to `http://127.0.0.1:11434`  

23. **Verify OpenWebUI is up and integrated with Ollama**

   - On the air‑gapped host, open a browser and go to:  
     `http://localhost:3000`  
   - In OpenWebUI’s settings, confirm the Ollama backend is reachable and models are listed.

---

## 10. Optional cleanup on the online VM

If you don’t want Ollama to remain installed on the online VM:

```bash
sudo pkill ollama || true
rm -rf ~/.ollama
# If you want to remove binaries (careful if shared):
# sudo rm -f /usr/bin/ollama
```

## 11. Changes i had to lookout for 
https://github.com/ollama/ollama/releases/download/v0.30.0-rc23/ollama-linux-amd64.tar.zst < Latest release
```bash
sudo mkdir -p /usr/local/bin
sudo mkdir -p /usr/local/lib/ollama
sudo tar --zstd -xf ollama-linux-amd64.tar.zst -C /usr/local
```

This gives you:

    /usr/local/bin/ollama

    /usr/local/lib/ollama/* (GPU libs)

```bash
sudo nano /etc/systemd/system/ollama.service
```

[Unit]
Description=Ollama LLM Service
After=network.target

[Service]
ExecStart=/usr/local/bin/ollama serve
Restart=always
RestartSec=3
Environment=OLLAMA_HOST=0.0.0.0
Environment=OLLAMA_PORT=11434
Environment="HOME=/usr/share/ollama"
WorkingDirectory=/usr/local/bin

[Install]
WantedBy=multi-user.target

We have to allow it to run on 0.0.0.0 instead of 127.0.0.1

```
docker run -d \
  --name open-webui \
  --add-host=host-gateway:host-gateway \
  -e OLLAMA_BASE_URL=http://host-gateway:11434 \
  -v open-webui:/app/backend/data \
  -p 3000:8080 \
  ghcr.io/open-webui/open-webui:main
```

# Get your container ID/name, then:
```
docker exec -it open-webui curl http://host-gateway:11434/api/tags
```

You can confirm the ollama user's home dir with:
```bashgetent passwd ollama```
# Should show something like: ollama:x:...::/usr/share/ollama:/bin/false



