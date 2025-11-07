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
- Increase USB buffers via `/etc/modprobe.d/usb.conf` if packets drop.
- Use powered USB 3.0 hub to ensure sufficient power.
