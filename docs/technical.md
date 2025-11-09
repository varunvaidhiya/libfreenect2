# Technical Notes

## Stack
- **Language:** C++14
- **Build System:** CMake
- **Dependencies:** libusb-1.0, libturbojpeg, GLFW (viewer), optional OpenCL/CUDA/OpenGL.
- **Target Platform:** Raspberry Pi 5 running Ubuntu 24.04 (64-bit).

## Environment Setup
```bash
sudo apt update
sudo apt install build-essential cmake pkg-config libusb-1.0-0-dev libturbojpeg0-dev libglfw3-dev -y
```

## Build Commands
```bash
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=$HOME/freenect2
make -j$(nproc)
make install
```

## NEON Configuration
- CMake automatically adds `-ftree-vectorize`.
- ARM64 builds gain `-O3 -march=native`.
- ARM32 builds add `-mfpu=neon -march=native`.

## Key Files
- `src/cpu_depth_packet_processor.cpp` (NEON code paths)
- `src/depth_packet_stream_parser.cpp`
- `CMakeLists.txt` (NEON flags)

## Testing
```bash
./build/bin/Protonect cpu -noviewer
./build/bin/Protonect cpu -noviewer -norgb
```

## Troubleshooting

### Depth Packet Loss on Raspberry Pi

If you encounter depth packet loss errors:
```
[Debug] [DepthPacketStreamParser] not all subsequences received 0
[Info] [DepthPacketStreamParser] 30 packets were lost
```

**See detailed troubleshooting guide**: `docs/troubleshooting_depth_packet_loss.md`

**Quick fixes**:
1. **Disable RGB stream** (reduces bandwidth by ~50%):
   ```bash
   ./build/bin/Protonect cpu -noviewer -norgb
   ```

2. **Increase USB transfer pools**:
   ```bash
   export LIBFREENECT2_IR_TRANSFERS=120
   ./build/bin/Protonect cpu -noviewer -norgb
   ```

3. **Increase USB kernel buffers**:
   ```bash
   sudo sh -c 'echo 1000 > /sys/module/usbcore/parameters/usbfs_memory_mb'
   ```

### General Issues
- Increase USB buffers via `/etc/modprobe.d/usb.conf` if packets drop.
- Use powered USB 3.0 hub to ensure sufficient power.
- Verify USB 3.0 connection: `lsusb -t` should show `5000M` (not `480M`).
