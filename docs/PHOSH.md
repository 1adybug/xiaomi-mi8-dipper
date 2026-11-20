# Phosh, USB networking and recovery on dipper

A Phosh session was tested on an existing Nura edge (formerly postmarketOS)
installation on a Mi 8 with the EA8074 panel patch and the
[FTS5 FIFO fix](https://github.com/Mechres/xiaomi-mi8-dipper/pull/1).
The phone booted to the Phosh greeter and opened a desktop session.
[Nura's supported interfaces](https://nura.eco/)

## Keep Android recovery ready

Before replacing `boot` and `userdata`, back up personal data and save the
appropriate Android ROM and recovery for `dipper` on the host. For a
LineageOS install, keep the signed ZIP and the corresponding recovery for
the installed release. Record checksums of your local backups and any
boot, recovery, DTBO, or vbmeta images saved from the phone. Use the
matching device guide for a return to Android.
[LineageOS Mi 8 device guide](https://wiki.lineageos.org/devices/dipper/)

For Linux kernel experiments, keep a known-good boot image and its
matching signed module tree. A distinct `CONFIG_LOCALVERSION` for the
test kernel lets both module trees coexist. Check `fastboot getvar product`
for `dipper`, then use `fastboot boot` to trial the new image. Confirm USB
SSH and check that three separate taps each produce a press and a release
at the greeter before flashing the boot partition. The trial and final
boot of the FTS5 FIFO fix were both exercised on hardware.

## Build an optional Phosh image

Prepare the device tree, local device package, and kernel using the
[port build instructions](../README.md#installing). Set the local build to
Phosh with OpenRC before generating the root filesystem.

```sh
pmbootstrap config device xiaomi-dipper
pmbootstrap config ui phosh
pmbootstrap config service_manager openrc
pmbootstrap install --add mesa-dri-gallium
```

The interactive UI menu uses `deviceinfo_drm` to list Phosh. The explicit
`ui` setting selects it for the local build. With the default recommendations
enabled, the installer adds the language support, fonts, and recommended apps.
The Phosh package enables greetd through its OpenRC install script.
[Nura UI package metadata](https://docs.nura.eco/pmaports/main/packaging/ui-packages/apkbuild-metadata.html)

Use the `flash_rootfs` and `flash_kernel` commands in the
[port installation instructions](../README.md#installing) for the generated
images. See the [GPU firmware notes](../README.md#after-installing) for hardware
acceleration.

## Check USB networking and the desktop

If NetworkManager takes over `usb0`, place the following in
`/etc/NetworkManager/conf.d/90-dipper-usb.conf` in the root filesystem. This
lets the USB gadget keep managing the interface.

```ini
[keyfile]
unmanaged-devices=interface-name:usb0
```

After boot, check `loginctl seat-status seat0` over SSH. The greeter needs
both DRM and input devices on the seat. When Phosh was brought up on the
existing installation, refreshing the device rules restored the seat on the
tested phone.

```sh
udevadm control --reload
for subsystem in drm input graphics; do
    udevadm trigger --action=add --subsystem-match="$subsystem"
done
udevadm settle
rc-service greetd restart
```

## Rear camera

The device tree describes the Semco IMX363 at CCI address `0x1a` on
the Dipper new MP v2.1 board with the P0 camera pinout. The sensor connects
to CAMSS through CSIPHY0. Other module variants may use a different address
or wiring. This port covers basic rear camera capture. Autofocus, optical
stabilization, and the other camera modules are unavailable.
[Dipper camera bring-up on Mobian](https://github.com/taygoth/dipper-mainline-port/tree/e6d1be1464e25a2bad68aa81641c0b7438edea78)

Build both the device package and the patched 7.1 kernel from the current
source. For an existing installation, use a higher local kernel `pkgrel`,
update any exact APK version pins, install both rebuilt packages, and boot
the regenerated boot image with its matching signed modules.

Install GNOME Snapshot on an existing Phosh system with `apk add snapshot`.
When building a Phosh image, add `snapshot` to the package list passed to
`pmbootstrap install --add`. Open **Camera** in the app grid.

To check detection and streaming, install `libcamera-tools` and run these
commands as the Phosh session user. Select the IMX363 index from `cam --list`
if it differs from `1`.

```sh
cam --list
cam -c 1 --capture=30
```

## Diagnose missing USB Ethernet on Windows

On one Windows host the `18D1:D001` USB device was assigned a legacy
Android ADB Recovery driver, so the CDC NCM network adapter never
appeared. In Device Manager, inspect the hardware ID and the bound
driver. The USB gadget must expose the `ncm.usb0` function for the
network setup tested here.

For a private test, set an authorized alternative in
`deviceinfo_usb_idVendor` and `deviceinfo_usb_idProduct` in
`/etc/deviceinfo` or the equivalent file in the build root, then run
`mkinitfs`. Boot the regenerated image and check that the host gets
`172.16.42.2` with the default USB subnet. The pid.codes shared test pair
`1209:0001` enabled the network adapter and SSH on the affected host. It
is reserved for private testing and can collide with other test devices.
Use an assigned identifier for redistributed devices.
[pid.codes shared test identifier](https://pid.codes/1209/0001/)
