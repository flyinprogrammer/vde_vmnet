# vde_vmnet: vmnet.framework support for rootless QEMU

`vde_vmnet` provides [vmnet.framework](https://developer.apple.com/documentation/vmnet) support for VDE applications such as QEMU.

`vde_vmnet` does not require QEMU to run as the root user.

(But `vde_vmnet` itself has to run as the root, in most cases.)

## Install

```console
brew install vde

make

make install
```

## Usage

```console
vde_switch --unix /tmp/vde
```

```console
sudo vde_vmnet /tmp/vde
```

Note: make sure to run `vde_vmnet` with root (`sudo`). See [FAQs](#FAQs) for the reason.

```console
qemu-system-x86_64 \
  -device virtio-net-pci,netdev=net0 -netdev vde,id=net0,sock=/tmp/vde \
  -m 4096 -accel hvf -cdrom ubuntu-21.04-desktop-amd64.iso
```

The guest IP is assigned by the DHCP server provided by macOS.

The guest is accessible to the internet, and the guest IP is accessible from the host.

To confirm, run `sudo apt-get update && sudo apt-get install -y apache2` in the guest, and access the guest IP via Safari on the host.

### Multi VM
Multiple VMs can be connected to a single `vde_vmnet` instance.

Make sure to specify unique MAC addresses to VMs: `-device virtio-net-pci,netdev=net0,mac=de:ad:be:ef:00:01` .

NOTE: don't confuse MAC addresses of VMs with the MAC address of `vde_vmnet` itself that is printed as `vmnet_mac_address` in the debug log.
You do not need to configure (and you can't, currently) the MAC address of `vde_vmnet` itself.

### PTP mode (Switchless mode)

- Pros: doesn't require the `vde_switch` process to be running
- Cons: no support for multi-VM

To enable PTP mode, append `[]` to the socket path argument of `vde_vmnet`.

```console
sudo vde_vmnet /tmp/vde[]
```

QEMU has to be launched with `port=65535` without `[]`.

```console
qemu-system-x86_64 -netdev vde,id=net0,sock=/tmp/vde,port=65535 ...
```

The "port" number here has nothing to do with TCP/UDP ports.

## FAQs
### Why does `vde_vmnet` require root?

Running `vde_vmnet` without root could be possible by signing the `vde_vmnet` binary with `com.apple.vm.networking` entitlement.

However, signing a binary with `com.apple.vm.networking` entitlement seems to require some contract with Apple.
:disappointed:

> This entitlement is restricted to developers of virtualization software. To request this entitlement, contact your Apple representative.

https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_vm_networking

### How is vde_vmnet related to QEMU-builtin vmnet support?
There are proposal to add builtin vmnet support for QEMU:
- https://lore.kernel.org/qemu-devel/20210617143246.55336-1-yaroshchuk2000@gmail.com/
- https://lore.kernel.org/qemu-devel/20210708054451.9374-1-akihiko.odaki@gmail.com/

However, QEMU-builtin vmnet is highly likely to require running the entire QEMU process as root.

On ther otherhand, `vde_vmnet` does not require the entire QEMU process to run as root, though `vde_vmnet` has to run as root.

### How to use static IP addresses?

- Decide a unique MAC address for the VM, e.g. `de:ad:be:ef:00:01`.

- Decide a static IP address, e.g., "192.168.60.100"

- Create `/etc/bootptab` like this. Make sure not to drop the "%%" header.
```
# bootptab
%%
# hostname      hwtype  hwaddr              ipaddr          bootfile
tmp-vm01        1       de:ad:be:ef:00:01   192.168.60.100
```

- Reload the DHCP daemon.
```
sudo /bin/launchctl unload -w /System/Library/LaunchDaemons/bootps.plist
sudo /bin/launchctl load -w /System/Library/LaunchDaemons/bootps.plist
```

- Run QEMU with the MAC address: `-device virtio-net-pci,netdev=net0,mac=de:ad:be:ef:00:01` .

NOTE: don't confuse MAC addresses of VMs with the MAC address of `vde_vmnet` itself that is printed as `vmnet_mac_address` in the debug log.
You do not need to configure (and you can't, currently) the MAC address of `vde_vmnet` itself.

## Links
- https://developer.apple.com/documentation/vmnet
- https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_vm_networking
- [`file:///Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks/vmnet.framework/Versions/Current/Headers/vmnet.h`](file:///Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/System/Library/Frameworks/vmnet.framework/Versions/Current/Headers/vmnet.h)

## Troubleshooting
- Set environment variable `DEBUG=1`