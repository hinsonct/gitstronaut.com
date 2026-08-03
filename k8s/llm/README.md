# Self-hosted LLM (Ollama + Open WebUI)

Runs a text-only LLM on the RTX 2070 Mobile GPU in the `hinson-server` k3s
node, with Open WebUI as a local chat frontend. Starts local-network-only;
can be exposed on gitstronaut.com later via a new Ingress host + Cloudflare
Tunnel/DNS entry, the same pattern used for the main site.

## 0. Prerequisite: GPU is not usable by containers yet

Confirmed on this node: the RTX 2070 Mobile is present (`lspci`), but no
NVIDIA driver is installed (`nvidia-smi` not found, no `nvidia` kernel
module loaded, no `dpkg` driver package — only firmware). None of the
manifests here will schedule successfully until this is fixed. This is a
host-level change (sudo, likely a reboot) — do it deliberately, not as part
of an unattended script.

```bash
# 1. Install the proprietary driver (Ubuntu 26.04 LTS)
sudo apt update
sudo ubuntu-drivers install nvidia   # or: sudo apt install nvidia-driver-<latest>
sudo reboot
# after reboot, confirm:
nvidia-smi

# 2. Install nvidia-container-toolkit so containerd can see the GPU
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=containerd --config=/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
sudo systemctl restart k3s

# 3. Register the "nvidia" RuntimeClass so pods can request it
kubectl apply -f - <<'EOF'
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
EOF

# 4. Install the device plugin (exposes nvidia.com/gpu as a schedulable resource)
kubectl apply -f k8s/llm/nvidia-device-plugin.yaml
kubectl -n kube-system get pods -l name=nvidia-device-plugin-ds   # should go Running
kubectl describe node hinson-server | grep -A3 'Capacity:\|Allocatable:' # should list nvidia.com/gpu
```

## 1. Deploy Ollama + Open WebUI

```bash
kubectl apply -f k8s/llm/namespace.yaml
kubectl apply -f k8s/llm/ollama-pvc.yaml -f k8s/llm/ollama-deployment.yaml -f k8s/llm/ollama-service.yaml
kubectl apply -f k8s/llm/open-webui-pvc.yaml -f k8s/llm/open-webui-deployment.yaml -f k8s/llm/open-webui-service.yaml
kubectl -n llm get pods -w
```

Open WebUI is reachable on the LAN at `http://172.20.18.60:30880` (NodePort
30880) once both pods are Running — no public exposure yet. First account
created there becomes the admin (`WEBUI_AUTH=true` is set, so it's not an
open signup page to the internet — but it *is* reachable to anyone on your
LAN until this moves behind the tunnel with proper auth).

## 2. Pull a model

Models live on the `ollama-models` PVC (100Gi, local-path). With 8GB of
VRAM, models in the 7B–14B range at Q4/Q5 quantization fit well and leave
headroom for context. Pull from inside the pod:

```bash
kubectl -n llm exec -it deploy/ollama -- ollama pull llama3.1:8b
# or something closer to what you were running in LM Studio, e.g.:
kubectl -n llm exec -it deploy/ollama -- ollama pull qwen2.5:14b-instruct-q4_K_M
```

Then select the model in Open WebUI, or hit the API directly:

```bash
curl http://172.20.18.60:30880/ollama/api/generate -d '{"model": "llama3.1:8b", "prompt": "hello"}'
```

## 3. Expanding beyond Ollama later

Open WebUI isn't locked to a single backend — it can register multiple
OpenAI-API-compatible "Connections" at once. To add e.g. a llama.cpp or
vLLM server later:

1. Add a new Deployment + Service in this `llm` namespace (copy the
   `ollama-*.yaml` pattern; give it its own GPU request — note this node
   only has one GPU, so a second GPU-bound deployment will queue behind
   whichever pod holds it unless you deliberately time-share or swap them).
2. Add its base URL as a new Connection in Open WebUI's admin settings.

## 4. Exposing this on gitstronaut.com (later, not yet)

When ready: add a `k8s/llm/ingress.yaml` with a new host (e.g.
`chat.gitstronaut.com`) routed to the `open-webui` Service — same pattern
as `k8s/ingress.yaml` — then add the DNS/tunnel route in Cloudflare like
was done for `www`/`test` in the site's ingress history. Before doing
that: this GPU has no request queuing/rate-limiting built in, so a public
endpoint needs auth (Open WebUI's built-in accounts, already on by
default here) and ideally a request-rate limit at the Ingress, since
concurrent requests will just serialize behind the single GPU and could
make the whole thing time out for everyone.
