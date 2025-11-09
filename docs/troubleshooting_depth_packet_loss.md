# Troubleshooting Depth Packet Loss on Raspberry Pi

## Problem Description

When running `Protonect cpu -noviewer` on Raspberry Pi, you may see:
```
[Debug] [DepthPacketStreamParser] not all subsequences received 0
[Debug] [DepthPacketStreamParser] skipping depth packet
[Info] [DepthPacketStreamParser] 30 packets were lost
```

## Root Cause

**USB Packet Loss at Hardware/Driver Level**: The Raspberry Pi's USB controller cannot reliably handle the high bandwidth requirements of Kinect v2 (~200-300 MB/s). Depth frames are split into 10 subsequences (subpackets). When USB packets are dropped:
- Not all subsequences arrive before a new frame sequence starts
- The parser detects incomplete frames (`current_subsequence_ = 0` means no subsequences received)
- Frames are skipped to maintain synchronization

## Why This Happens

1. **USB Bandwidth Limitations**: Kinect v2 requires sustained USB 3.0 bandwidth that Raspberry Pi may not provide reliably
2. **CPU Processing Speed**: CPU depth processing is computationally intensive, causing USB callbacks to be delayed
3. **USB Buffer Overflow**: If USB interrupts are delayed, kernel buffers overflow and packets are dropped
4. **USB Controller Limitations**: Raspberry Pi's integrated USB controller may have lower throughput than dedicated USB controllers

## Solutions (Ordered by Effectiveness)

### 1. Disable RGB Stream (Recommended - Reduces Bandwidth by ~50%)

**Quick Test**:
```bash
./build/bin/Protonect cpu -noviewer -norgb
```

This eliminates RGB JPEG decoding workload and reduces USB bandwidth by approximately 50%, significantly improving depth stream stability.

### 2. USB Transfer Pool Sizes (Already Optimized for ARM)

The default transfer pool size is automatically set to 70 on ARM/Raspberry Pi (optimized for USB controller capacity).

**If you see LIBUSB_ERROR_IO errors**, the transfer count may be too high:
```bash
# Reduce transfer count if you get LIBUSB_ERROR_IO errors
export LIBFREENECT2_IR_TRANSFERS=60  # Default on ARM: 70, reduce to 60 if needed
./build/bin/Protonect cpu -noviewer -norgb
```

**If you have stable USB 3.0 and want to try more transfers** (test carefully):
```bash
export LIBFREENECT2_IR_TRANSFERS=80  # Increase cautiously, monitor for errors
./build/bin/Protonect cpu -noviewer -norgb
```

**Note**: 
- More transfers = more memory usage and higher USB controller load
- Too many transfers (>100) cause LIBUSB_ERROR_IO on Raspberry Pi
- Default of 70 is a balance between packet buffering and USB controller capacity

### 3. USB System Tuning (Kernel Parameters)

Create or edit `/etc/modprobe.d/usb.conf`:
```bash
sudo nano /etc/modprobe.d/usb.conf
```

Add:
```
options usbcore usbfs_memory_mb=1000
```

Increase USB buffer sizes:
```bash
sudo sh -c 'echo 1000 > /sys/module/usbcore/parameters/usbfs_memory_mb'
```

**Permanent**: Add to `/etc/rc.local` or systemd service.

### 4. Verify USB 3.0 Connection

Check if Kinect is connected to USB 3.0 port:
```bash
lsusb -t
```

Look for `480M` (USB 2.0) vs `5000M` (USB 3.0). Kinect v2 requires USB 3.0.

### 5. Reduce CPU Processing Load

Disable expensive filters if not needed:
- Bilateral filter (enabled by default)
- Edge-aware filter

Modify your code to disable filters, or use a simpler processing pipeline.

### 6. Use Powered USB 3.0 Hub

Sometimes power delivery issues cause USB instability. Use a powered USB 3.0 hub between Pi and Kinect.

### 7. CPU Frequency Scaling

Ensure CPU is running at maximum frequency:
```bash
# Check current frequency
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq

# Set to performance governor
sudo cpupower frequency-set -g performance
```

### 8. Disable USB Auto-Suspend

Prevent USB devices from suspending:
```bash
# Create udev rule
sudo nano /etc/udev/rules.d/50-usb_power_save.rules
```

Add:
```
ACTION=="add", SUBSYSTEM=="usb", TEST=="power/control", ATTR{power/control}="on"
```

## Expected Behavior After Fixes

- **Good**: Occasional "packets were lost" messages (< 1 per 100 frames)
- **Acceptable**: Periodic losses (< 10 packets per 100 frames) with stable streaming
- **Poor**: Continuous "not all subsequences received" errors, no depth frames processed

## Monitoring Packet Loss

The parser logs packet loss every 30 frames or when loss exceeds 30 frames:
```
[Info] [DepthPacketStreamParser] X packets were lost
```

Monitor this value:
- **0-5 packets lost**: Excellent
- **5-30 packets lost**: Good, acceptable for most applications
- **30+ packets lost**: Poor, indicates USB bandwidth issues

## Code-Level Optimizations

If software-level fixes don't help, consider:

1. **Increase USB transfer timeout** (may help with delayed packets)
2. **Implement packet reordering** (handle out-of-order packets)
3. **Partial frame processing** (process frames with missing subsequences)
4. **Adaptive frame skipping** (skip frames more intelligently)

## Hardware Recommendations

For reliable Kinect v2 operation on Raspberry Pi:
- **Raspberry Pi 5** (recommended) - Better USB 3.0 support than Pi 4
- **Powered USB 3.0 hub** - Ensures stable power delivery
- **High-quality USB 3.0 cable** - Short cable (< 1m) reduces signal degradation
- **Adequate cooling** - Thermal throttling reduces USB performance

## Alternative Approaches

If packet loss persists:
1. **Use lower resolution** (if supported by your use case)
2. **Reduce frame rate** (process every Nth frame)
3. **Use OpenCL/GPU processing** (if available, reduces CPU load)
4. **Consider different depth sensor** (ASUS Xtion, Intel RealSense D400 series)

## Debugging

Enable detailed USB debugging:
```bash
export LIBUSB_DEBUG=3
./build/bin/Protonect cpu -noviewer -norgb 2>&1 | tee usb_debug.log
```

Check for USB errors:
- `LIBUSB_TRANSFER_OVERFLOW`: Buffer overflow
- `LIBUSB_TRANSFER_ERROR`: USB communication error
- `LIBUSB_TRANSFER_NO_DEVICE`: Device disconnected

## References

- libfreenect2 GitHub Issues: Search for "Raspberry Pi packet loss"
- USB 3.0 bandwidth requirements: ~200-300 MB/s for Kinect v2
- Depth frame structure: 10 subsequences per frame, ~33792 bytes per subsequence

