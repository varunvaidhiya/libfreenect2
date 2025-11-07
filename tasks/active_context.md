# Active Context

## Current Focus
- Stabilize Kinect v2 depth streaming on Raspberry Pi 5.
- Validate NEON-optimized CPU depth processor.

## Recent Changes
- Added NEON detection and optimized code paths in `cpu_depth_packet_processor.cpp`.
- Updated CMake to enable NEON flags on ARM builds.
- Seeded memory documentation structure.

## Next Steps
- Validate Protonect runtime on Pi with new build.
- Document USB tuning results and performance metrics.
- Capture lessons learned during testing.

## Open Risks / Issues
- Potential USB packet loss under sustained load.
- Thermal throttling on Raspberry Pi during extended runs.
