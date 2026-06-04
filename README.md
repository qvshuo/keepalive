# keepalive
Oracle Cloud VPS Keep-Alive Script

```shell
mkdir -p ~/.config/systemd/user ~/.local/bin

cat > ~/.config/systemd/user/keepalive.timer <<'EOF'
[Unit]
Description=Keepalive fixed timer

[Timer]
OnCalendar=*-*-* 03:00:00
AccuracySec=1min
Persistent=true
Unit=keepalive.service

[Install]
WantedBy=timers.target
EOF

cat > ~/.config/systemd/user/keepalive.service <<'EOF'
[Unit]
Description=Keepalive workload pipeline

[Service]
Type=oneshot
ExecStart=%h/.local/bin/keepalive.sh
TimeoutStartSec=4h
StandardOutput=journal
StandardError=journal
EOF

cat > ~/.local/bin/keepalive.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

log() {
  echo "[$(date '+%F %T')] $*"
}

LOCKFILE="/tmp/keepalive.lock"
exec 9>"$LOCKFILE"
if ! flock -n 9; then
  log "another keepalive instance is already running, exit"
  exit 0
fi

SLEEP_SEC=$((RANDOM % 3601))
log "stage0: sleep ${SLEEP_SEC}s before workload"
sleep "$SLEEP_SEC"

log "stage1: network streaming for 5 minutes"
python3 - <<'PY'
import urllib.request
import random
import time

urls = [
    "https://releases.ubuntu.com/latest/ubuntu.iso",
    "https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian.iso",
    "https://download.fedoraproject.org/pub/fedora/linux/releases/latest/Workstation/x86_64/iso/Fedora.iso",
]

start = time.time()
duration = 5 * 60
chunk_size = 1024 * 1024

while time.time() - start < duration:
    url = random.choice(urls)
    try:
        with urllib.request.urlopen(url, timeout=15) as resp:
            while time.time() - start < duration:
                chunk = resp.read(chunk_size)
                if not chunk:
                    break
    except Exception:
        time.sleep(2)

print(f"[python] stage1 done: elapsed={time.time() - start:.1f}s")
PY
log "stage1: done"

log "stage2+3: memory target ~50% used, keep MemAvailable >= 256MB; cpu busy segments target ~25%"
python3 - <<'PY'
import time
import math

def read_meminfo():
    mem = {}
    with open("/proc/meminfo") as f:
        for line in f:
            if ":" in line:
                k, v = line.split(":", 1)
                mem[k.strip()] = int(v.strip().split()[0]) * 1024
    return mem

def mb(x):
    return x / 1024 / 1024

mem = read_meminfo()
mem_total = mem["MemTotal"]
mem_avail = mem["MemAvailable"]
current_used = mem_total - mem_avail
target_used = int(mem_total * 0.50)

min_available = 256 * 1024 * 1024
need_alloc = max(0, target_used - current_used)
max_alloc_by_safety = max(0, mem_avail - min_available)
need_alloc = min(need_alloc, max_alloc_by_safety)

print(f"[python] MemTotal={mb(mem_total):.1f}MB")
print(f"[python] MemAvailable(before)={mb(mem_avail):.1f}MB")
print(f"[python] CurrentUsed(before)={mb(current_used):.1f}MB")
print(f"[python] TargetUsed={mb(target_used):.1f}MB")
print(f"[python] PlanAlloc={mb(need_alloc):.1f}MB")

bufs = []
steps = 10

if need_alloc > 0:
    step_bytes = need_alloc // steps
    remain = need_alloc % steps

    for i in range(steps):
        size = step_bytes + (remain if i == steps - 1 else 0)
        if size > 0:
            b = bytearray(size)
            for idx in range(0, len(b), 4096):
                b[idx] = 1
            bufs.append(b)

        mem_now = read_meminfo()
        alloc_total = sum(len(x) for x in bufs)
        print(
            f"[python] alloc step {i+1}/{steps}: "
            f"alloc_total={mb(alloc_total):.1f}MB, "
            f"MemAvailable={mb(mem_now['MemAvailable']):.1f}MB"
        )
        time.sleep(30)
else:
    print("[python] no extra allocation needed")
    time.sleep(5 * 60)

mem_after = read_meminfo()
used_after = mem_after["MemTotal"] - mem_after["MemAvailable"]
print(f"[python] CurrentUsed(after)={mb(used_after):.1f}MB, ratio={used_after / mem_after['MemTotal'] * 100:.1f}%")
print("[python] memory stage done, holding RSS")

busy_segments = [24 * 60, 24 * 60, 24 * 60]
sleep_segments = [9 * 60, 9 * 60]

busy_time = 1.0
idle_time = 3.0

def burn_for(seconds):
    end = time.time() + seconds
    seg_start = time.time()
    while time.time() < end:
        t0 = time.time()
        while time.time() - t0 < busy_time:
            acc = 0.0
            for i in range(1, 5000):
                acc += math.sqrt(i * i + 3)
        remain = end - time.time()
        if remain <= 0:
            break
        time.sleep(min(idle_time, remain))
    print(f"[python] busy segment done: elapsed={time.time() - seg_start:.1f}s")

for idx, seg in enumerate(busy_segments):
    print(f"[python] start busy segment {idx+1}: duration={seg}s, target cpu~25%")
    burn_for(seg)
    if idx < len(sleep_segments):
        slp = sleep_segments[idx]
        print(f"[python] start inter-segment sleep {idx+1}: duration={slp}s")
        time.sleep(slp)

print("[python] cpu stage done, releasing resources")
PY

log "all stages finished successfully"
EOF

chmod +x ~/.local/bin/keepalive.sh
systemctl --user daemon-reload
systemctl --user enable --now keepalive.timer
loginctl enable-linger "$USER"

echo "deployed."
systemctl --user status keepalive.timer --no-pager
```
