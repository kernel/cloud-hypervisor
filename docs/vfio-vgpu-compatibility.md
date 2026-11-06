# Testing vendor VFIO sparse BARs

These changes target vendor VFIO devices with sparse, mmap-only regions in a BAR that also contains MSI-X state. They preserve guest interception of the MSI-X table and PBA without changing the mmap requests advertised by the kernel driver.

## Companion dependency fix

`vfio-ioctls` 0.9.1 retains region metadata from the initial sizing ioctl, even after issuing the full-size capability query. Some drivers add `VFIO_REGION_INFO_FLAG_MMAP` only to the full-size response. With an NVIDIA L40S vGPU, the first BAR0 response has `argsz=96, flags=0xb`; the full response has `flags=0xf` and three sparse mmap areas. Keeping the initial flags prevents the VMM from mapping those regions.

The companion fix and its regression assertions are in `patches/vfio-ioctls-0.9.1-region-info.patch`. This patch is not automatically applied by the normal build. Apply it to a local checkout when validating this branch:

```sh
repo="$PWD"
git clone --depth=1 --branch vfio-ioctls-v0.9.1 https://github.com/rust-vmm/vfio.git /tmp/vfio-vgpu
git -C /tmp/vfio-vgpu apply "$repo/patches/vfio-ioctls-0.9.1-region-info.patch"
cargo build --release -p cloud-hypervisor --bin cloud-hypervisor \
  --config 'patch.crates-io.vfio-ioctls.path="/tmp/vfio-vgpu/vfio-ioctls"'
```

The local path override changes Cargo's lock resolution; do not publish that local path or its lockfile changes. A normal release build still needs a published dependency containing this fix.

## VMM changes

- Restore BAR values after size probes, initialize the VFIO driver's BAR configuration with the allocated guest addresses, and forward guest BAR writes. Physical VFIO drivers filter unsupported writes; mediated devices may need these updates for their MMIO translation.
- Prefer the kernel's explicit sparse mmap ranges over whole-BAR capabilities.
- Map each advertised host range in full. Some vendor drivers reject mmap requests for smaller portions of an advertised range.
- Register only the guest-visible portions with KVM, leaving pages containing MSI-X tables and PBAs intercepted. The guest mappings share ownership of the original full host mmap.

## Validation and limits

A v54.0 build with these VMM changes and the companion dependency patch was tested with Linux 6.8, an NVIDIA L40S-3Q vGPU, host driver 580.178.05, and guest driver 580.105.08. The guest initialized the NVIDIA driver, Chromium rendered a WebGL pixel using the NVIDIA renderer, and a new `ffmpeg` process encoded frames with `h264_nvenc`.

Use `iommufd=on,vfio_p2p_dma=off` for that configuration. The latter avoids unsupported BAR peer-to-peer mappings on the older host kernel. Host IOMMU isolation remained enabled.

This validates cold boot only. GPU checkpoint/restore did not pass hardware validation: the driver reported a guest-to-host PTE translation failure during state capture. An HTTP endpoint responding after restore is not sufficient evidence that rendering or encoding survived. Do not remove GPU snapshot restrictions based on the cold-boot result.

The full VFIO hardware integration matrix, other GPU profiles, other drivers, AArch64 and MSHV were not exercised. Keep those as separate review and rollout gates.
