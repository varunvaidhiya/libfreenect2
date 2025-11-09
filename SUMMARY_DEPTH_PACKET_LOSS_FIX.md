# Summary: Depth Packet Loss Fix for Raspberry Pi

## Problem
You're experiencing depth packet loss on Raspberry Pi with errors like:
```
[Debug] [DepthPacketStreamParser] not all subsequences received 0
[Info] [DepthPacketStreamParser] 30 packets were lost
```

## Root Cause
**USB bandwidth limitations** on Raspberry Pi. Kinect v2 requires ~200-300 MB/s USB 3.0 bandwidth, and the Pi's USB controller cannot reliably handle this, causing packets to be dropped at the hardware level before they reach the parser.

## Quick Fix (Most Effective)
**Disable RGB stream** to reduce bandwidth by ~50%:
```bash
./build/bin/Protonect cpu -noviewer -norgb
```

## Changes Made

### 1. Code Improvements
- **`src/libfreenect2.cpp`**: Added ARM-specific defaults that automatically increase USB transfer pools from 60 to 120 on Raspberry Pi
- **`src/depth_packet_stream_parser.cpp`**: Enhanced error logging to show:
  - How many subsequences were received (e.g., "3/10")
  - Packet loss percentage
  - Actionable warnings when loss is high

### 2. Documentation
- **`docs/troubleshooting_depth_packet_loss.md`**: Comprehensive troubleshooting guide
- **`docs/depth_packet_loss_analysis.md`**: Technical analysis of the issue
- **`docs/technical.md`**: Updated with quick reference

## Recommended Solutions (In Order)

1. **Disable RGB stream** (`-norgb`) - Most effective, reduces bandwidth by 50%
2. **Increase USB transfer pools** - Already done automatically on ARM, but can be tuned via environment variables
3. **Increase USB kernel buffers** - System-level tuning (see troubleshooting guide)
4. **Verify USB 3.0 connection** - Ensure Kinect is on USB 3.0 port
5. **Use powered USB 3.0 hub** - Ensures stable power delivery

## Testing
After rebuilding, test with:
```bash
cd build
make -j$(nproc)
./bin/Protonect cpu -noviewer -norgb
```

Monitor for packet loss messages. You should see:
- **Good**: < 5 packets lost per 100 frames
- **Acceptable**: 5-30 packets lost per 100 frames
- **Poor**: 30+ packets lost (indicates USB bandwidth issues)

## Environment Variables
You can further tune USB transfer pools:
```bash
export LIBFREENECT2_IR_TRANSFERS=120  # Default is now 120 on ARM
export LIBFREENECT2_IR_PACKETS=8      # Default, some Pi models may support 16
export LIBFREENECT2_RGB_TRANSFERS=20  # Only if using RGB stream
./build/bin/Protonect cpu -noviewer -norgb
```

## Next Steps
1. Rebuild the project: `cd build && make -j$(nproc)`
2. Test with `-norgb` flag: `./bin/Protonect cpu -noviewer -norgb`
3. Monitor packet loss in logs
4. If loss persists, follow the troubleshooting guide: `docs/troubleshooting_depth_packet_loss.md`

## Notes
- The code changes are backward compatible - they improve defaults for ARM and enhance diagnostics
- RGB stream disabling is the single most effective solution
- USB 3.0 connection is required for Kinect v2
- Some packet loss (< 5%) is acceptable for most applications

## Files Changed
- `src/libfreenect2.cpp` - ARM-specific USB transfer pool defaults
- `src/depth_packet_stream_parser.cpp` - Enhanced error logging
- `docs/troubleshooting_depth_packet_loss.md` - New troubleshooting guide
- `docs/depth_packet_loss_analysis.md` - New technical analysis
- `docs/technical.md` - Updated with quick reference

