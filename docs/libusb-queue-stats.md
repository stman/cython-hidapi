# Native libusb receive queue diagnostics

This contribution exposes the experimentally validated `hid.device.get_queue_stats()` API in Cython. The method returns a dictionary containing `received`, `dropped`, and `max_queue` counters for the existing open device.

```python
device = hid.device()
device.open(0x1A61, 0x3950)
try:
    result = device.get_queue_stats()
    if result["dropped"]:
        raise IOError("HID input reports were dropped")
finally:
    device.close()
```

## Requirements and implementation

This method requires an underlying HIDAPI C library that exports `hid_get_queue_stats()`, as implemented in the related C patch at [stman/hidapi](https://github.com/stman/hidapi/tree/fix/libusb-input-report-overflow).

The tested proof-of-concept uses Python 3.14 and `hidapi` 0.15.0 with the vendored **libusb** backend, and validates the native counters under the existing receive-thread mutex. It does not open a second HID device.

The native queue code was modified from `num_queued > 30` to `num_queued > 1022`, with received/dropped/max_queue accounting. The successful end-to-end run reported `received=22658`, `dropped=0`, `max_queue=84`, matching between Python and the native C close log.

## Integration note

The Python binding and C implementation are separate contributions. A Python build based on an unmodified HIDAPI C library **will not link** with this new method; the C patch must be included in the embedded HIDAPI sources or the build must link against the modified libusb backend. Cross-platform builds require an agreed approach for unsupported HIDAPI backends.

This branch contains the tested Python API adaptation. Further changes to the vendored/native build dependency may be needed to integrate it in the upstream package.
