-- Hide all raw Intel IPU6 ISYS capture devices from PipeWire.
-- These are internal Bayer sensor nodes (/dev/video1-48) that are not
-- usable by applications directly. Only the v4l2loopback device
-- "Intel MIPI Camera" (/dev/video0) should be visible.

v4l2_monitor.rules = v4l2_monitor.rules or {}

-- Disable all raw IPU6 V4L2 devices (the PCI ones)
table.insert(v4l2_monitor.rules, {
  matches = {
    {
      { "device.name", "matches", "v4l2_device.pci-0000_00_05.0*" },
    },
  },
  apply_properties = {
    ["device.disabled"] = true,
  },
})

-- Disable all raw IPU6 V4L2 source nodes (in case any slip through)
table.insert(v4l2_monitor.rules, {
  matches = {
    {
      { "node.name", "matches", "v4l2_input.pci-0000_00_05.0*" },
    },
  },
  apply_properties = {
    ["node.disabled"] = true,
  },
})
