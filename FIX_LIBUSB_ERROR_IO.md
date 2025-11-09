# Fix: LIBUSB_ERROR_IO on Raspberry Pi

## Problem
When running `Protonect cpu -noviewer` on Raspberry Pi, you see many `LIBUSB_ERROR_IO` errors:
```
[Error] [usb::TransferPool] failed to submit transfer: LIBUSB_ERROR_IO Input/Output Error
```

This happens because the USB transfer pool size (120 transfers) was too high for Raspberry Pi's USB controller.

## Solution Applied

### 1. Reduced Default Transfer Count
- **Changed**: ARM default from 120 → 70 transfers
- **Reason**: 120 transfers overwhelms Pi's USB controller, causing IO errors
- **Result**: More reliable transfer submission with fewer errors

### 2. Improved Error Handling
- **Limited error spam**: Only logs first 5 errors (instead of all)
- **Better diagnostics**: Shows failure percentage and actionable warnings
- **Raspberry Pi-specific guidance**: Suggests `-norgb` flag and transfer count reduction

### 3. Enhanced Startup Messages
- Clear error messages if transfers fail to submit
- Actionable suggestions for Raspberry Pi users

## Testing

**Rebuild and test**:
```bash
cd build
make -j$(nproc)
./bin/Protonect cpu -noviewer -norgb
```

**Expected results**:
- ✅ No or very few LIBUSB_ERROR_IO errors
- ✅ Successful transfer submission
- ✅ Stable depth streaming (may still have some packet loss, but transfers work)

## If You Still See Errors

### Option 1: Further Reduce Transfer Count
```bash
export LIBFREENECT2_IR_TRANSFERS=60
./build/bin/Protonect cpu -noviewer -norgb
```

### Option 2: Verify USB 3.0 Connection
```bash
lsusb -t
```
Look for `5000M` (USB 3.0), not `480M` (USB 2.0)

### Option 3: Use Powered USB 3.0 Hub
Power delivery issues can cause USB IO errors. Use a powered hub.

### Option 4: Increase USB Kernel Buffers
```bash
sudo sh -c 'echo 1000 > /sys/module/usbcore/parameters/usbfs_memory_mb'
```

## Key Takeaway

**ALWAYS use `-norgb` flag on Raspberry Pi** to:
1. Reduce USB bandwidth by ~50%
2. Reduce CPU load (no JPEG decoding)
3. Improve depth stream stability
4. Reduce USB controller stress

## Changes Made

1. **`src/libfreenect2.cpp`**: Reduced ARM default from 120 to 70 transfers
2. **`src/transfer_pool.cpp`**: 
   - Improved error handling (limit spam, show percentages)
   - Added Raspberry Pi-specific warnings
3. **`src/libfreenect2.cpp`**: Enhanced startup error messages

## Commit

These changes address the LIBUSB_ERROR_IO issue by using a more conservative transfer count that works reliably on Raspberry Pi's USB controller.

