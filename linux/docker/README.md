# Pre-built pod image (RunPod)

**Status: written on Windows and NOT built or tested yet.** No Docker exists on this PC. The
Dockerfile was checked by reading it only, and the script changes only with `bash -n` and a stubbed
image directory. The first build and the first pod are the real test (see "First-run checks" at the
end).

## What it is

`linux/setup-pod.sh` spends 10-15 minutes on every new pod installing toolchains. The image
`linux/docker/Dockerfile` installs them once, at build time. It is based on the template our pods
already use, `runpod/pytorch:2.8.0-py3.11-cuda12.8.1-cudnn-devel-ubuntu22.04`.

| In the image | Path |
|---|---|
| setup-pod.sh's apt packages (build-essential, tmux, rsync, jq, zstd, pigz, ...) | system |
| rustup, minimal profile, toolchain `1.98.1` (the PC's rustc) | `/opt/enclosure/opt/{cargo,rustup}` |
| crates.io crates of enclosure-bot, game-import, extconv (`cargo fetch --locked`) | `/opt/enclosure/opt/cargo/registry` |
| Node.js 24 LTS, checked against nodejs.org's SHA-256 | `/opt/enclosure/opt/node` |
| venv with `pip install -r linux/requirements.txt` (torch 2.10.0+cu128) | `/opt/enclosure/venv` |
| versions and input hashes | `/opt/enclosure/IMAGE-INFO.env` |

The image paths are outside `/workspace`, because RunPod mounts the pod volume over `/workspace`.

**Not in the image:**
- project sources, checkpoints and data;
- Rust binaries;
- credentials of any kind.

The build context is a whitelist (`Dockerfile.dockerignore`): `linux/requirements.txt` and the
Cargo manifests and lockfiles only. These are bind-mounted during the build and never copied into a
layer. The image keeps the base image's `/start.sh` (SSH with `PUBLIC_KEY`, Jupyter).

**How the scripts use it.**
- `linux/pod-env.sh` checks for `/opt/enclosure/IMAGE-INFO.env` and
  `/opt/enclosure/venv/bin/python`. If both exist, it points `OPT`, `VENV`, `CARGO_HOME` and
  `RUSTUP_HOME` at the image. Otherwise everything stays under `/workspace`, as before.
- `setup-pod.sh` then finds the apt packages, Node and the venv already installed and skips them.
- It uses the image's toolchain without contacting rustup.
- It runs `build-rust.sh` with `OFFLINE=1` when the pod's three `Cargo.lock` files match the ones
  the image fetched.
- The secret scan, manifest check, botSourceHash check and GPU smoke test are unchanged.
- `ENCLOSURE_NO_IMAGE=1` forces the old `/workspace` layout, even on an image pod.

**Why the Rust binaries are not baked in.**
- `rust/enclosure-bot` changes often, and `build-rust.sh` checks that botSourceHash matches the
  uploaded sources (`EXPECT_BOT_SOURCE_HASH` from `pack-info.env`).
- A baked binary would be stale after the next code change. It could also no longer be traced to
  the uploaded code.
- The expensive, rarely changing parts are cached instead: the toolchain and the crates.
- So a fresh pod still compiles the three crates, offline, typically 1-3 minutes with LTO (an
  estimate; measure it on the first pod). After that, `linux-bin/` and `linux-build/` on the volume
  make reruns instant.

**When to rebuild the image.**
- `linux/requirements.txt` changed. Without a rebuild, setup-pod.sh warns and pip-installs into the
  image venv on the container disk, which is lost at every restart. It works, but slowly.
- A `Cargo.lock` changed. Without a rebuild, cargo downloads crates online, as before.
- You want another toolchain or Node release.

## 0. Make the minimal build context (PC)

```
powershell -NoProfile -ExecutionPolicy Bypass -File linux\docker\export-build-context.ps1
```

- It writes `work\pod-image-context\` with 13 files plus `.gitattributes`.
- Every path is checked against `linux/pod-denylist.txt`. Bytes are unchanged, so the hashes match
  the pod's uploaded files.
- `-ListOnly` shows the file list and writes nothing. Re-run it after `requirements.txt` or a
  `Cargo.lock` changes.

## Route A: Docker Desktop on this PC (Windows 11 Home)

**What you must install or enable yourself:**

1. **Virtualization.** Check Task Manager → Performance → CPU → "Virtualization: Enabled". If it
   says Disabled, enable Intel VT-x or AMD SVM in the BIOS/UEFI.
2. **WSL 2.** It is required on Windows Home, where Hyper-V is not available. In an
   **administrator** PowerShell run `wsl --install --no-distribution`, then reboot. Check with
   `wsl --status` (default version 2).
3. **Docker Desktop.** Install it from docker.com and choose "Use WSL 2 based engine". It is free
   for personal use.
4. **Disk.** About 60-80 GB free on C:, because Docker's WSL disk grows with the image (~25-30 GB
   unpacked, an estimate) and its build cache. Afterwards, `docker system prune -a` frees it.

**Build and push** (PowerShell):

```
cd <project>\work\pod-image-context
docker build -f linux/docker/Dockerfile -t <you>/enclosure-pod:2026MMDD .
docker login                          # you log in yourself (Docker Hub; or: docker login ghcr.io)
docker push <you>/enclosure-pod:2026MMDD
```

- You can also build from the project root with the same command. The whitelist keeps everything
  else out of the context.
- **Caveat: upload size.** The push uploads the whole image, base layers included (~10-15 GB
  compressed, an estimate), over your home connection. That can take hours. Route B avoids it.
- **Docker Hub:** anonymous pulls are rate-limited per IP, and RunPod hosts may share IPs. A
  public Docker Hub repository usually works; ghcr.io has no such per-IP limit for public images.

## Route B: GitHub Actions to ghcr.io (no upload from home)

1. On github.com, create a new **private** repository, for example `enclosure-pod-image`.
2. Push the context directory to it. It must contain only the exported files, **never** the whole
   project (`work/meaf` holds credentials).

   ```
   cd <project>\work\pod-image-context
   git init -b main
   git add -A
   git commit -m "pod image context"
   git remote add origin https://github.com/<you>/enclosure-pod-image.git
   git push -u origin main               # you authenticate yourself (Git Credential Manager)
   ```

   The export already put the workflow at `.github/workflows/pod-image.yml` (a copy of
   `linux/docker/github-workflow-pod-image.yml`).
3. **Run the workflow.** Pushing to `main` starts it, or use Actions → pod-image → Run workflow,
   with an optional extra tag.
   - It frees runner disk space, builds, prints `IMAGE-INFO.env`, and pushes
     `ghcr.io/<you>/enclosure-pod:<YYYYMMDD>-<commit>` and `:latest`.
   - It uses only the repository's own `GITHUB_TOKEN` (`packages: write`). There are no other
     secrets.
   - Expect roughly 20-40 minutes (unmeasured).
4. **Choose the package visibility.** Go to github.com → your profile → Packages → enclosure-pod →
   Package settings.
   - **Public** (your decision): the image contains no code, data or secrets, only public
     toolchains and package versions. RunPod then pulls it without credentials. GitHub Packages
     bills only private storage and transfer.
   - **Private:** RunPod needs registry credentials. Create a GitHub fine-grained or classic token
     with `read:packages` **yourself**, and add it in RunPod → Settings → Container Registry Auth.
     Check GitHub's current Packages billing for a ~15 GB private image.

If the build fails with "no space left on device", the runner's disk is too small for this base
image. Re-run once; if it persists, see "Open risks".

## Use the image on RunPod

Create a template: RunPod console → Templates → New Template.

- **Container image:** `ghcr.io/<you>/enclosure-pod:<YYYYMMDD>-<commit>`.
  - Use a dated tag, not `latest`, so every pod gets the same image.
  - For full reproducibility, use the digest: `...enclosure-pod@sha256:...`.
- **Container registry credentials:** only for a private image (see above).
- **Container disk: 40 GB or more.** The unpacked image (~25-30 GB, an estimate) plus apt, crate
  extraction and logs need room.
- **Volume disk:** 120-300 GB, as in RUNPOD.md, **mounted at `/workspace`**.
- **Expose TCP port 22** for SSH and scp. HTTP 8888 (Jupyter) is optional.
- **Environment variables:** `PUBLIC_KEY` = your **public** SSH key (the `.pub` line). Nothing else.
  Never add tokens or passwords. RunPod also adds the account's SSH public key if one is set in
  Settings.
- **Start command:** leave it empty, so the base image's `/start.sh` runs.
- **Deploy filter:** a host driver that supports CUDA 12.8 (R570 or newer). Use the console's CUDA
  version filter (12.8+) when deploying.

Then follow RUNPOD.md from step 3, with this order on a fresh pod: first the checklist below, then
the code tarball, then the data tarball.

## Verify a fresh pod (reject faulty Community hosts in about 2 minutes)

Do this before uploading the data tarball. If any step fails, or a number is far off: **Stop →
Terminate** and deploy another pod.

**1. The GPU is visible and idle** (seconds):

```bash
nvidia-smi
nvidia-smi --query-gpu=name,driver_version,memory.total,memory.used,temperature.gpu,pcie.link.gen.current,pcie.link.width.current --format=csv
cat /opt/enclosure/IMAGE-INFO.env
```

Check:
- the expected GPU and memory;
- driver 570 or newer;
- memory.used near 0;
- a sane temperature;
- no ERR! fields;
- the PCIe link at full width (x16). It may show a lower gen while idle.

**2. torch CUDA initializes, fp64 matches the CPU, and a 15 s fp16 matmul runs** (~30 s):

```bash
/opt/enclosure/venv/bin/python - <<'PY'
import time, torch
assert torch.cuda.is_available(), "CUDA not available"
d = torch.device("cuda:0")
print(torch.__version__, torch.version.cuda, torch.cuda.get_device_name(0))
a, b = torch.randn(512, 512, dtype=torch.float64), torch.randn(512, 512, dtype=torch.float64)
err = (a @ b - (a.to(d) @ b.to(d)).cpu()).abs().max().item()
assert err < 1e-9, f"fp64 mismatch {err}"
x = torch.randn(8192, 8192, device=d, dtype=torch.float16); y = torch.randn_like(x)
for _ in range(3): x @ y
torch.cuda.synchronize(); n, t0 = 0, time.time()
while time.time() - t0 < 15:
    for _ in range(10): x @ y
    torch.cuda.synchronize(); n += 10
dt = time.time() - t0
print(f"fp16 {2 * 8192**3 * n / dt / 1e12:.1f} TFLOP/s over {dt:.0f} s; fp64 check ok")
PY
```

Compare the TFLOP/s with earlier pods of the same GPU type, and write the value down per pod. A
value far below the usual one (e.g. under ~70 %) suggests throttling or a bad host. Any CUDA error
or a hang means reject.

**3. Upload only the code tarball and run setup without the Rust build** (~1 min):

Upload the code tarball (RUNPOD.md step 3), extract it (step 4), then run:

```bash
bash /workspace/enclosure-engine/linux/setup-pod.sh --skip-build
```

This runs:
- the secret scan;
- the code manifest check;
- "apt packages present", "Rust toolchain 1.98.1 from the image", "venv requirements up to date";
- the GPU smoke test: fp64 check, fp16 TFLOP/s, and `infer_server --fast` on pre10 (RDY1
  handshake, one request, clean shutdown).

Expect "GPU smoke test OK".

**4. Keep the pod.** Upload and extract the data tarball, then run setup again with the build:

```bash
bash /workspace/enclosure-engine/linux/setup-pod.sh
```

The log should say "Cargo.lock files match the image: building offline". Then continue with
RUNPOD.md step 6.

## First-run checks (the image has never been built)

1. **Build log.**
   - `cargo fetch --locked` succeeds with the stub sources.
   - The Node LTS lookup works.
   - The torch import line shows `2.10.0+cu128 cuda 12.8`.
   - `IMAGE-INFO.env` is printed. Note the image size (`docker image ls`).
2. **On the first pod.**
   - `setup-pod.sh` prints "pre-built image /opt/enclosure ...".
   - It skips apt, rustup, Node and pip.
   - The offline build succeeds.
   - `EXPECT_BOT_SOURCE_HASH` matches.
3. **Timing.** Measure from deploy to "setup-pod.sh OK": image pull, the setup without the build,
   and the Rust build. Record it with the pod type. The 1-2 minute target is not measured yet.

## Open risks

- **Image size.** The base image already carries CUDA devel, cuDNN and a system torch 2.8 that we
  do not use. Our venv adds ~7 GB.
  - Pull and extract time on a Community host depends on its bandwidth. A first pull can take
    several minutes, and RunPod's cached copy of the template's base layers may not be reused
    after the image is re-pushed to another registry.
  - Possible shrink, untested: `--build-arg BASE_IMAGE=` a smaller RunPod base without torch. It
    must keep `/start.sh`, SSH and python3.
- **CUDA/driver.** torch 2.10.0+cu128 ships its own CUDA 12.8 runtime and needs a driver of R570
  or newer. The base image's CUDA toolkit does not matter for it.
  - Hosts with older drivers fail at step 2.
  - RTX 5090 (Blackwell) needs the cu128 wheels, which these are.
- **GitHub runner disk.** A ~25-30 GB image is close to the limit of a standard runner even after
  the cleanup step. If it fails, use a larger runner or build with Route A.
- **Python version.** The venv uses the base image's `python3`, as setup-pod.sh does. Check the
  version in `IMAGE-INFO.env`: below 3.11, requirements.txt picks numpy 2.2.6 instead of 2.4.3.
  To force a version, use `--build-arg BASE_PYTHON=python3.11`.
- **Node.** "Newest 24 LTS at build time" is fixed in the image. Use `--build-arg
  NODE_VERSION=v24.x.y` for an exact pin.
- **`linux/jobs/selfplay-batches.sh`** hardcodes `$WS/venv/bin/python`, so it does not find the
  image venv (left unchanged here). On an image pod, change its `PY=` line to
  `/opt/enclosure/venv/bin/python`, or have it fall back to that path when `$WS/venv` is missing.
