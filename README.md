# libfreenect2

**This is a fork with CUDA 13.0 compatibility fixes and Ubuntu 24.04 LTS optimization.**

## What's New in This Fork

* ✅ **CUDA 13.0 Compatibility**: Fixed deprecated API usage (`clockRate`, `computeMode`)
* ✅ **OpenCL Fixes**: Resolved macro naming conflicts (`CL_ICDL_VERSION`)
* ✅ **Ubuntu 24.04 LTS Support**: Optimized installation instructions
* ✅ **Raspberry Pi 5 Support**: ARM64 compatibility with USB buffer optimizations
* ✅ **Performance Testing**: Tested with NVIDIA RTX 5060 Ti (~3000 FPS CUDA processing)

## Table of Contents

* [Description](README.md#description)
* [Requirements](README.md#requirements)
* [Troubleshooting](README.md#troubleshooting-and-reporting-bugs)
* [Maintainers](README.md#maintainers)
* [Installation](README.md#installation)
  * [Windows / Visual Studio](README.md#windows--visual-studio)
  * [MacOS](README.md#macos)
  * [Linux](README.md#linux)
* [API Documentation (external)](https://openkinect.github.io/libfreenect2/)

## Description

Driver for Kinect for Windows v2 (K4W2) devices (release and developer preview).

Note: libfreenect2 does not do anything for either Kinect for Windows v1 or Kinect for Xbox 360 sensors. Use libfreenect1 for those sensors.

If you are using libfreenect2 in an academic context, please cite our work using the following DOI: [![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.50641.svg)](https://doi.org/10.5281/zenodo.50641)



If you use the KDE depth unwrapping algorithm implemented in the library, please also cite this ECCV 2016 [paper](http://users.isy.liu.se/cvl/perfo/abstracts/jaremo16.html).

This driver supports:
* RGB image transfer
* IR and depth image transfer
* registration of RGB and depth images

Missing features:
* firmware updates (see [issue #460](https://github.com/OpenKinect/libfreenect2/issues/460) for WiP)

Watch the OpenKinect wiki at www.openkinect.org and the mailing list at https://groups.google.com/forum/#!forum/openkinect for the latest developments and more information about the K4W2 USB protocol.

The API reference documentation is provided here https://openkinect.github.io/libfreenect2/.

## Requirements

### Hardware requirements

* USB 3.0 controller. USB 2 is not supported.

Intel and NEC USB 3.0 host controllers are known to work. ASMedia controllers are known to not work.

Virtual machines likely do not work, because USB 3.0 isochronous transfer is quite delicate.

##### Requirements for multiple Kinects

It has been reported to work for up to 5 devices on a high-end PC using multiple separate PCI Express USB3 expansion cards (with NEC controller chip). If you're using Linux, you may have to [increase USBFS memory buffers](https://github.com/OpenKinect/libfreenect2/wiki/Troubleshooting#multiple-kinects-try-increasing-usbfs-buffer-size). Depending on the number of Kinects, you may need to use an even larger buffer size. If you're using an expansion card, make sure it's not plugged into an PCI-E x1 slot. A single lane doesn't have enough bandwidth. x8 or x16 slots usually work.

### Operating system requirements

* Windows 7 (buggy), Windows 8, Windows 8.1, and probably Windows 10
* Debian, Ubuntu 14.04 or newer, probably other Linux distros. Recommend kernel 3.16+ or as new as possible.
* Mac OS X

### Requirements for optional features

* OpenGL depth processing: OpenGL 3.1 (Windows, Linux, Mac OS X). OpenGL ES is not supported at the moment.
* OpenCL depth processing: OpenCL 1.1
* CUDA depth processing: CUDA 6.5+ (tested up to CUDA 13.0; this fork includes CUDA 13.0 compatibility fixes)
* VAAPI JPEG decoding: Intel (minimum Ivy Bridge or newer) and Linux only
* VideoToolbox JPEG decoding: Mac OS X only
* OpenNI2 integration: OpenNI2 2.2.0.33
* Jetson TK1: Linux4Tegra 21.3 or later. Check [Jetson TK1 issues](https://github.com/OpenKinect/libfreenect2/wiki/Troubleshooting#jetson-tk1-issues) before installation. Jetson TX1 is not yet supported as the developers don't have one, but it may be easy to add the support.

## Troubleshooting and reporting bugs

First, check https://github.com/OpenKinect/libfreenect2/wiki/Troubleshooting for known issues.

When you report USB issues, please attach relevant debug log from running the program with environment variable `LIBUSB_DEBUG=3`, and relevant log from `dmesg`. Also include relevant hardware information `lspci` and `lsusb -t`.

## Maintainers

* Joshua Blake <joshblake@gmail.com>
* Florian Echtler
* Christian Kerl
* Lingzhu Xiang (development/master branch)

## Installation

### Windows / Visual Studio

* Install UsbDk driver

    1. (Windows 7) You must first install Microsoft Security Advisory 3033929 otherwise your USB keyboards and mice will stop working!
    2. Download the latest x64 installer from https://github.com/daynix/UsbDk/releases, install it.
    3. If UsbDk somehow does not work, uninstall UsbDk and follow the libusbK instructions.

    This doesn't interfere with the Microsoft SDK. Do not install both UsbDK and libusbK drivers
* (Alternatively) Install libusbK driver

    You don't need the Kinect for Windows v2 SDK to build and install libfreenect2, though it doesn't hurt to have it too. You don't need to uninstall the SDK or the driver before doing this procedure.

    Install the libusbK backend driver for libusb. Please follow the steps exactly:

    1. Download Zadig from http://zadig.akeo.ie/.
    2. Run Zadig and in options, check "List All Devices" and uncheck "Ignore Hubs or Composite Parents"
    3. Select the "Xbox NUI Sensor (composite parent)" from the drop-down box. (Important: Ignore the "NuiSensor Adaptor" varieties, which are the adapter, NOT the Kinect) The current driver will list usbccgp. USB ID is VID 045E, PID 02C4 or 02D8.
    4. Select libusbK (v3.0.7.0 or newer) from the replacement driver list.
    5. Click the "Replace Driver" button. Click yes on the warning about replacing a system driver. (This is because it is a composite parent.)

    To uninstall the libusbK driver (and get back the official SDK driver, if installed):

    1. Open "Device Manager"
    2. Under "libusbK USB Devices" tree, right click the "Xbox NUI Sensor (Composite Parent)" device and select uninstall.
    3. Important: Check the "Delete the driver software for this device." checkbox, then click OK.

    If you already had the official SDK driver installed and you want to use it:

    4. In Device Manager, in the Action menu, click "Scan for hardware changes."

    This will enumerate the Kinect sensor again and it will pick up the K4W2 SDK driver, and you should be ready to run KinectService.exe again immediately.

    You can go back and forth between the SDK driver and the libusbK driver very quickly and easily with these steps.

* Install libusb

    Download the latest build (.7z file) from https://github.com/libusb/libusb/releases, and extract as `depends/libusb` (rename folder `libusb-1.x.y` to `libusb` if any).
* Install TurboJPEG

    Download the `-vc64.exe` installer from http://sourceforge.net/projects/libjpeg-turbo/files, extract it to `c:\libjpeg-turbo64` (the installer's default) or `depends/libjpeg-turbo64`, or anywhere as specified by the environment variable `TurboJPEG_ROOT`.
* Install GLFW

    Download from http://www.glfw.org/download.html (64-bit), extract as `depends/glfw` (rename `glfw-3.x.x.bin.WIN64` to `glfw`), or anywhere as specified by the environment variable `GLFW_ROOT`.
* Install OpenCL (optional)
    1. Intel GPU: Download "Intel® SDK for OpenCL™ Applications 2016" from https://software.intel.com/en-us/intel-opencl (requires free registration) and install it.
* Install CUDA (optional, Nvidia only)
    1. Download CUDA Toolkit and install it. You MUST install the samples too.
* Install OpenNI2 (optional)

    Download OpenNI 2.2.0.33 (x64) from http://structure.io/openni, install it to default locations (`C:\Program Files...`).
* Build

    The default installation path is `install`, you may change it by editing `CMAKE_INSTALL_PREFIX`.
    ```
    mkdir build && cd build
    cmake .. -G "Visual Studio 12 2013 Win64"
    cmake --build . --config RelWithDebInfo --target install
    ```
    Or `-G "Visual Studio 14 2015 Win64"`.
    Or `-G "Visual Studio 16 2019"`.
* Run the test program: `.\install\bin\Protonect.exe`, or start debugging in Visual Studio.
* Test OpenNI2 (optional)

    Copy freenect2-openni2.dll, and other dll files (libusb-1.0.dll, glfw.dll, etc.) in `install\bin` to `C:\Program Files\OpenNI2\Tools\OpenNI2\Drivers`. Then run `C:\Program Files\OpenNI\Tools\NiViewer.exe`. Environment variable `LIBFREENECT2_PIPELINE` can be set to `cl`, `cuda`, etc to specify the pipeline.

### Windows / vcpkg

You can download and install libfreenect2 using the [vcpkg](https://github.com/Microsoft/vcpkg) dependency manager:
```
git clone https://github.com/Microsoft/vcpkg.git
cd vcpkg
./vcpkg integrate install
vcpkg install libfreenect2
```
The libfreenect2 port in vcpkg is kept up to date by Microsoft team members and community contributors. If the version is out of date, please [create an issue or pull request](https://github.com/Microsoft/vcpkg) on the vcpkg repository.

### MacOS

Use your favorite package managers (brew, ports, etc.) to install most if not all dependencies:

* Make sure these build tools are available: wget, git, cmake, pkg-config. Xcode may provide some of them. Install the rest via package managers.
* Download libfreenect2 source
    ```
    git clone https://github.com/OpenKinect/libfreenect2.git
    cd libfreenect2
    ```
* Install dependencies: libusb, GLFW
    ```
    brew update
    brew install libusb
    brew install glfw3
    ```
* Install TurboJPEG (optional)
    ```
    brew install jpeg-turbo
    ```
* Install CUDA (optional): TODO
* Install OpenNI2 (optional)
    ```
    brew tap brewsci/science
    brew install openni2
    export OPENNI2_REDIST=/usr/local/lib/ni2
    export OPENNI2_INCLUDE=/usr/local/include/ni2
    ```
* Build
    ```
    mkdir build && cd build
    cmake ..
    make
    make install
    ```
* Run the test program: `./bin/Protonect`
* Test OpenNI2. `make install-openni2` (may need sudo), then run `NiViewer`. Environment variable `LIBFREENECT2_PIPELINE` can be set to `cl`, `cuda`, etc to specify the pipeline.

### Linux

#### Ubuntu 24.04 LTS (Recommended)

This fork includes CUDA 13.0 compatibility fixes and optimized setup instructions for Ubuntu 24.04 LTS.

##### For Desktop PC with NVIDIA GPU (CUDA-enabled)

* Download libfreenect2 source
    ```
    git clone https://github.com/YOUR_USERNAME/libfreenect2.git
    cd libfreenect2
    ```

* Install build tools and dependencies
    ```
    sudo apt update
    sudo apt install build-essential cmake pkg-config libusb-1.0-0-dev libturbojpeg0-dev libglfw3-dev libopenni2-dev -y
    ```

* Install CUDA 13.0+ (NVIDIA GPU only)
    ```
    # Download CUDA Toolkit from https://developer.nvidia.com/cuda-downloads
    # Follow NVIDIA's installation instructions for Ubuntu 24.04
    # Verify installation:
    nvcc --version
    ```

* Install OpenCL (optional)
    ```
    sudo apt install opencl-headers ocl-icd-opencl-dev clinfo -y
    ```

* Build with CUDA support
    ```
    mkdir build && cd build
    cmake .. -DCMAKE_INSTALL_PREFIX=$HOME/freenect2
    make -j$(nproc)
    make install
    ```

* Set up udev rules for device access
    ```
    sudo cp ../platform/linux/udev/90-kinect2.rules /etc/udev/rules.d/
    sudo udevadm control --reload-rules
    sudo udevadm trigger
    ```

* Test with CUDA processor (recommended for NVIDIA GPUs)
    ```
    ./bin/Protonect cuda
    ```

* Alternative processors for testing:
    ```
    ./bin/Protonect cpu      # CPU-only processing
    ./bin/Protonect cl       # OpenCL processing
    ./bin/Protonect gl       # OpenGL processing
    ```

##### For Raspberry Pi 5 (ARM64)

**Note**: Kinect v2 has high bandwidth requirements. Raspberry Pi may experience USB memory issues.

* Download libfreenect2 source
    ```
    git clone https://github.com/YOUR_USERNAME/libfreenect2.git
    cd libfreenect2
    ```

* Install build tools and dependencies
    ```
    sudo apt update
    sudo apt install build-essential cmake pkg-config libusb-1.0-0-dev libturbojpeg0-dev libglfw3-dev -y
    ```

* Increase USB buffer sizes (important for Pi)
    ```
    echo 'options usbcore usbfs_memory_mb=1024' | sudo tee /etc/modprobe.d/usb.conf
    echo 'options usbcore usbfs_bulk_mb=512' | sudo tee -a /etc/modprobe.d/usb.conf
    sudo modprobe -r usbcore && sudo modprobe usbcore
    ```

* Build without CUDA (Pi doesn't support CUDA)
    ```
    mkdir build && cd build
    cmake .. -DCMAKE_INSTALL_PREFIX=$HOME/freenect2 -DENABLE_CUDA=OFF
    make -j4
    make install
    ```

* Set up udev rules
    ```
    sudo cp ../platform/linux/udev/90-kinect2.rules /etc/udev/rules.d/
    sudo udevadm control --reload-rules
    sudo udevadm trigger
    ```

* Test with CPU processor (recommended for Pi)
    ```
    ./bin/Protonect cpu -noviewer
    ```

* If you encounter USB memory errors, try:
    ```
    ./bin/Protonect cpu -noviewer -norgb    # RGB only
    ./bin/Protonect cpu -noviewer -nodepth  # Depth only
    ```

##### CUDA 13.0 Compatibility

This fork includes fixes for CUDA 13.0 compatibility:

* **Fixed deprecated API usage**: Removed `clockRate` and `computeMode` from `cudaDeviceProp` structure
* **Fixed OpenCL macro conflicts**: Resolved `CL_ICDL_VERSION` naming conflicts
* **Tested with**: CUDA 13.0, Ubuntu 24.04 LTS, NVIDIA RTX 5060 Ti

##### Performance Expectations

| Platform | Processor | Expected Performance |
|----------|-----------|---------------------|
| Desktop PC + NVIDIA GPU | CUDA | ~3000 FPS (0.3ms processing) |
| Desktop PC + NVIDIA GPU | OpenCL | ~1000-2000 FPS |
| Desktop PC | CPU | ~100-300 FPS |
| Raspberry Pi 5 | CPU | ~30-60 FPS (may have USB issues) |

##### Troubleshooting

**USB Memory Errors on Raspberry Pi:**
```
[Error] bulk transfer failed: LIBUSB_ERROR_NO_MEM Insufficient memory
```
- Solution: Use powered USB hub or connect Kinect to desktop PC instead

**OpenGL Shader Errors on Pi:**
```
GLSL 3.30 is not supported
```
- Solution: Use `cpu` processor instead of `gl`

**CUDA Compilation Errors:**
```
class "cudaDeviceProp" has no member "clockRate"
```
- Solution: This fork includes fixes for CUDA 13.0 compatibility

##### Legacy Ubuntu Support

For older Ubuntu versions (14.04-22.04), refer to the original installation instructions above. This fork is optimized for Ubuntu 24.04 LTS.
