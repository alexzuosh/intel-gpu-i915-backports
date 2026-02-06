# Intel i915 GPU Debugger - Quick Start Guide

**Get debugging working in 5 minutes**

---

## 1️⃣ Enable Debugging (2 minutes)

```bash
# Enable GuC firmware logging
echo 2 > /sys/kernel/debug/dri/0/gt0/uc/guc_log_level

# Make hang detection more responsive (for debugging)
echo 500 > /sys/kernel/debug/dri/0/gt0/heartbeat_interval_ms

# (Optional) Stream GuC logs in real-time
tail -f /sys/kernel/debug/dri/0/gt0/uc/guc_log_dump &
```

---

## 2️⃣ Run Your GPU Application (1 minute)

```bash
# Start your GPU-using application
./your_gpu_app &
APP_PID=$!

# Wait for it to hang or complete
sleep 30

kill $APP_PID 2>/dev/null
```

---

## 3️⃣ Capture Error State (1 minute)

```bash
# Save GPU state snapshot
cat /sys/kernel/debug/dri/0/i915_error_state > /tmp/gpu_error.txt

# View first 100 lines
head -100 /tmp/gpu_error.txt
```

---

## 4️⃣ Analyze What Happened (1 minute)

```bash
# Check if GPU reset occurred
RESET_COUNT=$(cat /sys/class/drm/card0/error/reset_count)
echo "Reset count: $RESET_COUNT"

# Look for errors
grep -i "fault\|error\|hung" /tmp/gpu_error.txt

# See which engine hung
grep -A 5 "^Engine:" /tmp/gpu_error.txt

# Check active process
grep "comm\|pid\|uid" /tmp/gpu_error.txt
```

---

## Common Issues & Quick Fixes

### Issue: "No such file or directory"
```bash
# Debugfs may not be mounted
sudo mount -t debugfs none /sys/kernel/debug
```

### Issue: "Permission denied"
```bash
# Need root access
sudo bash  # or use sudo for each command
```

### Issue: "Can't find dri/0"
```bash
# Check available devices
ls /sys/kernel/debug/dri/
# Use the correct number (might be 1, 2, etc.)
```

---

## Production Settings (Minimal Overhead)

```bash
# Disable expensive debugging
echo -1 > /sys/kernel/debug/dri/0/gt0/uc/guc_log_level

# Increase heartbeat interval (less frequent checks)
echo 5000 > /sys/kernel/debug/dri/0/gt0/heartbeat_interval_ms
```

---

## Development Settings (Maximum Visibility)

```bash
# Enable verbose firmware logging
echo 3 > /sys/kernel/debug/dri/0/gt0/uc/guc_log_level

# Ultra-sensitive hang detection
echo 500 > /sys/kernel/debug/dri/0/gt0/heartbeat_interval_ms
```

---

## Key Files

| File | Purpose |
|------|---------|
| `/sys/kernel/debug/dri/0/i915_error_state` | Last GPU error snapshot |
| `/sys/kernel/debug/dri/0/gt0/uc/guc_log_dump` | Firmware logs |
| `/sys/class/drm/card0/error/reset_count` | How many times GPU reset |
| `/sys/kernel/debug/dri/0/gt0/heartbeat_interval_ms` | Hang detection tuning |

---

## Next Steps

- Full guide: See **08-Debugger-Support.md**
- Implementation: See **08b-Debugger-Implementation.md**
- Detailed scenarios: See **DEBUGGER-SUPPORT-SUMMARY.md**

