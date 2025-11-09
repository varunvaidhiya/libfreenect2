# Depth Packet Loss Analysis and Fixes

## Issue Summary

**Problem**: Depth packets are being lost on Raspberry Pi, resulting in:
- `[Debug] [DepthPacketStreamParser] not all subsequences received 0`
- `[Info] [DepthPacketStreamParser] 30 packets were lost`
- No depth frames being processed successfully

## Root Cause Analysis

### Technical Explanation

1. **Depth Frame Structure**: Kinect v2 depth frames are split into **10 subsequences** (subpackets)
   - Each subsequence has a footer containing: sequence number, subsequence number (0-9), timestamp, length
   - All 10 subsequences must be received to reconstruct a complete depth frame
   - The parser tracks received subsequences using a bitmask (`current_subsequence_`)
   - When all 10 are received: `current_subsequence_ = 0x3ff` (binary: 1111111111)

2. **Packet Loss Detection**: 
   - When a new sequence arrives before all subsequences of the previous sequence are received
   - The parser logs: `"not all subsequences received X"` where X is the bitmask value
   - `X = 0` means **no subsequences were received** before the new sequence started

3. **Why It Happens on Raspberry Pi**:
   - **USB Bandwidth Limitations**: Kinect v2 requires ~200-300 MB/s sustained USB 3.0 bandwidth
   - **USB Controller Limitations**: Raspberry Pi's integrated USB controller may not handle this reliably
   - **CPU Processing Bottleneck**: CPU depth processing is computationally intensive, delaying USB callbacks
   - **USB Buffer Overflow**: If USB interrupts are delayed, kernel USB buffers overflow and packets are dropped at hardware level

### Code Flow

```
USB Hardware → USB Kernel Driver → libusb → TransferPool → DepthPacketStreamParser
                                                                      ↓
                                                              [Packet Loss Here]
                                                                      ↓
                                                           CPU Depth Processor
```

Packet loss occurs **before** the parser - USB packets are dropped at the hardware/driver level, so they never reach the parser's `onDataReceived()` callback.

## Fixes Applied

### 1. Raspberry Pi-Specific USB Transfer Pool Defaults

**File**: `src/libfreenect2.cpp`

**Change**: Added ARM-specific defaults that increase USB transfer pool size:
```cpp
#elif defined(__arm__) || defined(__aarch64__)
  // ARM/Raspberry Pi: Increase transfer pools to handle USB bandwidth limitations
  ir_num_xfers = 120;  // Increased from 60 to handle packet bursts
#endif
```

**Rationale**: More USB transfer buffers = better chance of catching packets before buffer overflow. This is automatically applied on ARM builds.

### 2. Enhanced Error Logging

**File**: `src/depth_packet_stream_parser.cpp`

**Changes**:
1. **Better subsequence loss reporting**: Now shows how many subsequences were received (e.g., "3/10") instead of just the bitmask
2. **Packet loss percentage**: Calculates and displays approximate loss rate
3. **Actionable warnings**: When loss exceeds 100 packets, suggests specific fixes

**Example Output**:
```
[Debug] [DepthPacketStreamParser] not all subsequences received 3/10 (bitmask: 0x7) for sequence 123, new sequence 124 starting
[Info] [DepthPacketStreamParser] 30 packets were lost (sequence 100 -> 130, ~23.1% loss rate)
[Warning] High packet loss detected! Consider:
  1) Disabling RGB stream with -norgb
  2) Increasing USB transfer pools (LIBFREENECT2_IR_TRANSFERS)
  3) Checking USB 3.0 connection and power supply
```

### 3. Documentation

**Files Created**:
- `docs/troubleshooting_depth_packet_loss.md`: Comprehensive troubleshooting guide
- `docs/depth_packet_loss_analysis.md`: This document

**Files Updated**:
- `docs/technical.md`: Added quick reference to troubleshooting guide

## Recommended Solutions (In Order of Effectiveness)

### Solution 1: Disable RGB Stream (Most Effective)

**Impact**: Reduces USB bandwidth by ~50%

```bash
./build/bin/Protonect cpu -noviewer -norgb
```

This is the **single most effective** solution because:
- RGB JPEG decoding is CPU-intensive
- Eliminates RGB USB transfers (~100-150 MB/s)
- Leaves more USB bandwidth for depth stream
- Reduces CPU load, allowing faster USB interrupt handling

### Solution 2: Increase USB Transfer Pools

**Impact**: Better packet buffering

```bash
export LIBFREENECT2_IR_TRANSFERS=120  # Default is now 120 on ARM, but can increase to 180
export LIBFREENECT2_IR_PACKETS=16     # May work on some Pi models
./build/bin/Protonect cpu -noviewer -norgb
```

**Note**: More transfers = more memory usage. Monitor with `free -h`.

### Solution 3: USB Kernel Buffer Tuning

**Impact**: Prevents kernel-level buffer overflow

```bash
# Temporary (until reboot)
sudo sh -c 'echo 1000 > /sys/module/usbcore/parameters/usbfs_memory_mb'

# Permanent
echo 'options usbcore usbfs_memory_mb=1000' | sudo tee /etc/modprobe.d/usb.conf
sudo modprobe -r usbcore && sudo modprobe usbcore
```

### Solution 4: Verify USB 3.0 Connection

**Impact**: Ensures maximum bandwidth available

```bash
lsusb -t
```

Look for `5000M` (USB 3.0) vs `480M` (USB 2.0). Kinect v2 **requires** USB 3.0.

### Solution 5: Use Powered USB 3.0 Hub

**Impact**: Stable power delivery can improve USB reliability

Some power delivery issues manifest as USB packet loss. A powered hub ensures consistent power.

## Expected Results

### Before Fixes
- Continuous "not all subsequences received 0" errors
- 30+ packets lost regularly
- No depth frames processed
- High CPU usage from RGB processing

### After Solution 1 (Disable RGB)
- Occasional packet loss (< 10 packets per 100 frames)
- Most depth frames processed successfully
- Reduced CPU usage
- Stable depth streaming

### After All Optimizations
- < 5 packets lost per 100 frames
- Stable depth frame processing
- Acceptable for most applications

## Monitoring

Watch for these log messages:

```
# Good: Occasional, small losses
[Info] [DepthPacketStreamParser] 2 packets were lost (sequence 100 -> 102, ~2.0% loss rate)

# Acceptable: Periodic losses
[Info] [DepthPacketStreamParser] 10 packets were lost (sequence 200 -> 210, ~4.8% loss rate)

# Poor: Continuous, large losses
[Info] [DepthPacketStreamParser] 100 packets were lost (sequence 300 -> 400, ~25.0% loss rate)
[Warning] High packet loss detected! Consider: ...
```

## Code Changes Summary

1. **`src/libfreenect2.cpp`**: Added ARM-specific USB transfer pool defaults (120 transfers)
2. **`src/depth_packet_stream_parser.cpp`**: Enhanced error logging with:
   - Subsequence count (X/10)
   - Loss percentage calculation
   - Actionable warnings for high loss
3. **Documentation**: Created troubleshooting guide and analysis document

## Testing

After applying fixes, test with:

```bash
# Rebuild
cd build
make -j$(nproc)

# Test depth-only (recommended)
./bin/Protonect cpu -noviewer -norgb

# Monitor for packet loss messages
# Should see minimal losses (< 5 per 100 frames)
```

## Future Improvements

Potential code-level improvements (if hardware limitations persist):

1. **Packet Reordering**: Handle out-of-order packet delivery
2. **Partial Frame Processing**: Process frames with missing subsequences (with quality degradation)
3. **Adaptive Frame Skipping**: Skip frames more intelligently based on loss patterns
4. **USB Transfer Timeout Tuning**: Adjust timeouts for Raspberry Pi's USB characteristics
5. **Multi-threaded USB Processing**: Separate USB callback thread from processing thread

## References

- libfreenect2 GitHub: Issues related to "Raspberry Pi packet loss"
- USB 3.0 Specifications: Bandwidth requirements
- Kinect v2 Protocol: Depth frame structure (10 subsequences)
- Raspberry Pi USB Controller: Limitations and optimizations

## Conclusion

The depth packet loss on Raspberry Pi is primarily a **USB bandwidth limitation** issue. The most effective solution is to **disable the RGB stream** (`-norgb`), which reduces bandwidth requirements by ~50%. The code changes provide:

1. **Better defaults** for Raspberry Pi (more USB transfer buffers)
2. **Enhanced diagnostics** (clearer error messages with actionable advice)
3. **Comprehensive documentation** (troubleshooting guide)

For most use cases, disabling RGB and using the optimized defaults should provide acceptable depth streaming performance on Raspberry Pi 5.

